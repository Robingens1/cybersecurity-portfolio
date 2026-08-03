# Project 2: Network Traffic Analysis, Forensic Triage & Protocol Recovery

## Project Overview
This project shows how I used command-line packet capture (`tcpdump`) and a graphical network analyzer (`Wireshark`) to do a deep network traffic analysis and forensic troubleshooting exercise.

The goal was to capture a simulated malware download over the network, trace an unencrypted redirect, work out why part of the capture came up empty, and then successfully re-capture and analyze the full secure (TLS 1.3) session.

### Core Skills Practiced
* Packet capture and re-capture using `tcpdump`
* Basic process management (`killall`)
* Troubleshooting at the raw TCP layer (`ip.addr` and connection flags)
* Investigating tool-specific behavior (how `curl` handles redirects)
* Reading a TLS handshake and pulling out session details (SNI, TLS version)
* Comparing an incomplete capture against a full, successful one

---

## Lab Setup
* **Analysis Machine:** Kali Linux VM (Static IP: `192.168.xx.xxx`)
* **Interface:** `eth0`
* **Local DNS Gateway:** `192.168.64.2`
* **Target Web Server:** `89.238.73.97`

---

## Step-by-Step Walkthrough

### Phase 1: Setup
1. Checked the network interface to confirm everything was configured correctly:
   ```bash
   ip a
   ```
   *Confirmed the interface name `eth0` and static IP `192.168.xx.xxx/24`.*

<img width="819" height="415" alt="Kali_Linux_IP_Project1" src="https://github.com/user-attachments/assets/02e44fd4-1958-44c1-8ad9-4030513058b4" />

*Figure 1: Checking the active network interface and IP address.*

2. Updated and installed the packet analysis tools:
   ```bash
   sudo apt update && sudo apt install -y wireshark tshark
   ```
   *Installed successfully from the Kali repositories.*

<img width="827" height="455" alt="Kali_update_ _wireshark_installation_termina1" src="https://github.com/user-attachments/assets/0826b9b0-30c5-4bdb-8400-093acdd35da0" />

*Figure 2: Installing Wireshark and tshark.*

---

### Phase 2: First Capture Attempt
1. In **Terminal 1**, started capturing traffic on `eth0`:
   ```bash
   sudo tcpdump -i eth0 -w /tmp/malware_download.pcap
   ```
2. In **Terminal 2**, simulated a malware download using the EICAR test file (a safe, industry-standard test file used for exactly this purpose), over plain HTTP:
   ```bash
   curl http://eicar.org -o /tmp/eicar_test.txt
   ```
3. **First observation:** the download only pulled in **266 bytes** before stopping.

<img width="730" height="315" alt="kali_wireshark_simulation_ _download_terminal2_first_attempt" src="https://github.com/user-attachments/assets/4aa4d0ca-665f-4cbb-a0b9-d40698a5c757" />
<img width="553" height="274" alt="image" src="https://github.com/user-attachments/assets/90ed6245-5875-49f2-8a7e-d1b570482098" />


*Figure 3: curl transfer stopping early.*

4. Stopped the capture in Terminal 1 with `Ctrl + C` and checked the file:
   ```bash
   ls -lh /tmp/malware_download.pcap
   ```
   *179 packets (15 KB) were captured.*

---

### Phase 3: Filtering and Investigating
1. Opened the capture in Wireshark:
   ```bash
   wireshark /tmp/malware_download.pcap &
   ```
2. Looked through the unfiltered capture first, and saw normal background noise — DNS lookups to `192.168.64.2`, ARP broadcasts, and some unrelated scan traffic aimed at another device on the network (`192.168.64.129`).

<img width="991" height="582" alt="Wireshark_Fresh_Capture1" src="https://github.com/user-attachments/assets/9cf41e3c-0561-4dd9-ac36-bd92972d5f66" />
<img width="720" height="140" alt="image" src="https://github.com/user-attachments/assets/ed7e8a9b-4156-40ad-a0f5-a1a4ed8e2134" />


*Figure 4: Unfiltered view showing normal background traffic.*

3. Applied an `http` filter to isolate the actual web request:
   * **Packet #26:** My machine (`192.168.xx.xxx`) sending an unencrypted `GET / HTTP/1.1` request, using `curl/8.18.0` as the user agent.
   * **Packet #28:** The server (`89.238.73.97`) responding with `301 Moved Permanently`.

<img width="992" height="578" alt="Wireshark_http_filter_applied" src="https://github.com/user-attachments/assets/99acc40d-6bbe-4b6f-abb3-4722a86e436a" />

*Figure 5: HTTP request (packet 26) and redirect response (packet 28).*

4. Followed the full conversation using `tcp.stream eq 5`. This showed the server's response header pointing to `Location: https://eicar.org`, along with the 266-byte payload.

