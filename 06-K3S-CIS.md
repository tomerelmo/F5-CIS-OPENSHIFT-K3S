# F5 Container Ingress Services (CIS) — K3S Single-Server Deployment Guide

> **CIS purpose:** Automate BIG-IP Virtual Server configuration from Kubernetes CRD objects
> **CIS version:** `2.20.3`
> **BIG-IP pair:** BIGIP-K3S-01 / BIGIP-K3S-02
> **Data subnet:** `10.1.20.0/24` (Float: `10.1.20.30`, K3S-01: `10.1.20.31`, K3S-02: `10.1.20.32`)
---

## Step 0 — Set the K3S Node IP to the Data Interface

By default, K3S advertises the node IP from the interface with the default route — which in the UDF lab is the management interface, not the data network. CIS in NodePort mode uses the node's `InternalIP` as the BIG-IP pool member address, so if it picks the wrong IP, traffic from the BIG-IP will never reach the K3S node.

We fix this by telling K3S to advertise `10.1.20.95` (the data subnet IP) as its node IP:
```bash
sudo tee -a /etc/rancher/k3s/config.yaml <<EOF
node-ip: "10.1.20.95"
EOF

sudo systemctl restart k3s
```

Verify:
```bash
kubectl get nodes -o wide
```

The `INTERNAL-IP` column should show `10.1.20.95`. If it does, CIS will automatically use this IP for pool members.

---

## Step 1 — Download the CIS Helm Chart

From the K3S server web shell:

```bash
cd ~
git clone https://github.com/F5Networks/k8s-bigip-ctlr.git
cd k8s-bigip-ctlr/helm-charts/f5-bigip-ctlr/
```

---

## Step 2 — Install CIS with Helm

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

helm install cis ./ \
  --set bigip_secret.create="true" \
  --set bigip_secret.username="admin" \
  --set bigip_secret.password="F5train1!" \
  --set args.bigip_url="https://10.1.30.200" \
  --set args.bigip_partition="k8s" \
  --set args.custom-resource-mode="true" \
  --set args.insecure="true" \
  --set args.gtm-bigip-password="F5train1!" \
  --set args.gtm-bigip-username="admin" \
  --set args.gtm-bigip-url="https://10.1.30.200" \
  --set args.pool_member_type="nodeport" \
  --set args.ipam="true" \
  --set version="2.20.3"
```

Expected output:

```
NAME: cis
LAST DEPLOYED: ...
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

---

## Step 3 — Check CIS Pod Status

```bash
kubectl get pods -A | grep cis
```

You will likely see the pod in a crash loop:

```
kube-system   cis-f5-bigip-ctlr-xxxxxxxxxx-xxxxx   0/1   CrashLoopBackOff   ...
```

Check the logs:

```bash
CIS_POD=$(kubectl get pods -n kube-system -l app=f5-bigip-ctlr -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n kube-system $CIS_POD
```

You will see:

```
[ERROR] [AS3][BigIP] RPM is not installed on BIGIP, Error response from BIGIP with status code 404
[ERROR] AS3 RPM is not installed on BIGIP
```

This is expected — we need to install the AS3 RPM on the BIG-IP devices first.

---

## Step 4 — Install AS3 on the BIG-IP Devices

Download the AS3 RPM from:

```
https://github.com/F5Networks/f5-appsvcs-extension/releases/download/v3.56.0/f5-appsvcs-3.56.0-10.noarch.rpm
```

#### Upload the RPM to both BIG-IP devices:

If not done with openshift section Log in to the TMUI of **BIGIP-K3S-01 or 02 (the active one)**:

```
user: admin
password: F5train1!
```

![bigip login page](/img/03-bigip-login.png)

Navigate to **iApps → Package Management LX**:

![iapp mgmt package lx](/img/03-package-management-lx.png)

Press **Import** and upload the RPM:

![import AS3](/img/03-import-as3.png)


---

## Step 5 — Wait for CIS to Become Ready

After installing AS3, the CIS pod will recover on its own. Wait a few minutes and check:

```bash
kubectl get pods -A | grep cis
```

Expected (after several minutes):

```
kube-system   cis-f5-bigip-ctlr-xxxxxxxxxx-xxxxx   1/1   Running   ...
```

> **Note:** It may take several minutes and multiple restarts before the pod reaches `1/1 Running`. This is normal.

---

## Step 6 — Deploy a Test Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: kube-system
spec:
  type: NodePort
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

> **Note:** Service type is `NodePort` (not ClusterIP) — this matches the `pool_member_type: nodeport` we configured in CIS.

Verify:

```bash
kubectl get pods | grep nginx
kubectl get svc nginx-demo
```

