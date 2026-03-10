# F5 GSLB — Multi-Cluster DNS Load Balancing Guide

> **Purpose:** DNS-based high availability and load balancing between the OpenShift and K3S clusters
> **Method:** F5 BIG-IP DNS (GTM) with ExternalDNS CRDs and ratio-based load balancing
> **Hostname:** `nginx-demo.cis.lab`
> **OC site VS IP:** `10.1.10.100` — K3S site VS IP: `10.1.20.100`

---

## Why GSLB?

By design, we always want to keep our network resilient and highly available. We achieve this through several mechanisms:

- **BIG-IP HA pairs** — each cluster has two BIG-IP devices that back each other up. This is a network-level HA mechanism that requires Layer 3 connectivity between the devices.
- **But what if** we have the same application running on two clusters that don't have any network connectivity between them?

This is where **GSLB** (Global Server Load Balancing) comes in. GSLB uses **DNS** as the high availability and load balancing mechanism. When a client queries the DNS name, the BIG-IP DNS system evaluates the health of each site and returns the appropriate IP address based on the configured load balancing method — whether that's availability, geographic proximity, or in our case, a **ratio-based distribution**.

---

## Architecture Overview

```
                          DNS Query: nginx-demo.cis.lab
                                    |
                            BIG-IP DNS (GTM)
                           /                  \
                     OC-Cluster              K3S-Cluster
                    (Datacenter)            (Datacenter)
                    /          \                  |
              BIGIP-OC-01  BIGIP-OC-02     BIGIP-K3S-01/02
                    |                             |
              VS: 10.1.10.100              VS: 10.1.20.100
                    |                             |
              OpenShift Pods               K3S Pods
              (nginx-demo)                (nginx-demo)
```

Both clusters are running the same `nginx-demo` application with a VirtualServer CRD using `ipamLabel: test`. The BIG-IP DNS system ties them together via a Wide IP that resolves `nginx-demo.cis.lab` to one (or both) of the site VIPs based on health and ratio.

---

## Prerequisites

Before starting, ensure the following are in place:

- The **nginx-demo** VirtualServer is deployed and healthy on **both** clusters (OC: `10.1.10.100`, K3S: `10.1.20.100`)
- CIS is running on **both** clusters with `--ipam=true` and GTM parameters configured (`--gtm-bigip-url`, `--gtm-bigip-username`, `--gtm-bigip-password`)
- The ExternalDNS CRD is already installed (included in the `customresourcedefinitions.yml` we applied earlier)

---
## step 0 - Change the gtm server objects of k3s

Open both BIGIP-K3S-01 and 02 and run:

```bash
tmsh
modify gtm server BIGIP-K3S-01 devices replace-all-with { bigip-k3s-01 { addresses replace-all-with { 10.1.20.31 { } } } }

modify gtm server BIGIP-K3S-02 devices replace-all-with { bigip-k3s-02 { addresses replace-all-with { 10.1.20.32 { } } } }

save sys config
```


## Step 1 — Verify the GTM Infrastructure on the BIG-IP

Open the TMUI of one of the BIG-IP devices and navigate to **DNS → GSLB → Data Centers**:

![data centers](/img/04-dc-section.png)

You should see two Data Centers:

- **OC-Cluster** — contains BIGIP-OC-01 and BIGIP-OC-02
- **K3S-Cluster** — contains BIGIP-K3S-01 and BIGIP-K3S-02

Each Data Center groups a set of BIG-IP servers. For a site to be considered "alive" by GSLB, the servers within its Data Center must be reachable and healthy.

Navigate to **DNS → GSLB → Servers** and verify all four servers are present with `virtual-server-discovery: enabled`. This is critical — it allows the BIG-IP DNS to automatically discover the Virtual Servers that CIS creates on the LTM side.

> **Note:** The GTM sync-group between the OC and K3S BIG-IP pairs is already configured. Any GSLB changes made on one device will automatically sync to the others.

---

## Step 2 — Create the ExternalDNS CRD on the OpenShift Cluster

From the **ocp-provisioner** web shell, apply the ExternalDNS CRD.

Key points about this YAML:

- `domainName` must match the `host` field in the VirtualServer CRD (`nginx-demo.cis.lab`)
- `dataServerName` must match the GSLB Server name as configured on the BIG-IP (`/Common/BIGIP-OC-01`)
- `loadBalanceMethod: ratio` enables ratio-based distribution between pools

```bash
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: ExternalDNS
metadata:
  name: edns-nginx-oc
  namespace: default
  labels:
    f5cr: "true"
spec:
  domainName: nginx-demo.cis.lab
  dnsRecordType: A
  loadBalanceMethod: ratio
  pools:
    - dnsRecordType: A
      loadBalanceMethod: ratio
      dataServerName: /Common/BIGIP-OC-01
      monitor:
        type: http
        send: "GET / HTTP/1.1\r\nHost: nginx-demo.cis.lab\r\nConnection: close\r\n\r\n"
        recv: ""
        interval: 10
        timeout: 30
EOF
```

Verify:

```bash
oc get externaldns
```

---

## Step 3 — Create the ExternalDNS CRD on the K3S Cluster

