# F5 IPAM Controller — K3S Single-Server Deployment Guide

> **IPAM purpose:** Automatic IP allocation for F5 BIG-IP VirtualServer CRDs via labels (`test` / `prod`)
> **Image version:** `f5networks/f5-ipam-controller:0.1.12`

---

## Step 1 — Switch VLANs on the BIG-IP Pair (BIGIP-K3S)

> **⚠️ Do this BEFORE anything else.**

Connect to the web shell of **both** BIG-IP devices in the BIGIP-K3S pair from the UDF environment.

![BIG-IP Web Shell](/img/bigip-webshell.png)

On **each** BIG-IP, run the following commands in TMSH to reassign the VLAN interfaces:

```text
modify net vlan cis_mgmt interfaces delete { all }
modify net vlan k3s_data  interfaces delete { all }

modify net vlan cis_mgmt interfaces add {1.1}
modify net vlan k3s_data  interfaces add {1.2}

create auth partition k8s
```

Verify on each device:

```text
show net vlan cis_mgmt
show net vlan k3s_data
```

Make sure `cis_mgmt` is on interface `1.1` and `k3s_data` is on interface `1.2` before proceeding.

---

## Step 2 — Connect to the K3S Server Web Shell

From the UDF environment, open the **web shell** on the K3S server.

![K3S Web Shell](/img/k3s-webshell.png)

---

## Step 3 — Create a Working Directory

```bash
mkdir -p ~/f5-ipam && cd ~/f5-ipam
```

---

## Step 4 — Install Helm 3 and Add the F5 IPAM Repository

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

```bash
helm repo add f5-ipam-stable https://f5networks.github.io/f5-ipam-controller/helm-charts/stable
helm repo update
```

Verify:

```bash
helm search repo f5-ipam-stable
```

You should see `f5-ipam-stable/f5-ipam-controller` in the output.

---

## Step 5 — Prepare the Persistent Volume

FIC uses a SQLite DB that requires specific UID/GID permissions (`1200:1200`). We create a local PV with a pre-created directory on the host.

### 5a — Create the host directory

```bash
sudo mkdir -p /var/f5-ipam
sudo chmod 777 /var/f5-ipam
```

### 5b — Create the PersistentVolume manifest

Get your K3S node name:

```bash
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo $NODE_NAME
```

Create and apply the PV:

```bash
cat <<EOF > pv-ipam.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ipam-local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ipam
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /var/f5-ipam
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - ${NODE_NAME}
EOF

kubectl apply -f pv-ipam.yaml
```

Confirm:

```bash
kubectl get pv ipam-local-pv
# STATUS should be "Available"
```

---

## Step 6 — Download and Customise values.yaml

```bash
wget -O values.yaml \
  https://raw.githubusercontent.com/F5Networks/f5-ipam-controller/main/helm-charts/f5-ipam-controller/values.yaml
```

Apply **all** the edits below. You can use `sed` or edit manually with `vi`/`nano`.

### 6a — Set the IP ranges

Find the `ip_range` line and replace it:

```yaml
args:
  orchestration: "kubernetes"
  provider: "f5-ip-provider"
  ip_range: '{"test":"10.1.20.100-10.1.20.120", "prod":"10.1.20.130-10.1.20.150"}'
```

> The keys `test` and `prod` are the **ipamLabel** values you will reference in VirtualServer CRDs.

### 6b — Configure the PVC to use our local PV

```yaml
pvc:
  create: true
  name: ""
  storageClassName: ipam
  accessMode: ReadWriteOnce
  storage: 1Gi
```

### 6c — Set the image configuration

```yaml
image:
  user: f5networks
  repo: f5-ipam-controller
  pullPolicy: IfNotPresent
  version: 0.1.12
```



### 6d — Set the security context (critical for SQLite DB permissions)

Make sure these lines exist in your `values.yaml`:

```yaml
securityContext:
  runAsUser: 1200
  runAsGroup: 1200
  fsGroup: 1200
```

---

## Step 7 — Install with Helm

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

helm install ipam f5-ipam-stable/f5-ipam-controller \
  -f values.yaml \
  -n kube-system
```

---

## Step 8 — Verify the Deployment

### Check the pod

```bash
kubectl get pods -n kube-system | grep ipam
```

Expected output (wait up to ~60 seconds):

```
ipam-f5-ipam-controller-xxxxxxxxxx-xxxxx   1/1     Running   0   30s
```

### Check the PVC is bound

```bash
kubectl get pvc -n kube-system | grep ipam
```

Should show `Bound` status.

### Check the logs for errors

```bash
kubectl logs -n kube-system -l app=f5-ipam-controller --tail=30
```

Look for:

```
[INFO] [INIT] Starting: F5 IPAM Controller
[DEBUG] [MGR] Creating Manager with Provider: f5-ip-provider
```

If you see `permission denied` on the SQLite DB, re-check directory permissions on `/var/f5-ipam` (Step 5a).

---

## Step 9 — Verify the IPAM CRD Exists

```bash
kubectl get crd | grep ipam
```

You should see `ipams.fic.f5.com`.

---

## Troubleshooting Checklist

| Symptom | Fix |
|---------|-----|
| Pod in `CrashLoopBackOff` | Check logs — usually a permissions issue on `/app/ipamdb`. Re-run Step 5a. |
| PVC stuck in `Pending` | Confirm the PV `storageClassName` matches the PVC (`ipam`). Run `kubectl get pv,pvc -n kube-system`. |
| PVC `Pending` after helm reinstall | The old PV is stuck in `Released` state. Run: `kubectl delete pv ipam-local-pv && sudo rm -rf /var/f5-ipam/* && kubectl apply -f pv-ipam.yaml`, then re-run `helm install`. |
| Image pull error | Verify internet access or import the image manually (see 6c air-gapped note). |
| `ipams.fic.f5.com not found` | The Helm chart installs the CRD automatically. Run: `helm upgrade --install ipam f5-ipam-stable/f5-ipam-controller -f values.yaml -n kube-system` |
| `Kubernetes cluster unreachable` on helm install | K3S kubeconfig not set. Run: `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` before any helm/kubectl command. |

---
