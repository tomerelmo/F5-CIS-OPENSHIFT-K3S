# F5 Container Ingress Services (CIS) — OpenShift Deployment Guide

> **CIS purpose:** Automate BIG-IP Virtual Server configuration from Kubernetes CRD objects
> **CIS version:** `2.20.3`
> **BIG-IP pair:** BIGIP-OC-01 / BIGIP-OC-02
> **Float Self-IP:** `10.1.30.201`

---

## Step 1 — Download the CIS Helm Chart

Now that the IPAM is installed, we can move on to installing CIS.

```bash
git clone https://github.com/F5Networks/k8s-bigip-ctlr.git
cd k8s-bigip-ctlr/helm-charts/f5-bigip-ctlr/
```

---

## Step 2 — Pull the CIS Image to the Worker Nodes

Before installing, pull the CIS image into the podman cache on both workers:

```bash
# Worker 1
oc debug node/worker-1.ocp.f5-udf.com -- chroot /host podman pull docker.io/f5networks/k8s-bigip-ctlr:2.20.3

# Worker 2
oc debug node/worker-2.ocp.f5-udf.com -- chroot /host podman pull docker.io/f5networks/k8s-bigip-ctlr:2.20.3
```

---

## Step 3 — Install CIS with Helm

We use `--set` flags to override the default `values.yaml` parameters. The `bigip_url` and `gtm-bigip-url` are set to the **Float Self-IP** of the BIG-IP cluster, so if a failover occurs the system remains resilient.

```bash
helm install cis ./ \
  --set bigip_secret.create="true" \
  --set bigip_secret.username="admin" \
  --set bigip_secret.password="F5train1!" \
  --set args.bigip_url="https://10.1.30.201" \
  --set args.bigip_partition="k8s" \
  --set args.custom-resource-mode="true" \
  --set args.insecure="true" \
  --set args.gtm-bigip-password="F5train1!" \
  --set args.gtm-bigip-username="admin" \
  --set args.gtm-bigip-url="https://10.1.30.201" \
  --set args.pool_member_type="cluster" \
  --set image.user="docker.io/f5networks" \
  --set image.repo="k8s-bigip-ctlr" \
  --set image.pullPolicy="IfNotPresent" \
  --set version="2.20.3" \
  --set args.ipam="true"
```

Expected output:

```
NAME: cis
LAST DEPLOYED: Tue Mar  3 09:13:36 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Container Ingress Services controller: cis
```

---

## Step 4 — Diagnose the CrashLoop — Install AS3

The CIS pod will initially be in a crash loop. Let's check the logs to find out why:

```bash
oc get pods -A | grep cis

# Take your CIS pod name and check its logs
oc logs -n kube-system <CIS_POD_NAME>
```

You will see:

```
[WARNING] BIG-IP credentials are not set. Checking back to Creds Directory.
[INFO] [INIT] Starting: Container Ingress Services - Version: v2.20.3
[ERROR] [AS3][BigIP] RPM is not installed on BIGIP, Error response from BIGIP with status code 404
[ERROR] AS3 RPM is not installed on BIGIP
```

This error means we need to install the **AS3 automation engine** on the BIG-IP devices.

#### Download the AS3 RPM

```
https://github.com/F5Networks/f5-appsvcs-extension/releases/download/v3.56.0/f5-appsvcs-3.56.0-10.noarch.rpm
```

#### Upload AS3 to the BIG-IP

Log in to **BIGIP-OC-01** TMUI:

```
user: admin
password: F5train1!
```

![bigip login page](/img/03-bigip-login.png)

Navigate to **iApps → Package Management LX**:

![iapp mgmt package lx](/img/03-package-management-lx.png)

Press **Import** and upload the RPM:

![import AS3](/img/03-import-as3.png)

**Repeat the upload on all four BIG-IP devices:**

- BIGIP-OC-01
- BIGIP-OC-02
- BIGIP-K3S-01
- BIGIP-K3S-02

---

## Step 5 — Wait for CIS to Become Ready

After installing AS3, the CIS pod will recover on its own. It may take several minutes and multiple restarts before reaching `1/1 Running`:

```bash
oc get pods -A | grep -i cis
# Initially:
# kube-system   cis-f5-bigip-ctlr-...   0/1   Running   10 (6m9s ago)   27m

# After a few minutes:
# kube-system   cis-f5-bigip-ctlr-...   1/1   Running   10 (7m25s ago)  28m
```

> **Note:** Be patient — it takes some time for the container to become ready.

---

## Step 6 — Deploy a Test Application

Pull the nginx image on both workers, then deploy a test application with a ClusterIP service:

```bash
# Pull the image on Worker 1
oc debug node/worker-1.ocp.f5-udf.com -- chroot /host podman pull docker.io/nginx:1.25

# Pull the image on Worker 2
oc debug node/worker-2.ocp.f5-udf.com -- chroot /host podman pull docker.io/nginx:1.25
```

Deploy the application:

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

Verify:

```bash
# Confirm both resources were created
# deployment.apps/nginx-demo created
# service/nginx-demo created

oc get pods -A | grep nginx
```

> **Reference:** For a full list of CRD examples and possibilities, see:
> `https://github.com/F5Networks/k8s-bigip-ctlr/tree/master/docs/config_examples/customResource`
>
> Pay attention to the different resource types: **VirtualServer** (HTTP), **TransportServer** (non-HTTP), **VirtualServerWithTLSProfile**, and **ExternalDNS** (used later for GSLB).

---

## Step 7 — Install the Custom Resource Definitions

Before creating a VirtualServer CRD, OpenShift needs to know what that resource type is. If you apply the VS now, you will get:

```
error: resource mapping not found for name: "nginx-demo-vs" namespace: "default" from "STDIN":
no matches for kind "VirtualServer" in version "cis.f5.com/v1"
ensure CRDs are installed first
```

Install the CRD definitions:

```bash
cd /home/cloud-user/cis/k8s-bigip-ctlr/docs/config_examples/customResourceDefinitions/
oc apply -f customresourcedefinitions.yml
```

Expected output:

```
customresourcedefinition.apiextensions.k8s.io/virtualservers.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/tlsprofiles.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/transportservers.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/externaldnses.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/ingresslinks.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/policies.cis.f5.com created
```

---

## Step 8 — Create a VirtualServer CRD

Now let's create a VirtualServer object. Notice the `ipamLabel: test` — IPAM will allocate an IP from the test range.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: default
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
          targetPort: 80
EOF
```

Verify the VS was created and received an IP from IPAM:

```bash
oc get vs
```

Expected output:

```
NAME            HOST                 TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS   IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
nginx-demo-vs   nginx-demo.cis.lab                                              test        10.1.10.100     OK       44s
```

---

## Step 9 — Verify the Virtual Server on the BIG-IP

Navigate to **Local Traffic → Virtual Servers** on the BIG-IP TMUI:

![virtual server under local traffic](/img/03-ltm-vs.png)

#### Where is the Virtual Server?

CIS is configured to work with a specific partition. You need to switch the partition in the top-right corner to the one we configured in the Helm values:

```
--set args.bigip_partition="k8s"
```

Switch to the `k8s` partition and you will see the Virtual Server:

![partition](/img/03-partition.png)

We are currently on BIGIP-OC-02 (the active device):

![bigip 02](/img/03-bigip02.png)

Now go to the UDF components section and open the TMUI of **BIGIP-OC-01** (if BIGIP-OC-01 is your active device, then log in to BIGIP-OC-02 instead).

Switch to the correct partition — can you see the Virtual Server?

---

## Step 10 — Enable Auto-Sync on the BIG-IP HA Pair

The configuration is currently synced manually. To enable automatic sync, log in to the **active** BIG-IP TMUI and navigate to:

**Device Management → Device Groups → OC_Cluster**

![device-mgmt](/img/03-device-mgmt.png)

Switch from **Manual Sync** to **Automatic Sync**:

![auto sync](/img/03-autosync.png)

---

## Step 11 — Verify Auto-Sync Works

Delete the VS, confirm it disappears from both BIG-IP devices, then recreate it:

```bash
oc delete vs nginx-demo-vs
```

Verify the VS is gone on **both** BIG-IP TMUI pages. Then recreate it:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: default
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
          targetPort: 80
EOF
```

Now the configuration is synced automatically — every change made by CIS will immediately affect the other BIG-IP.

> **Discussion:** What are the benefits and downsides of automatic sync in this design?

---

## Step 12 — Add Pod Routes on the BIG-IP

Let's test traffic to the application at `10.1.10.100`:

```text
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# list ltm node /k8s/10.244.3.70
ltm node /k8s/10.244.3.70 {
    address 10.244.3.70
    partition k8s
}
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.3.70
PING 10.244.3.70 (10.244.3.70) 56(84) bytes of data.
^C
--- 10.244.3.70 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

The BIG-IP can't reach the pod IPs because it doesn't know the pod subnets (`10.244.3.0/24` for Worker 1 and `10.244.4.0/24` for Worker 2). We need to define routes through the `cis_data` interface.

On **one of the BIG-IP devices**, navigate to **Network → Routes**:

![network routes](/img/03-route-network.png)

Press **Add** in the top-right corner and create the following routes:

- `10.244.3.0/24` via `10.1.10.9` (Worker 1)
- `10.244.4.0/24` via `10.1.10.10` (Worker 2)

![add worker pod gateway](/img/03-add-worker-pod-gw.png)

> **Don't forget** to add routes for both subnets.

Verify connectivity from the BIG-IP web shell:

```text
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.3.70
PING 10.244.3.70 (10.244.3.70) 56(84) bytes of data.
64 bytes from 10.244.3.70: icmp_seq=1 ttl=62 time=2.11 ms

root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.4.19
PING 10.244.4.19 (10.244.4.19) 56(84) bytes of data.
64 bytes from 10.244.4.19: icmp_seq=1 ttl=62 time=3.05 ms
```

---

## Step 13 — Test Traffic to the Application

From the OCP Provisioner web shell, curl the VS IP with the correct Host header:

```bash
curl http://10.1.10.100 -H "Host: nginx-demo.cis.lab"
```

Expected output:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
...
</html>
```

We now have a BIG-IP HA pair with identical Virtual Server configurations — both devices are fault tolerant and back each other up.

---

## Step 14 — Test HA Failover

#### Find the active BIG-IP

Log in to the web shell of both BIGIP-OC devices:

![webshell](/img/03-webshell.png)

Run the following command to determine which device is active:

```text
tmsh show sys failover
# Failover active for 1d 23:19:21
```

#### Reboot the active device and test immediately

On the **active** BIG-IP:

```text
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# quit
[root@BIGIP-OC-02:Active:In Sync] config # reboot
```

**Immediately** test from the OCP Provisioner:

```bash
curl http://10.1.10.100 -H "Host: nginx-demo.cis.lab"
```

You should still get the nginx welcome page — the standby BIG-IP has taken over.

#### Verify the failover

On the other BIG-IP device:

```text
root@(BIGIP-OC-01)(cfg-sync Disconnected)(Active)(/Common)(tmos)# show sys failover
Failover active for 0d 00:01:12
```

Notice it has only been active for about 1 minute — confirming the failover occurred successfully.

> The BIG-IP HA pair provides fault tolerance — both devices share the same VS configuration, so if one fails, the other continues serving traffic seamlessly.