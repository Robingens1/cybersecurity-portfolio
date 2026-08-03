# Project 2: Network Traffic Analysis, Forensic Triage & Protocol Recovery

## Project Overview
This project demonstrates how to perform deep network traffic analysis, forensic stream auditing, and diagnostic incident recovery using command-line packet recorders (`tcpdump`) and graphical network protocol analyzers (`Wireshark`). 

The objective of this lab is to intercept a simulated malware transfer request over the wire, trace unencrypted protocol redirections, diagnose tool-specific session terminations, and execute a successful remediation capture to isolate and map secure cryptographic TLSv1.3 metadata indicators.

### Core Competencies Developed
* Command-line packet capture and session overrides via `tcpdump`.
* System environment maintenance and process management (`killall`).
* Deep-dive network troubleshooting at the raw TCP layer (`ip.addr` and Layer 4 flags).
* Forensic discovery of application-specific request anomalies (e.g., `curl` redirect handling).
* Cryptographic handshake analysis and session metadata extraction (SNI auditing, TLS version mapping).
* Multi-stage session benchmarking (Comparing an unhandled termination against a full payload capture).

---

## Lab Architecture & Environment Check
* **Analysis Machine:** Kali Linux VM (Static IP: `192.168.64.128`)
* **Target Interface:** `eth0`
* **Local DNS Gateway:** `192.168.64.2`
* **Target Web Server Host:** `89.238.73.97`

---

## Step-by-Step Implementation Walkthrough

### Phase 1: Environment Setup & Tool Initialization
1. Powered on the network analysis environment and audited the active layer-2/layer-3 settings to confirm the target interface framework:
   ```bash
   ip a
   ```
   *Verified interface name `eth0` and confirmed static IP registration at `192.168.xx.xxx/24`.*

![Figure 1](images/1_interface_check.png)
*Figure 1: Verification of active network interface properties and local IP allocation via Terminal 1.*

2. Initialized system application repositories and updated the local packet analysis suite to guarantee robust parsing metrics:
   ```bash
   sudo apt update && sudo apt install -y wireshark tshark
   ```
   *Successfully fetched 74.9 MB of dependency packages from the official kali-rolling mirrors.*

![Figure 2](images/2_apt_initialization.png)
*Figure 2: Execution of package manager update strings to pull down updated core binaries.*

---

### Phase 2: Staging the Initial Network Capture
1. Opened **Terminal 1** on the Kali VM and initialized `tcpdump` to stream raw bytes passing through `eth0` directly to storage:
   ```bash
   sudo tcpdump -i eth0 -w /tmp/malware_download.pcap
   ```
2. Opened **Terminal 2** to simulate a web transfer request, targeting the standard EICAR anti-malware test signature domain over insecure HTTP:
   ```bash
   curl http://eicar.org -o /tmp/eicar_test.txt
   ```
3. **Forensic Discovery:** The curl transfer meter recorded exactly **266 bytes** received before the session closed out.

![Figure 3](images/3_initial_curl_transfer.png)
*Figure 3: Terminal 2 unhandled insecure request metrics highlighting immediate connection cutoff.*

4. Returned to Terminal 1 and broke the active recording track via `Ctrl + C`. Audited the resulting file payload footprint to ensure packet validation:
   ```bash
   ls -lh /tmp/malware_download.pcap
   ```
   *Confirmed exactly 179 packets (15 KB) were safely committed to disk.*

---

### Phase 3: Traffic Filtering & Threat Hunting Analysis
1. Launched the graphical workspace out of the shell framework into a separate decoupled background thread under PID 18132:
   ```bash
   wireshark /tmp/malware_download.pcap &
   ```
2. Audited the raw capture file before filtering, exposing internal background noise (DNS queries to `192.168.64.2`, Layer 2 ARP broadcasts, and unrelated `SYN`/`RST, ACK` scanning attempts directed at a neighboring node `192.168.64.129`).

