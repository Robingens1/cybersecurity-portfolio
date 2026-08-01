# SOC Lab: Detecting SSH Brute-Force Attacks with Wazuh SIEM

## Objective
To deploy an enterprise-grade SIEM solution (Wazuh) to monitor a Linux endpoint, simulate an automated credential-stuffing/brute-force attack, and analyze the resulting security telemetry to validate detection mechanisms.

## Skills & Tools Demonstrated
* **SIEM/EDR Engineering:** Wazuh Server deployment, agent configuration, and security telemetry engineering.
* **Log & Event Analysis:** Interpreting Linux `syslog` and `sshd` authentication logs.
* **Threat Simulation:** Executing automated credential attacks using Hydra.
* **Framework Mapping:** Mapping real-time alerts to the MITRE ATT&CK framework (T1110 - Brute Force).

## Environment Architecture
* **SIEM Manager:** Wazuh Server (Amazon Linux 2023) | IP: `192.168.64.129`
* **Monitored Endpoint:** Kali Linux VM running Wazuh Agent | IP: `192.168.64.128`
* **Network Switch:** VMware Private NAT Network

---

## Project Execution Steps

### Phase 1: Endpoint Enrollment & Telemetry Verification
1. Provisioned the Wazuh Manager instance via an OVA deployment within VMware.
2. Resolved initial service timeouts by scaling virtual hardware to **4GB RAM / 2 CPU Cores** and applying an extended systemd service timeout override (`TimeoutStartSec=900`).
3. Installed the `.deb` Wazuh agent package on the Kali Linux endpoint.
4. Verified that the agent successfully connected to the manager and began streaming logs over the local NAT subnet.

<img width="1164" height="601" alt="image" src="https://github.com/user-attachments/assets/1c5f04c3-d3ef-46e5-8e16-84959e7c442a" />

<img width="1363" height="593" alt="Wazuh_ThreatHunt_Dashboard_B4-Attack" src="https://github.com/user-attachments/assets/9025cb48-c2e5-481d-a86c-d27bd6a1aa8d" />

<img width="814" height="512" alt="image" src="https://github.com/user-attachments/assets/aae1c303-f1ee-4300-894a-100f4586cfa1" />

<img width="1349" height="551" alt="kali-Linux_Endpoint_after_successful_connection" src="https://github.com/user-attachments/assets/7a5c0994-af45-42b1-b3b8-38bbe95de427" />

*Caption: Wazuh dashboard confirming the Kali Linux endpoint is Active and streaming telemetry.*
---

### Phase 2: Threat Simulation (SSH Brute Force)
To simulate an active adversary attempting unauthorized access, an automated dictionary attack was executed against the local SSH daemon using **Hydra**:

```bash
hydra -l non_existent_user -P /usr/share/wordlists/fasttrack.txt ssh://127.0.0.1 -t 4 -V
```
<img width="1050" height="458" alt="image" src="https://github.com/user-attachments/assets/c6658577-11a5-4ecf-bfaa-df73a740735a" />
<img width="1366" height="553" alt="image" src="https://github.com/user-attachments/assets/26163ebf-aab2-4775-858a-e1c21953f837" />
<img width="1366" height="633" alt="image" src="https://github.com/user-attachments/assets/914447cc-dc68-44aa-95a2-3fc6656ad166" />

*Caption: Kali Linux terminal output confirming local IP configuration, wordlist generation, and successful threat simulation utilizing THC Hydra against the target SSH service via a 5-line dictionary attack payload.*

The attack rapidly flooded the authentication subsystem, generating sequence login failures by cycling passwords against a non-existent user profile to trigger detection thresholds.

---

### Phase 3: Telemetry Analysis & Incident Triage
Upon refreshing the Wazuh Threat Hunting module, an immediate anomaly spike was identified.

#### High-Level Dashboard Insights
* **Authentication Failures:** Spiked instantly to **76 hits**.
* **Total Alert Volume:** Reached **126 events** within a tight 20-minute window.
* **MITRE ATT&CK Mapping:** Successfully categorized under **T1110 (Brute Force / Password Guessing)**.

<img width="1366" height="540" alt="Security telemetry visualization showing the massive spike in authentication failures during the Hydra execution window" src="https://github.com/user-attachments/assets/d623c0d1-10d8-4945-847c-1baa0138a778" />

<img width="1366" height="594" alt="Wazuh_ThreatHunt_Dashboard_After-Attack2" src="https://github.com/user-attachments/assets/899c2b5d-8570-40bc-88f0-ab4e0b1ca8bb" />

*Caption: Security telemetry visualization showing the massive spike in authentication failures during the Hydra execution window.*

#### Granular Log Deep-Dive
Drilling into the raw event logs confirmed the exact mechanics of the attack vectors:
* **Rule ID 5710 (`sshd: Attempt to login using a non-existent user`):** Triggered continuously as Hydra targeted invalid account names.
* **Rule ID 2502 (`syslog: User missed the password more than one time`):** Escalated to a **Level 10 Severity Alert** due to the high frequency of failures originating from a single source.

<img width="1366" height="632" alt="Wazuh_ThreatHunt_Event_Dashboard_B4-Attack" src="https://github.com/user-attachments/assets/d7f23da3-3ae9-4a25-9f8b-ab6405790aa4" />

<img width="1366" height="642" alt="Wazuh_ThreatHunt_Event_Dashboard_After-Attack" src="https://github.com/user-attachments/assets/99275a53-031d-4970-bd4b-7222919c721a" />

<img width="1365" height="543" alt="Wazuh_ThreatHunt_Event_Dashboard_After-Attack_logs1" src="https://github.com/user-attachments/assets/7c8d00eb-9232-4916-b618-c359b8239b44" />

<img width="1364" height="533" alt="image" src="https://github.com/user-attachments/assets/211f34d6-18ad-4d58-8f9b-fbe2c543ec4d" />

*Caption: Raw log table showing active parsing of Rule 5710 and Rule 2502 events by the Wazuh manager.*

---

## Defensive Recommendations & Remediation
To mitigate this vector in a real-world enterprise environment, the following controls should be introduced:
1. **Implement Account Lockout Policies:** Configure `faillock` on the Linux endpoint to temporarily freeze user accounts after 5 consecutive failed attempts.
2. **Deploy Fail2ban:** Automatically block network traffic at the firewall layer (iptables) for source IPs generating excessive Rule 5710/2502 triggers.
3. **Enforce Public Key Authentication:** Disable password authentication entirely in `/etc/ssh/sshd_config` and change the default SSH port (22) to a non-standard port to reduce automated noise.
4. **Restrict SSH User Access:** Modify `/etc/ssh/sshd_config` to include the `AllowUsers` directive (e.g., `AllowUsers safeuser`). This ensures that even if a threat actor guesses a password for an account like `robinson`, the SSH daemon will explicitly drop the connection unless that specific username is pre-whitelisted.

