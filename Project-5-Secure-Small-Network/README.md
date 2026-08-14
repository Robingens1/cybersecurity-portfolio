# Capstone Project: Design, Simulation, and Hardening of a Secure Small Network

## 📌 Project Overview
This capstone project implements a multi-layered defense-in-depth security architecture for a small office network. The layout isolates untrusted environments from production servers and administrative domains. 

The lab environment was fully simulated inside **VMware Workstation** utilizing a **pfSense Community Edition Next-Generation Firewall**, a **Kali Linux** administration/penetration testing workstation, and simulated internal zones.
<img width="589" height="451" alt="Kali_Configuration" src="https://github.com/user-attachments/assets/f9fb185e-d117-4c88-913d-6e966c6ca3eb" />

---

## 🗺️ 1. Network Topology & Interface Architecture
The network infrastructure isolates different trust zones using physical and virtual segmentation strategies:
*   **WAN Boundary Interface (`em0`):** Automatically routes to the upstream provider gateway.
*   **LAN Management Interface (`em1`):** Serves as the primary enterprise gateway (`192.168.1.1/24`).
*   **Asset Configuration Schema:**
    *   **Core Security Gateway (pfSense):** `192.168.1.1`
    *   **Admin Workstation (Kali Linux):** `192.168.1.20`
    *   **Secure Web Server (Production Zone):** `192.168.1.10`
    *   **IoT Security Camera Subnet (Untrusted Zone):** Assigned to isolated segments (`192.168.1.30` / `192.168.30.0/24`).

<img width="483" height="492" alt="secure-small-network-topology png" src="https://github.com/user-attachments/assets/bd7338a6-ef30-4c7e-9547-401eddfc5a3c" />

**Topology Filename:** `secure-small-network-topology.png`

---

## 🛡️ 2. Step-by-Step Security Controls Implementation

### Control 1: Control Plane Hardening & Initial Configuration
To mitigate risk against default software credentials and weak baseline setups, the administrative interface was fully hardened.
*   **Action Executed:** Walked through the system initialization wizard. Updated host identifiers, synchronized network time records, established reliable external DNS forwarders (`1.1.1.1` / `8.8.8.8`), and replaced default configuration passwords with enterprise-grade credentials.
*   **Verification Image:**
  <img width="1089" height="496" alt="01-pfsense-initial-wizard-complete png" src="https://github.com/user-attachments/assets/b79d502e-2b38-4086-bce7-9d861fa94011" />

    ![pfSense Setup Wizard Finalized](01-pfsense-initial-wizard-complete.png)

### Control 2: Traffic Restriction & Firewall Filtering
To control internal network communication and prevent lateral device exposure, traffic boundaries were built directly on the interface ruleset.
*   **Action Executed:** Configured an explicit **Block** policy at the top of the interface firewall rule queue. This drops any traffic originating from the unprivileged host **`192.168.1.30`** bound for other zones, stopping internal movement if an endpoint is compromised.
*   **Verification Image:**
<img width="1114" height="525" alt="02-pfsense-firewall-lan-rules png" src="https://github.com/user-attachments/assets/967cd5e3-bfb1-4a37-a0ff-180e50047366" />

     ![Firewall Interface Traffic Rules Table](02-pfsense-firewall-lan-rules.png)

### Control 3: Protocol Analysis & Wireshark Traffic Inspection
Deep packet inspection was executed to observe how baseline network layer communications and network scans traverse internal networks.
*   **Action Executed:** Initialized a network sniffer (`Wireshark`) on interface `eth0` to evaluate live framing, tracking active Address Resolution Protocol (ARP) device mappings and tracking failing TCP handshake connection sequences.
*   **Verification Image:**
<img width="991" height="554" alt="04-kali-wireshark-packet-capture png" src="https://github.com/user-attachments/assets/12244ec7-cc95-4acb-b45f-6d01349e8c8f" />

    ![Wireshark Packet Analysis Captures](04-kali-wireshark-packet-capture.png)