![Figure 4](images/4_unfiltered_pcap_view.png)
*Figure 4: Analysis dashboard mapping out baseline local area background noise and protocol states.*

3. Applied an isolated `http` display filter to clear the interface and map out the web connection sequence:
   * **Packet #26:** Shows the Kali analysis endpoint (`192.168.64.128`) generating an unencrypted `GET / HTTP/1.1` request using user-agent `curl/8.18.0`.
   * **Packet #28:** Reveals the remote web server (`89.238.73.97`) responding with an explicit HTTP status code `301 Moved Permanently`.

![Figure 5](images/5_http_filter_dashboard.png)
*Figure 5: HTTP display filters capturing outbound request packet 26 and inbound redirect response packet 28.*

4. Reconstructed the plaintext session layer by applying the display filter `tcp.stream eq 5`. The conversation transcript revealed an Apache server header forcing an immediate upgrade direction via: `Location: https://eicar.org` alongside the 266-byte payload.

![Figure 6](images/6_tcp_stream_transcript.png)
*Figure 6: Reconstructed TCP conversation transcript confirming the forced secure protocol redirection path.*

---

### Phase 4: The Cryptographic Filter Mystery & Forensic Discovery
To trace the subsequent secure connection, attempts were made to apply standard sub-protocol display filters (`tls` and legacy `ssl`), but both queries yielded completely empty packet list rows. To determine why no secure traffic occurred, high-level structural assumptions were bypassed to review raw Layer 4 actions.

1. Applied a universal network layer filter to expose all raw frames exchanged with the web server:
   ```text
   ip.addr == 89.238.73.97
   ```
2. **The Discovery:** Scrolled to the absolute bottom of the output sequence to review the `Info` column. Packets **32 and 33** revealed that the connection to the remote host tore down completely with a `FIN, ACK` packet sequence immediately following the server's `301 Moved Permanently` response.
3. **The Root Cause:** By default, standard plain `curl` actions process an HTTP redirect and stop immediately—the utility **does not** follow the upgraded location path automatically. No secure TLS handshakes or file bytes ever traversed the interface during the initial recording window because the client app abandoned the request.

---

### Phase 5: Step-by-Step Recovery & Complete Capture Sequence
To properly complete the threat lifecycle simulation, a remediation phase was structured to force the client application into the secure zone and record the complete asset delivery.

1. Maintained a clean laboratory memory footprint by executing a strict terminal task-clearance to ensure no conflicting captures locked the interface:
   ```bash
   killall -9 wireshark
   ```
2. Initialized a fresh packet capture override file in **Terminal 1**:
   ```bash
   sudo tcpdump -i eth0 -w /tmp/malware_final.pcap
   ```
3. Executed the corrected threat simulation parameters in **Terminal 2**, appending the explicit `-L` (Follow Location) flag to force `curl` to dynamically pursue the redirect into the secure HTTPS channel:
   ```bash
   curl -L http://eicar.org -o /tmp/eicar_final.txt
   ```
4. **Data Verification:** The client successfully picked up the initial 266 bytes from the redirect page, followed the target location, and successfully pulled down the full **362.7 KB** payload over a 9-second tracking window.

![Figure 7](images/7_remediated_curl_transfer.png)
*Figure 7: Terminal 2 tracking parameters validating multi-stage payload downloads over the corrected channel.*

5. Returned to Terminal 1, stopped the tracking stream with `Ctrl + C`, and verified a significant surge in recorded metrics.

![Figure 8](images/8_tcpdump_packet_counts.png)
*Figure 8: Capture interface volumes logs confirming successful ingestion of 251 packets.*

---

### Phase 6: Final Validation & Forensic Stream Reconstruction
1. Loaded the finalized remediation pcap file into the graphical environment under a clean process:
   ```bash
   wireshark /tmp/malware_final.pcap &
   ```
