# Cisco Secure Workload — AnyConnect NVM Integration Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [What CSW Gets from AnyConnect NVM](#3-what-csw-gets-from-anyconnect-nvm)
4. [Use Cases](#4-use-cases)
5. [Prerequisites](#5-prerequisites)
6. [Step A — Deploy AnyConnect NVM on Endpoints](#6-step-a--deploy-anyconnect-nvm-on-endpoints)
7. [Step B — Configure the AnyConnect Connector on CSW](#7-step-b--configure-the-anyconnect-connector-on-csw)
8. [Optional: LDAP Integration for User Labels](#8-optional-ldap-integration-for-user-labels)
9. [Verification](#9-verification)
10. [Limits](#10-limits)
11. [Troubleshooting](#11-troubleshooting)
12. [Related Resources](#12-related-resources)

---

## 1. Overview

**Cisco AnyConnect Network Visibility Module (NVM)** provides endpoint-level telemetry — process-level flows, user identity, device context, and FQDN of destinations — from managed endpoints running **Cisco AnyConnect Secure Mobility Client**.

Unlike the CSW deep-visibility agent (which requires installation on each workload), AnyConnect NVM uses the existing AnyConnect client already deployed on laptops, desktops, and VPN endpoints. Flow records are exported in **IPFIX format** to the **AnyConnect Connector** on a CSW Edge appliance, giving CSW full flow visibility from user endpoints both **on-premises and off-premises (VPN)**.

### Why it matters
- Endpoint visibility **without deploying a separate CSW agent** on each device
- Captures **process-level flow context** — which application initiated the connection
- Extends CSW visibility to **roaming and remote users** connected via VPN
- Adds **user identity** to flows via LDAP/AD integration

### Minimum versions required

| Component | Minimum Version |
|-----------|----------------|
| Cisco AnyConnect NVM | 4.2+ (NVM record type support) |
| Cisco AnyConnect NVM (agent version reporting) | 4.9+ |
| Cisco Secure Workload | 3.x+ |
| NVM IPFIX collector port | UDP/TCP **4739** |

---

## 2. Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     Enterprise + Remote Endpoints                   │
│                                                                     │
│  ┌──────────────────┐        ┌──────────────────────────────────┐  │
│  │  Laptop / Desktop│        │   Remote (VPN) Endpoint           │  │
│  │  AnyConnect NVM  │        │   AnyConnect NVM                  │  │
│  │  (On-prem)       │        │   (Off-prem, VPN tunnel)          │  │
│  └────────┬─────────┘        └──────────┬───────────────────────┘  │
│           │  IPFIX (UDP 4739)           │  IPFIX via VPN tunnel     │
│           └──────────────────┬──────────┘                          │
│                              ▼                                      │
│             ┌────────────────────────────────────┐                 │
│             │     CSW Edge Appliance              │                 │
│             │     (AnyConnect Connector)          │                 │
│             │                                     │                 │
│             │  • Receives IPFIX from endpoints    │                 │
│             │  • Registers endpoints as agents    │                 │
│             │  • Forwards flows to CSW collectors │                 │
│             │  • Labels endpoints via LDAP        │                 │
│             └────────────────┬───────────────────┘                 │
│                              │                                      │
│             ┌────────────────▼───────────────────┐                 │
│             │  Cisco Secure Workload Cluster      │                 │
│             │  • Flow analysis & ADM              │                 │
│             │  • Endpoint inventory with labels   │                 │
│             │  • Policy based on user + process   │                 │
│             └────────────────────────────────────┘                 │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. What CSW Gets from AnyConnect NVM

AnyConnect NVM generates three IPFIX record types:

### Endpoint Record
Registers the device in CSW inventory:

| Field | Description | Example |
|-------|-------------|---------|
| UDID | Unique device identifier | `a1b2c3d4-...` |
| Hostname | Endpoint hostname | `LAPTOP-JSMITH` |
| OS Name | Operating system | `Windows 10` |
| OS Version | OS version | `10.0.19044` |
| Manufacturer | Device manufacturer | `Dell Inc.` |

### Interface Record
Network interface details per endpoint (IP addresses, MACs, interface type).

### Flow Record
Per-flow telemetry with full application and user context:

| Field | Description | Example |
|-------|-------------|---------|
| 5-tuple | src/dst IP:port + protocol | `10.20.30.41:50234 → 10.100.1.5:443 TCP` |
| Bytes in/out | Traffic volume | `12400 / 890` |
| Process name | Application that initiated the flow | `chrome.exe` |
| Process path | Full path to the binary | `C:\Program Files\Google\Chrome\chrome.exe` |
| Username | Logged-in user | `corp\jsmith` |
| FQDN | Destination DNS name | `api.internal.corp.com` |

---

## 4. Use Cases

### Use Case 1 — Application-Level Microsegmentation from Endpoints
**Scenario:** Understand which applications on endpoints are connecting to sensitive servers.

CSW ADM shows `chrome.exe` on Finance laptops connecting to production database APIs — helping security teams identify shadow access patterns and build precise allow policies.

### Use Case 2 — Remote User Visibility (VPN Endpoints)
**Scenario:** Security team needs full flow visibility for VPN-connected remote workers.

AnyConnect NVM tunnels IPFIX records through the VPN connection to the AnyConnect Connector on-premises. Remote users appear in CSW inventory with the same flow fidelity as on-premise workloads.

### Use Case 3 — User-Based Policy with Process Context
**Scenario:** Allow only specific applications on endpoints to reach production app servers.

Using LDAP integration, flows are labeled with AD username. Combine with process labels to write policies like:
- `Finance-User-Endpoints + process=TradingApp → Trading-Servers: ALLOW TCP 8443`
- `Any-Endpoint + process=chrome.exe → Trading-Servers: DENY`

### Use Case 4 — Shadow IT Detection
**Scenario:** Detect unauthorized SaaS or cloud usage from corporate endpoints.

FQDN of destination is captured in flow records. Flows to unexpected external destinations from sensitive endpoint scopes trigger CSW Traffic Alerts.

---

## 5. Prerequisites

### On endpoints
- [ ] Cisco AnyConnect Secure Mobility Client **4.2+** installed
- [ ] **Network Visibility Module (NVM)** enabled in AnyConnect
- [ ] NVM profile configured with IPFIX collector pointing to CSW Edge appliance IP on **UDP 4739**
- [ ] NVM profiles deployed via ASA/FTD headend, ISE, or MDM (Intune/JAMF)

### CSW / infrastructure
- [ ] **CSW Edge appliance** deployed and registered (AnyConnect connector runs on Edge)
- [ ] Edge appliance reachable from endpoints on **UDP port 4739**
- [ ] VPN headend (ASA/FTD) configured to allow IPFIX passthrough on port 4739
- [ ] VRF configuration on Edge appliance covers the endpoint subnet(s)
- [ ] (Optional) LDAP/AD server reachable from Edge appliance for user label enrichment

---

## 6. Step A — Deploy AnyConnect NVM on Endpoints

### A1 — Create the NVM profile

The NVM profile is an XML file that tells AnyConnect NVM where to send IPFIX records.

Create `NVM_Profile.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<NVMProfile xmlns="http://schemas.xmlsoap.org/encoding/"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <CollectorConfig>
    <Collector>
      <CollectorAddress>10.x.x.x</CollectorAddress>  <!-- CSW Edge appliance IP -->
      <CollectorPort>4739</CollectorPort>
      <CollectorProtocol>UDP</CollectorProtocol>
    </Collector>
  </CollectorConfig>
  <FlowExportConfig>
    <FlowInactiveTimeout>15</FlowInactiveTimeout>
    <FlowActiveTimeout>60</FlowActiveTimeout>
  </FlowExportConfig>
</NVMProfile>
```

Replace `10.x.x.x` with the IP of your CSW Edge appliance.

### A2 — Deploy the NVM profile

**Option A — Via Cisco ASA/FTD headend:**
1. Upload `NVM_Profile.xml` to the ASA/FTD group policy
2. Under **AnyConnect Client Profile**, select NVM profile type and upload the XML
3. Assign the profile to the connection profile / tunnel group

**Option B — Via Cisco ISE (MDM-managed):**
1. In ISE, navigate to **Policy > Policy Elements > Results > Client Provisioning**
2. Create an AnyConnect Configuration with the NVM profile included
3. Assign via posture policy or provisioning portal

**Option C — Manual (lab/testing):**
Copy `NVM_Profile.xml` to the endpoint:
- Windows: `C:\ProgramData\Cisco\Cisco AnyConnect Secure Mobility Client\NVM\`
- macOS: `/opt/cisco/anyconnect/nvm/`

### A3 — Verify NVM is sending IPFIX

On the endpoint, check AnyConnect NVM statistics:
- **Windows:** Right-click AnyConnect tray → Statistics → NVM tab
- Confirm "Flows Exported" counter is incrementing

On the CSW Edge appliance, use:
```bash
tcpdump -i any -n port 4739
```
You should see IPFIX packets arriving from endpoint IPs.

---

## 7. Step B — Configure the AnyConnect Connector on CSW

### B1 — Navigate to connector configuration

1. In CSW UI: **Manage > Virtual Appliances**
2. Select your **Edge appliance**
3. Click **Connectors** tab → **+ Add Connector** → **AnyConnect**

### B2 — Configure connector settings

| Field | Value |
|-------|-------|
| **Connector Name** | e.g., `anyconnect-nvm-connector` |
| **Connector Type** | AnyConnect |
| **LDAP Configuration** | (Optional — see Section 8) |

### B3 — Configure VRF assignment

AnyConnect endpoints must be associated with a VRF:
1. Navigate to **Manage > Workloads > Agents > Configuration tab**
2. Under **Agent Remote VRF Configurations** → **Create Config**
3. Provide:
   - VRF Name
   - IP subnet CIDR covering endpoint addresses
   - Port range (typically `4739-4739`)

### B4 — Apply configuration

Click **Test and Apply**. The connector will:
1. Start listening for IPFIX on UDP 4739
2. Register as a proxy agent with the CSW cluster
3. Begin forwarding endpoint flows to CSW collectors

---

## 8. Optional: LDAP Integration for User Labels

The AnyConnect connector can enrich endpoint flows with LDAP/AD user attributes, labeling each endpoint IP with the logged-in user's AD attributes.

| Field | Description |
|-------|-------------|
| LDAP Server | FQDN of AD/LDAP server |
| LDAP Port | 636 (LDAPS) or 389 |
| Bind Username | Service account DN |
| Bind Password | Service account password |
| Username Attribute | Attribute mapping to NVM username (e.g., `sAMAccountName`) |
| Additional Attributes | Up to 6 AD attributes (e.g., `department`, `title`, `manager`) |

With LDAP configured, CSW inventory shows:
```
IP: 10.20.30.41
  user/username   = jsmith
  user/department = Finance
  user/title      = Analyst
```

---

## 9. Verification

### Check connector status
**Manage > Virtual Appliances > [Edge] > Connectors**
AnyConnect connector should show **Status: Active**

### Check endpoint inventory
1. **Inventory > Workloads** → search for an endpoint IP
2. Confirm endpoint appears with type `AnyConnect Agent`
3. If LDAP configured, confirm user labels are present

### Check flow data
1. **Observe > Traffic** → filter by source = endpoint subnet
2. Confirm flows appear with process context (`proc/name`, `proc/path`)

### Verify IPFIX reception on Edge appliance
```bash
# On Edge appliance:
netstat -anu | grep 4739          # confirm port is listening
tail -f /usr/local/tet/log/anyconnect-connector.log
```

---

## 10. Limits

| Metric | Limit |
|--------|-------|
| AnyConnect connectors per Edge appliance | 1 |
| AnyConnect connectors per tenant | 1 |
| IPFIX collector port | UDP 4739 (fixed) |
| Maximum endpoints per connector | Platform dependent |
| Agent check-in interval | Every 20–30 minutes |

---

## 11. Troubleshooting

| Symptom | Check |
|---------|-------|
| Endpoints not appearing in inventory | Verify IPFIX packets reaching Edge on UDP 4739; check VRF assignment matches endpoint subnet |
| No user labels | Confirm LDAP configuration is correct; verify service account has read permissions |
| Flow data missing process context | Confirm NVM version ≥ 4.2; check NVM profile is correctly deployed on endpoint |
| Connector shows disconnected | Restart connector from CSW UI; check Edge appliance logs |
| VPN endpoints not showing | Verify VPN headend allows IPFIX (UDP 4739) through the tunnel |

---

## 12. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [csw-ise-integration](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid: user-identity–aware policy via pxGrid 2.0 | Identity + AnyConnect pairing |
| [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL flow ingestion from Cisco Secure Firewall | Network-layer visibility |
| [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk SIEM | SecOps alerting |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents on Linux/Windows workloads | Server-side agent deployment |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [CSW-Operations-Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops scripts: health checks, reporting | Ongoing operations |

> **Suggested pairing:** Deploy **ISE connector** alongside **AnyConnect NVM** for full user + device + flow context in a single CSW tenant.

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