<img width="994" height="534" alt="Wireshark_TCP_Stream-Window" src="https://github.com/user-attachments/assets/116b2150-d2f7-475a-85f7-d70e191e6ade" />

*Figure 6: TCP stream showing the redirect instruction.*

---

### Phase 4: The Missing TLS Traffic
When I tried filtering for `tls` or `ssl` traffic to see the secure connection, both filters returned nothing.

1. To figure out why, I filtered by IP address instead, to see all traffic to and from the server:
   ```text
   ip.addr == 89.238.73.97
   ```
2. **What I found:** Packets 32 and 33 showed the connection closing (`FIN, ACK`) right after the server's `301 Moved Permanently` response — no secure connection was ever attempted.
3. **Root cause:** By default, `curl` does not automatically follow redirects. So when the server said "go to HTTPS," curl just stopped. No TLS handshake or file transfer ever happened in this capture, because the client never continued the request.

---

### Phase 5: Fixing It and Capturing the Full Download
1. Closed the previous Wireshark session:
   ```bash
   killall -9 wireshark
   ```
2. Started a new capture in **Terminal 1**:
   ```bash
   sudo tcpdump -i eth0 -w /tmp/malware_final.pcap
   ```
3. Re-ran the download in **Terminal 2**, this time adding the `-L` flag so curl would follow the redirect:
   ```bash
   curl -L http://eicar.org -o /tmp/eicar_final.txt
   ```
4. **Result:** curl picked up the initial 266 bytes, followed the redirect, and downloaded the full file — **362.7 KB** over about 9 seconds.

<img width="553" height="274" alt="image" src="https://github.com/user-attachments/assets/eadb231c-41c1-4a01-95ac-55e07e4bc4a7" />
<img width="906" height="290" alt="kali_wireshark_simulation_ _download_terminal2_second_attempt" src="https://github.com/user-attachments/assets/aa786630-0fba-434b-986a-b0674ed89ab0" />


*Figure 7: Full download completing successfully after following the redirect.*

5. Stopped the capture with `Ctrl + C` and confirmed a much larger capture this time.

<img width="553" height="274" alt="image" src="https://github.com/user-attachments/assets/a90920bf-b4f7-46bd-a37d-c8b87867f519" />

*Figure 8: 251 packets captured this time, compared to 179 before.*

---

### Phase 6: Analyzing the Full Secure Session
1. Opened the new capture:
   ```bash
   wireshark /tmp/malware_final.pcap &
   ```
2. Filtered with `tcp.stream eq 5` to isolate the download session:
   * **Packets 30, 31, 32 (TCP Handshake):** My machine (source port **44532**) connecting to the server on port **443** via `SYN` → `SYN, ACK` → `ACK`.
   * **Packet 33 (Client Hello):** The start of the TLS handshake. The **SNI (Server Name Indication)** field confirmed the domain being requested: `SNI=eicar.org`.
   * **Packet 36 (Server Hello):** The server confirming the connection would use **TLS 1.3**.
   * **Packets 40–172:** The actual file transfer, all encrypted as TLS 1.3 application data.
   * **Packets 173 & 175 (Teardown):** The connection closing cleanly with `FIN, ACK`.

<img width="990" height="588" alt="Wireshark_new_tls_packet33_detailed_inspection" src="https://github.com/user-attachments/assets/df40bd2d-d225-4363-a695-35e18495b660" />

<img width="992" height="617" alt="Wireshark_new_tls_filter_applied" src="https://github.com/user-attachments/assets/1134087f-4c11-4872-be91-df91f814414d" />


*Figure 9: Full TLS session lifecycle, from handshake to teardown.*

---

## Project Conclusion & Defensive Recommendations

While this lab focused on the mechanics of the capture and figuring out why the first attempt failed, here's what I'd recommend for a real network based on what this traffic showed:

### 1. Enforce HTTPS by Default (HSTS)
**The risk:** The first request went out as plain, readable HTTP — exposing the target domain and client details (`User-Agent: curl/8.18.0`) before any redirect happened.
**The fix:** Turn on **HTTP Strict Transport Security (HSTS)**. This makes clients automatically connect over HTTPS from the start, so that unencrypted first request never happens.

### 2. TLS Inspection at the Network Edge
**The risk:** Once the connection moved to TLS 1.3, the traffic was fully encrypted — meaning standard intrusion detection systems and firewalls can't inspect it for threats.
**The fix:** Use a **TLS inspection (break-and-inspect) proxy** at the network perimeter. This lets the security team decrypt, inspect, and re-encrypt traffic without leaving the connection unprotected.

### 3. DNS-Layer Filtering
**The risk:** The lookup to the external domain went out without any restriction.
**The fix:** Route DNS through a filtering service (like Cisco Umbrella or Quad9) or internal blocklists. If a device tries to reach a known-bad domain, the request gets sinkholed (returned a dead address like `0.0.0.0`) before a connection is even attempted.