2. **Isolating the Target Conversation**: Applied the display filter `tcp.stream eq 5` to isolate the exact session timeline of the download event, mapping out the full packet sequence lifecycle:
   * **Packets 30, 31, & 32 (The 3-Way Handshake)**: Exposes the entry checkpoint at Layer 4—showing your Kali host dynamically assigning Source Port **44532** to establish the TCP socket connection to the web server's Destination Port **443** via the standard sequence (`SYN` → `SYN, ACK` → `ACK`).
   * **Packet 33 (`Client Hello`)**: Captures the exact moment the Kali host initiated the cryptographic handshake. Auditing the **Server Name Indication (SNI)** metadata field explicitly identifies the target domain as `SNI=eicar.org`.
   * **Packet 36 (`Server Hello`)**: Marks the remote web host locking down the secure cryptographic parameters using highly secure **TLSv1.3** versions.
   * **Bulk Data Encapsulation**: Packets 40 through 172 switch protocol flags entirely to **TLSv1.3 / Application Data**, representing the fully encrypted payload transfer moving across the network interface card.
   * **Packets 173 & 175 (Connection Teardown)**: Captures the precise moment the file download concludes and the socket is gracefully closed via raw Layer 4 termination flags (`FIN, ACK`).

![Figure 9](images/9_tls_stream_lifecycle.png)

---

## Project Conclusion & Defensive Recommendations
---

### 🛡️ Step 2: Strategic Security Recommendations for the Lab

Adding formal defensive mitigation steps at the end of the lab turns your project from a basic network tutorial into an authoritative **Forensic Engineering Report** that proves your security maturity. Paste this block at the very bottom of your markdown repository:

```markdown
## Future Defensive Countermeasures & Engineering Recommendations

While this lab focused strictly on the network capture mechanics and identifying tool-specific behaviors, an enterprise network must employ tactical architectural defenses to handle anomalous outbound protocol switches. 

### 1. Mandatory HTTP-to-HTTPS Redirection (HSTS Enforcement)
*   **The Risk Identified:** The unencrypted cleartext `GET` request in Packet 22 exposed the destination intent (`Host: eicar.org`) and client configuration metadata (`User-Agent: curl/8.18.0`) across inline tracking points before a redirect was parsed.
*   **The Recommendation:** Implement **HTTP Strict Transport Security (HSTS)** headers across the internal web infrastructure. HSTS strictly forces modern web user agents to automatically convert insecure link requests to HTTPS locally on the client host *before* any frames ever traverse the network card interface, stopping cleartext protocol leaks at the origin.

### 2. Strategic SSL/TLS Decryption Proxy Architecture
*   **The Risk Identified:** As documented in Phase 6, the actual target payload execution shifted into a closed, secure **TLSv1.3 tunnel**. Because the communication parameters are completely encrypted, traditional inline Network Intrusion Detection Systems (NIDS) and next-generation firewalls (NGFW) cannot analyze the payload data over the wire for malicious signatures.
*   **The Recommendation:** Deploy a centralized **SSL/TLS Inspection (TLS Break-and-Inspect)** architecture at the corporate perimeter gateway proxy. By terminating the external encrypted session at the proxy firewall, inspecting the cleartext bytes for threats, and re-encrypting the tunnel back to the client device, security teams regain complete signature tracking without sacrificing edge protection.

### 3. DNS-Layer Content Filtering & Sinkholing
*   **The Risk Identified:** Packets 13 and 14 exposed the infrastructure performing unhindered upstream public domain queries to discover external network locations.
*   **The Recommendation:** Reroute local DNS routing infrastructure into automated corporate protective recursive DNS firewalls (e.g., Cisco Umbrella, Quad9, or customized local RPZ zones). If an endpoint initiates an tracking request for a documented threat repository or unverified external node, the DNS provider safely **sinkholes** the response by returning a dead loopback IP address (`0.0.0.0`), neutralizing the threat vector prior to TCP Handshake establishment.