From the **K3S server** web shell, apply the matching ExternalDNS CRD. This one points to the K3S BIG-IP server:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

cat <<'EOF' | kubectl apply -f -
apiVersion: cis.f5.com/v1
kind: ExternalDNS
metadata:
  name: edns-nginx-k3s
  namespace: kube-system
  labels:
    f5cr: "true"
spec:
  domainName: nginx-demo.cis.lab
  dnsRecordType: A
  loadBalanceMethod: ratio
  pools:
    - dnsRecordType: A
      loadBalanceMethod: ratio
      dataServerName: /Common/BIGIP-K3S-02
      monitor:
        type: http
        send: "GET / HTTP/1.1\r\nHost: nginx-demo.cis.lab\r\nConnection: close\r\n\r\n"
        recv: ""
        interval: 10
        timeout: 30
EOF
```

Verify:

```bash
kubectl get externaldns
```

---

## Step 4 — Verify the Wide IP on the BIG-IP

Navigate to **DNS → GSLB → Wide IPs** on the BIG-IP TMUI.

You should see a Wide IP for `nginx-demo.cis.lab` with:

- **Load Balancing Method:** Ratio
- **Two pools** — one pointing to the OC site VirtualServer (`10.1.10.100`) and one pointing to the K3S site VirtualServer (`10.1.20.100`)

Navigate to **DNS → GSLB → Pools** and verify both pools have green health status. If a pool shows red, check that the corresponding VirtualServer is healthy on that cluster.

> **Ratio tuning:** By default, both pools will have equal ratio (1:1). To adjust the ratio (e.g., send 80% of traffic to OC and 20% to K3S), you can modify the ratio values on the pool members via the BIG-IP TMUI under the pool properties.

---

## Step 5 — Test DNS Resolution

From any BIG-IP web shell, query the Wide IP using the GTM listener:

```text
dig @10.1.1.12 nginx-demo.cis.lab A
```

You should see an A record returned — either `10.1.10.100` (OC) or `10.1.20.100` (K3S) depending on the ratio and health status.

Run the query multiple times and observe that responses alternate between the two IPs based on the ratio configuration:

```text
dig @10.1.1.12 nginx-demo.cis.lab A +short
dig @10.1.1.12 nginx-demo.cis.lab A +short
dig @10.1.1.12 nginx-demo.cis.lab A +short
```

---

## Step 6 — Test Application Traffic via GSLB

From the **ocp-provisioner** web shell, test traffic through each site VIP:

```bash
# Test OC site directly
curl http://10.1.10.100 -H "Host: nginx-demo.cis.lab"

# Test K3S site directly
curl http://10.1.20.100 -H "Host: nginx-demo.cis.lab"
```

Both should return the nginx welcome page.

---

## Step 7 — Test Site Failover

To simulate a site failure, delete the VirtualServer on one of the clusters and observe GSLB behavior:

#### Take down the OC site

```bash
# On the OCP provisioner
oc delete vs nginx-demo-vs
```

Wait approximately 30 seconds for the health monitor to detect the failure, then query DNS again:

```text
dig @10.1.1.12 nginx-demo.cis.lab A +short
```

The response should now **only** return `10.1.20.100` (K3S) — GSLB has removed the unhealthy OC site from the DNS rotation.

#### Restore the OC site

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

After the health monitor marks the OC site healthy again, both IPs will be returned in DNS responses.

---

## How It All Fits Together

| Layer | Mechanism | What It Protects Against |
|-------|-----------|------------------------|
| BIG-IP HA Pair | Active/Standby failover | Single device failure within a site |
| CIS + IPAM | Automated VS creation + IP allocation | Manual configuration errors |
| GSLB (DNS) | Cross-site health monitoring + ratio LB | Entire site failure, load distribution |

The combination of these three layers provides end-to-end resilience: if a single BIG-IP fails, the HA pair handles it. If an entire site goes down, GSLB stops sending DNS responses to that site. And with ratio-based load balancing, you can control how much traffic each site receives during normal operations.

---

## Troubleshooting Checklist

| Symptom | Fix |
|---------|-----|
| Wide IP not created | Ensure the ExternalDNS `domainName` matches the VirtualServer `host` field exactly. CIS only creates the Wide IP when both exist. |
| Pool member missing | Verify `virtual-server-discovery: enabled` on the GTM server object. Check: `tmsh list gtm server` |
| Pool shows red/unhealthy | The underlying VS is down or the health monitor can't reach it. Check VS status: `oc get vs` or `kubectl get vs` |
| DNS returns only one IP | The other site's pool is unhealthy. Check the pool status in TMUI under DNS → GSLB → Pools. |
| `dig` returns no answer | Verify the GTM listener is active: `tmsh list gtm listener`. Ensure you're querying the correct BIG-IP IP. |
| ExternalDNS CRD not processed | Check CIS logs: `kubectl logs -n kube-system -l app=f5-bigip-ctlr`. Look for `Processing WideIP` entries. |
| Ratio not working as expected | Adjust ratio values on pool members in TMUI: DNS → GSLB → Pools → select pool → Members. |