### Control 4: Advanced Network Segmentation (VLAN Integration)
To properly separate untrusted IoT hardware on a single switch, virtual boundaries were introduced into the core interface setup.
*   **Action Executed:** Created a dedicated 802.1Q Virtual Local Area Network (**VLAN ID 30**) parented to the local interface, establishing a completely segmented sub-interface called **`IOT_ZONE`** operating on the `192.168.30.1/24` gateway.
*   **Verification Image:**
<img width="1164" height="582" alt="05-pfsense-network-segmentation-vlan png" src="https://github.com/user-attachments/assets/fba1c88d-5acd-41e5-9fc7-8fb744c8910b" />

    ![VLAN Interface Assignments Panel](05-pfsense-network-segmentation-vlan.png)

### Control 5: User Access Controls & Account Privilege Management
Following the Principle of Least Privilege, administrative capabilities were restricted by separating user accounts.
*   **Action Executed:** Created a new, non-root user role profile (`auditor_robin`) enforcing strong complexity requirements, separating general daily monitoring and log audits from the root `admin` account.
*   **Verification Image:**
<img width="1094" height="574" alt="06-pfsense-access-control-user-roles png" src="https://github.com/user-attachments/assets/161fe30c-a3e5-463b-b0e6-753263f6befb" />

    ![Hardened System User Management Interface](06-pfsense-access-control-user-roles.png)

---

## ☣️ 3. Simulated Threat & Incident Response Profile

### 🚨 Attack Vector: Network Reconnaissance and Profiling
An internal attacker attempts to scan the local subnet environment. Using the Kali Linux machine, the adversary drops into a terminal and executes an aggressive fast port scan (`nmap -F 192.168.1.1`) to locate open management pathways (such as HTTP, HTTPS, and DNS) on the security gateway.

### 🛡️ Defensive Response & Impact Mitigation
The network scan successfully uncovers the necessary administrative pathways from the authorized administrator station. However, because **Control 2 (Firewall Filtering)** and **Control 4 (VLAN Segmentation)** are actively running, if an identity or asset inside the untrusted `IOT_ZONE` tries to run a matching scan, the packets are immediately dropped by the firewall. This keeps your internal network entirely hidden from compromised network zones.
*   **Verification Image:**
<img width="656" height="266" alt="03-kali-nmap-recon-simulation png" src="https://github.com/user-attachments/assets/23165d31-3341-4755-93a4-e48db6630c04" />

    ![Nmap Attack Reconnaissance Scanning Artifact](03-kali-nmap-recon-simulation.png)

---

## 📝 4. Final Reflection & Strategic Upgrades
This architecture demonstrates how combining firewall filtering policies, strict role management, and solid network segmentation keeps small business environments safe.

**Next Milestones:** Future expansions will focus on integrating your active **Wazuh-SIEM** dashboard to aggregate firewall log files and trigger real-time alerts for unexpected configuration changes or repeated block events.

# 📋 Appendix: Small Office / Home Office (SOHO) Security Checklist

Use this simple, step-by-step checklist to secure any standard home or small office network framework:

- [ ] 1. Change Default Router Credentials
      Immediately change the factory-default admin username and password on your internet router to prevent automated script attacks.
- [ ] 2. Isolate IoT Devices (Smart TVs, Cameras, Smart Lights)
      Log into your router and place all IoT devices onto a dedicated "Guest Network" or separate VLAN so they cannot talk to your private computers.
- [ ] 3. Enable WPA3 or WPA2-AES Encryption
      Ensure your Wi-Fi network password utilizes strong WPA2/WPA3 encryption protocols. Never leave a network open without a password.
- [ ] 4. Disable Remote Management, Universal Plug and Play (UPnP), and WPS
      Turn off router features like UPnP and WPS, which allow external networks or malicious software to automatically open holes in your firewall.
- [ ] 5. Keep Firmware Updated
      Set your router, firewall, and smart devices to automatically download and install security updates to protect against newly discovered exploits.
- [ ] 6. Enforce Least Privilege Access Control
      Create standard, unprivileged user accounts for daily tasks and restrict full administrator account logins exclusively for configuration changes.