---

## Step 7 — Create a VirtualServer CRD

> **Reminder:** The F5 CRDs should already be installed from the IPAM guide (Step 7). If not, run:
> ```bash
> kubectl apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml
> ```

Now create the VirtualServer:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: kube-system
  labels:
    f5cr: "true"
spec:
  host: nginx-demo.cis.lab
  ipamLabel: test
  snat: auto
  pools:
    - path: /
      service: nginx-demo
      servicePort: 80
      monitors:
        - type: http
          interval: 10
          timeout: 10
          send: "GET / HTTP/1.1\r\nConnection: close\r\n\r\n"
          recv: ""
EOF
```

Verify the VS was created and got an IP from IPAM:

```bash
kubectl get vs
```

Expected:

```
NAME            HOST                 TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS   IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
nginx-demo-vs   nginx-demo.cis.lab                                              test        10.1.20.100     OK       44s
```

---

## Step 8 — Verify on the BIG-IP

Navigate to **Local Traffic → Virtual Servers** on the BIG-IP TMUI.

![virtual server under local traffic](/img/03-ltm-vs.png)

#### Where is the Virtual Server?

CIS is configured to use a specific partition. Switch the partition in the top-right corner to:

```
k8s
```

![partition](/img/03-partition.png)

You should now see the `nginx-demo-vs` Virtual Server configured.

---

## Step 9 — Configure BIG-IP Pod Routing

Since we are using `nodeport` mode, the BIG-IP needs to reach the K3S node on the data network. On K3S single-server, the node IP on the data subnet is the K3S server itself.

On **one of the BIGIP-K3S** devices, verify you can reach the K3S node:

```text
ping 10.1.20.31
```

> With `nodeport` mode, BIG-IP sends traffic to the **node IP + NodePort**, so you don't need pod-level routes like in cluster mode. As long as BIG-IP can reach `10.1.20.31` (the K3S server) on the data VLAN, traffic will flow.

If the ping fails, verify the VLAN configuration from the IPAM guide Step 1 was applied correctly.

---

## Step 10 — Enable Auto-Sync on the BIG-IP HA Pair

On the **active** BIGIP-K3S device, navigate to:

**Device Management → Device Groups → K3S_Cluster** (or your device group name)

![device-mgmt](/img/03-device-mgmt.png)

Switch from **Manual Sync** to **Automatic Sync**:

![auto sync](/img/03-autosync.png)

Now any VS change made by CIS will automatically replicate to the standby device.

---

## Step 11 — Test Traffic

From the K3S server web shell:

```bash
curl http://10.1.20.100 -H "Host: nginx-demo.cis.lab"
```

Expected:

```html
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
```

---

## Step 12 — Test HA Failover

#### Find the active BIG-IP

From the web shell of both BIGIP-K3S devices:

```text
tmsh show sys failover
```

One will show `Failover active`.

#### Reboot the active device and immediately test

```text
# On the active BIGIP-K3S:
reboot
```

Immediately from the K3S server:

```bash
curl http://10.1.20.100 -H "Host: nginx-demo.cis.lab"
```

You should still get the nginx response — the standby BIG-IP has taken over.

Verify the failover on the other device:

```text
tmsh show sys failover
# Should show: Failover active for 0d 00:0X:XX
```

> The BIG-IP HA pair provides fault tolerance — both devices share the same VS configuration, so if one fails, the other continues serving traffic seamlessly.

---

## Troubleshooting Checklist

| Symptom | Fix |
|---------|-----|
| CIS pod in `CrashLoopBackOff` with AS3 error | Install the AS3 RPM on both BIG-IP devices (Step 4). |
| `no matches for kind "VirtualServer"` | CRDs not installed. Run: `kubectl apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml` |
| VS created but no IP assigned | IPAM controller not running. Check: `kubectl get pods -n kube-system \| grep ipam` |
| VS shows `OK` but BIG-IP has no VS | Check you're looking at the correct partition (`k8s`) on the BIG-IP TMUI. |
| Traffic not reaching nginx | Verify BIG-IP can ping the K3S node (`10.1.20.31`). Check VLAN interfaces from IPAM guide Step 1. |
| Config not syncing to standby BIG-IP | Enable auto-sync on the device group (Step 10). |
| `Kubernetes cluster unreachable` | Run: `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` |

---

## CRD Examples Reference

For more VirtualServer CRD examples (TLS profiles, TransportServer for non-HTTP, ExternalDNS for GSLB), see:

```
https://github.com/F5Networks/k8s-bigip-ctlr/tree/master/docs/config_examples/customResource
```