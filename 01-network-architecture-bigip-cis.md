# Network Architecture — BIG-IP + OpenShift + k3s (CIS + IPAM)

This document describes the network layout used for the CIS lab environment.

The environment contains:

- **2 BIG-IP HA pairs**
  - BIG-IP HA pair for **OpenShift**
  - BIG-IP HA pair for **k3s**
- Dedicated VLANs per cluster
- Shared CIS management network
- Floating Self IPs used for HA traffic groups

---

# 1. High-Level Topology

```
                    +-------------------------+
                    |      CIS Controllers    |
                    | (OpenShift / k3s Pods) |
                    +------------+------------+
                                 |
                                 | 10.1.30.0/24 (cis_mgmt)
                                 |
         ----------------------------------------------------------
                    Shared CIS Management Network
         ----------------------------------------------------------

        OPENSHIFT STACK                          K3S STACK
  ---------------------------              ---------------------------
  BIGIP-OC-01   BIGIP-OC-02                BIGIP-K3S-01 BIGIP-K3S-02
      |              |                         |              |
      |              |                         |              |
   10.1.10.0/24 (oc_data)                 10.1.20.0/24 (k3s_data)
      |                                        |
   OpenShift Cluster                        k3s Cluster
```

---

# 2. Network Segments

| Network | Purpose | Subnet |
|---|---|---|
| `cis_mgmt` | CIS → BIG-IP management/control plane | `10.1.30.0/24` |
| `oc_data` | OpenShift application data / VIPs | `10.1.10.0/24` |
| `k3s_data` | k3s application data / VIPs | `10.1.20.0/24` |

---

# 3. BIG-IP OpenShift HA Pair

## 3.1 BIGIP-OC-01 (Standby)

### VLANs

| VLAN | Interface | Tag |
|---|---|---|
| `oc_data` | 1.1 | 4004 |
| `cis_mgmt` | 1.2 | 4005 |

### Self IPs

| Name | Address | Type | Traffic Group |
|---|---|---|---|
| `oc_data` | 10.1.10.20/24 | Non-floating | local-only |
| `cis_mgmt` | 10.1.30.4/24 | Non-floating | local-only |
| `oc_data_float` | 10.1.10.200/24 | Floating | traffic-group-1 |
| `cis_mgmt_float` | 10.1.30.201/24 | Floating | traffic-group-1 |

---

## 3.2 BIGIP-OC-02 (Active)

### VLANs

| VLAN | Interface | Tag |
|---|---|---|
| `oc_data` | 1.1 | 4030 |
| `cis_mgmt` | 1.2 | 4031 |

### Self IPs

| Name | Address | Type | Traffic Group |
|---|---|---|---|
| `oc_data` | 10.1.10.21/24 | Non-floating | local-only |
| `cis_mgmt` | 10.1.30.5/24 | Non-floating | local-only |
| `oc_data_float` | 10.1.10.200/24 | Floating | traffic-group-1 |
| `cis_mgmt_float` | 10.1.30.201/24 | Floating | traffic-group-1 |

---

### OpenShift BIG-IP HA Notes

- Floating IPs are shared between both devices.
- CIS should connect using:

```
10.1.30.201  (cis_mgmt_float)
```

- VIPs created by CIS/IPAM will live on:

```
10.1.10.0/24 (oc_data)
```

---

# 4. BIG-IP k3s HA Pair

## 4.1 BIGIP-K3S-01

### VLANs

| VLAN | Interface | Tag |
|---|---|---|
| `k3s_data` | 1.1 | 4006 |
| `cis_mgmt` | 1.2 | 4007 |

### Self IPs

| Name | Address | Type | Traffic Group |
|---|---|---|---|
| `k3s_data` | 10.1.20.4/24 | Non-floating | local-only |
| `cis_mgmt` | 10.1.30.6/24 | Non-floating | local-only |
| `cis_mgmt_float` | 10.1.30.200/24 | Floating | traffic-group-1 |

---

## 4.2 BIGIP-K3S-02

### VLANs

| VLAN | Interface | Tag |
|---|---|---|
| `k3s_data` | 1.1 | 4008 |
| `cis_mgmt` | 1.2 | 4007 |

### Self IPs

| Name | Address | Type | Traffic Group |
|---|---|---|---|
| `k3s_data` | 10.1.20.5/24 | Non-floating | local-only |
| `cis_mgmt` | 10.1.30.7/24 | Non-floating | local-only |
| `k3s_data_float` | 10.1.20.200/24 | Floating | traffic-group-1 |
| `cis_mgmt_float` | 10.1.30.200/24 | Floating | traffic-group-1 |

---

### k3s BIG-IP HA Notes

- CIS should connect via:

```
10.1.30.200 (cis_mgmt_float)
```

- VIPs allocated by IPAM should come from:

```
10.1.20.0/24 (k3s_data)
```

---

# 5. Traffic Flow Model

## Control Plane (CIS → BIG-IP)

```
CIS Pod
   |
   | HTTPS (REST API)
   v
cis_mgmt_float (10.1.30.x)
```

Used for:

- AS3 declarations
- LTM object creation
- pool/member updates
- IPAM coordination

---

## Data Plane (Client → BIG-IP → Cluster)

```
Client
   |
   v
VIP (oc_data / k3s_data)
   |
BIG-IP
   |
Pool Members (Cluster Nodes / Pods)
```

---

# 6. Design Decisions

### Separate BIG-IP pair per cluster

Benefits:

- Isolation between environments
- Independent failover domains
- Easier troubleshooting
- Cleaner IPAM pools

---

### Shared CIS Management Network

All BIG-IPs expose CIS management via:

```
10.1.30.0/24
```

Advantages:

- Simplifies CIS configuration
- Consistent automation logic
- Easier lab scaling

---

# 7. HA Model Summary

| Stack | Floating Mgmt | Floating Data |
|---|---|---|
| OpenShift | 10.1.30.201 | 10.1.10.200 |
| k3s | 10.1.30.200 | 10.1.20.200 |

---

# 8. Next Steps (Guide Flow)

1. Configure BIG-IP base networking
2. Configure HA + traffic-group
3. Deploy IPAM controller
4. Deploy CIS
5. Deploy sample app
6. Create VirtualServer CR
7. Validate VIP creation
8. Perform HA failover test
