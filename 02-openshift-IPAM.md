# F5 IPAM Controller — OpenShift Deployment Guide

> **IPAM purpose:** Automatic IP allocation for F5 BIG-IP VirtualServer CRDs via labels (`test` / `prod`)
> **Image version:** `f5networks/f5-ipam-controller:0.1.12`
> **Cluster:** OpenShift with 3 masters and 2 workers

---

## Overview

The IPAM controller is a container responsible for **IP allocation** across the Virtual Services we deploy. We configure it with predefined IP ranges and later reference those ranges as labels on the VirtualServer CRDs we create.

---

## Step 1 — Connect to the OCP Provisioner Web Shell

Open the web shell on the **ocp-provisioner** server:

![picture of the ocp-provisioner](/img/02-1.png)

Log in as `cloud-user` and create a working directory:

```bash
su cloud-user
mkdir -p /home/cloud-user/ipam
cd /home/cloud-user/ipam
```

---

## Step 2 — Add the F5 IPAM Helm Repository

```bash
helm repo add f5-ipam-stable https://f5networks.github.io/f5-ipam-controller/helm-charts/stable
```

Verify it was added:

```
"f5-ipam-stable" has been added to your repositories
```

---

## Step 3 — Download and Customise values.yaml

Download the default values file — this will create the necessary components for bringing up the IPAM (RBAC, ServiceAccount, PersistentVolume, and image configuration):

```bash
wget -O value.yaml https://raw.githubusercontent.com/F5Networks/f5-ipam-controller/main/helm-charts/f5-ipam-controller/values.yaml
```

Edit the following sections to prepare for installation.

### 3a — Set the IP Ranges

Find the `ip_range` line and replace it with your environment's subnets:

```yaml
# Find this line:
  ip_range: '{"test":"172.16.1.1-172.16.1.5", "prod":"172.16.1.50-172.16.1.55"}'

# Replace with:
  ip_range: '{"test":"10.1.10.100-10.1.10.120", "prod":"10.1.10.130-10.1.10.150"}'
```

### 3b — Configure the PersistentVolumeClaim

Set PVC creation to `true` and configure the storage class:

```yaml
pvc:
  create: true
  name:
  storageClassName: ipam
  accessMode: ReadWriteOnce
  storage: 1Gi
```

### 3c — Set the Image Configuration

Update the image section to pull from Docker Hub with the correct version:

```yaml
image:
  user: docker.io/f5networks
  repo: f5-ipam-controller
  pullPolicy: IfNotPresent
  version: 0.1.12
```

---

## Step 4 — Pull the IPAM Image to the Worker Nodes

Log in to each worker node, pull the image into podman, and create the PVC directory.

#### Worker 1

```bash
oc debug node/worker-1.ocp.f5-udf.com
# Wait a few seconds until you see: sh-5.1#
chroot /host
podman pull docker.io/f5networks/f5-ipam-controller:0.1.12
exit
exit
```

#### Worker 2

Repeat the same steps for Worker 2:

```bash
oc debug node/worker-2.ocp.f5-udf.com
chroot /host
podman pull docker.io/f5networks/f5-ipam-controller:0.1.12
exit
exit
```

---

## Step 5 — Create a Local PersistentVolume on Worker 1

First, create the directory on the worker node:

```bash
oc debug node/worker-1.ocp.f5-udf.com
chroot /host
mkdir -p /var/f5-ipam
chmod 777 /var/f5-ipam
exit
exit
```

Now create and apply the PV manifest:

```bash
cat <<'EOF' > pv-ipam.yaml
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
            - worker-1.ocp.f5-udf.com
EOF

oc apply -f pv-ipam.yaml
```

Expected output:

```
persistentvolume/ipam-local-pv created
```

> **Production note:** In production, the F5 IPAM Controller should use a dynamically provisioned PersistentVolume backed by a resilient CSI storage class (e.g., Ceph RBD, cloud block storage, or enterprise SAN) with `volumeBindingMode: WaitForFirstConsumer`, ensuring the IPAM database is durable, node-independent, and survives pod restarts or node failures.

---

## Step 6 — Install IPAM with Helm

```bash
helm install ipam f5-ipam-stable/f5-ipam-controller \
  -f value.yaml \
  -n kube-system
```

---

## Step 7 — Verify the Deployment

Check that the IPAM pod is running and ready:

```bash
oc get pods -A | grep ipam
```

Expected output:

```
kube-system   ipam-f5-ipam-controller-5f766c75cf-ncgpc   1/1   Running   0   5m30s
```

---

## How IPAM Labels Work

The IPAM controller is configured with two IP ranges, each identified by a label:

```json
{"test":"10.1.10.100-10.1.10.120", "prod":"10.1.10.130-10.1.10.150"}
```

When deploying a new VirtualServer CRD object, you specify the label (`test` or `prod`) using the `ipamLabel` field. The IPAM controller then allocates an IP address from the corresponding pool and assigns it to the VirtualServer on the BIG-IP.