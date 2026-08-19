# Capstone Project: Demonstrating FTP vs. SFTP Security

## Executive Summary
This laboratory project provides a practical demonstration contrasting the security baselines of the unencrypted File Transfer Protocol (FTP) against the secure, cryptographic Secure File Transfer Protocol (SFTP). Utilizing a virtualized environment, network traffic was actively intercepted using a packet analyzer during standard authentication and file upload operations. The objective of this capstone is to prove the vulnerability of plaintext transmissions to credential harvesting and data payload extraction, verifying the critical necessity of transport-layer encryption in enterprise networks.

## Environment Details
*   **Operating System:** Kali Linux (Rolling Edition)
*   **Packet Analyzer:** Wireshark v4.x
*   **Insecure Daemon:** vsftpd v3.0.5
*   **Secure Daemon:** OpenSSH Server (SFTP subsystem)
*   **Network Interface:** Local Loopback (`127.0.0.1` / `lo`)
*   **Test Identity:** `labuser`
*   **Target Payload:** `secret.txt` ("CONFIDENTIAL: This is a top-secret project file...")

---

## Phase 1: Environment Setup & Configurations

To establish an active laboratory environment, the target transfer protocols were installed, isolated, and verified for baseline operability.

### 1.1 Insecure FTP Server Configuration
The File Transfer Protocol daemon (`vsftpd`) was modified to permit local user logging and administrative data-writing privileges.

```text
![Uploading Verification of unencrypted vsftpd configuration file parameters (local_enable and write_enable enabled).jpg…]()

Figure 1.1: Verification of unencrypted vsftpd configuration file parameters (local_enable and write_enable enabled).
```

### 1.2 Target Identity & Sensitive Payload Instantiation
A dedicated non-root account was created to validate standard authentication parameters alongside a dummy high-value payload file.

```text
[Insert your user creation screenshot here]
Figure 1.2: Creation of the target test user account (labuser) on the local host.
```

```text
[Insert your file creation screenshot here]
Figure 1.3: Creation of the sensitive plaintext payload file (secret.txt) prior to network transmission.
```

---

## Phase 2: Insecure FTP Traffic Capture & Extraction

An active network capture session was bound to the loopback interface (`lo`) prior to conducting an interactive plaintext transaction. 

### 2.1 Interactive FTP Session
The client connected unencrypted via TCP Port 21, authenticating and pushing the sensitive file to the server.

```text
[Insert your image_RBRZj3.png screenshot here]
Figure 2.0: Interactive terminal session showing successful vsftpd authentication and unencrypted file transmission (secret.txt) via the local loopback interface.
```

### 2.2 Control Channel Network Sniffing
Applying an active display filter for the `ftp` protocol immediately isolated the control channel, mapping the unencrypted request sequences explicitly.

```text
[Insert your image_0G84Yl.png screenshot here]
Figure 2.1: Wireshark packet capture session logging unencrypted FTP control channel traffic over the loopback interface.
```

### 2.3 Plaintext Credential Harvesting
Reconstructing the underlying TCP stream compiled the fragmented parameters back into a readable sequence, displaying the raw credential pair:
*   **Username:** `labuser`
*   **Password:** `password123`

```text
[Insert your Wireshark FTP Follow TCP Stream popup screenshot here]
Figure 2.2: Plaintext credential extraction via Wireshark TCP Stream inspection of the unencrypted FTP control channel.
```

### 2.4 Payload Interception
Isolating the active data transfer channel (`ftp-data` filter) and mapping the reconstructed TCP stream extracted the complete contents of `secret.txt` seamlessly without any cryptographic resistance.

```text
[Insert your Wireshark ftp-data Follow TCP Stream popup screenshot here]
Figure 2.3: Successful extraction of transmitted plaintext file contents (secret.txt) via Wireshark ftp-data stream reconstruction.
```

---

## Phase 3: Secure SFTP Traffic Capture & Analysis

To document mitigation protocols, the secure sub-system was activated to transition transport-layer handling through an SSHv2 encrypted tunnel.

### 3.1 Host Authenticity Validation
The initial handshake triggered a host-key verification prompt to protect the transaction against local machine spoofing or Man-in-the-Middle (MitM) interference.

```text
[Insert your image_-5JwMh.png screenshot here]
Figure 3.0: SSH host key verification prompt during the initial secure SFTP connection initialization to the local loopback interface.
```

### 3.2 Secure Multi-Channel Transfer
The client authenticated securely, transferring `secret.txt` directly to the `/home/labuser/` directory over an encrypted sub-channel.

```text
[Insert your image_LkNSDX.png screenshot here]
Figure 3.1: Interactive terminal session showing successful SFTP authentication and secure, encrypted file upload (secret.txt) over the loopback interface.
```

### 3.3 Protocol Encapsulation Analysis
Filtering for the `ssh` protocol confirmed that all transactional, authentication, and directory details were completely encapsulated within opaque `SSHv2` packets over TCP Port 22.

```text
[Insert your image_n0c_Rl.png screenshot here]
Figure 3.2: Wireshark packet capture session displaying encrypted SSHv2 packets isolating the secure SFTP session from data sniffing.
```

### 3.4 Cryptographic Verification (Ciphertext Proof)
Attempting a complete TCP Stream reconstruction revealed high-entropy randomized ciphertext symbols. Neither the password credentials nor the file payload data could be read or extracted by the analyzer.

```text
[Insert your image_J3bTvI.png screenshot here]
Figure 3.3: Cryptographic protection validation showing unreadable ciphertext session data via Wireshark TCP Stream inspection of the SFTP session.
```

---

## Phase 4: Definitive Comparative Matrix

| Security Parameter | Standard FTP | Secure SFTP | Risk Implication |
| :--- | :--- | :--- | :--- |
| **Control Channel Port** | TCP 21 | TCP 22 (SSH Subsystem) | Ports are easily distinguished during reconnaissance scans. |
| **Data Channel Port** | TCP 20 (Passive random) | Multiplexed over TCP 22 | FTP requires multiple dynamic port allocations, complicating firewalls. |
| **Authentication Mode** | Plaintext ASCII | Asymmetric Cryptography | FTP passwords are swept easily from local segments. |
| **Data Confidentiality** | None (Plaintext) | Enforced Encryption (AES/ChaCha20) | FTP payload data is highly vulnerable to sniffing and extraction. |
| **Integrity Checks** | None | Enforced (HMAC-SHA256 / Poly1305) | FTP data can be altered mid-transit without triggering errors. |
| **Host Verification** | Unsupported | Supported (Fingerprints/Known Hosts) | FTP does not offer mechanisms to prevent connection spoofing. |

## Conclusion & Engineering Recommendations
The empirical findings collected during this laboratory project prove that **FTP poses a severe vulnerability to infrastructure confidentiality and integrity**. Any actor maintaining visibility over a localized network layer can capture control traffic and compromise operational assets instantly. 

### Recommendations:
1.  **Immediate Deprecation:** Terminate all legacy FTP services across production systems globally.
2.  **Enforce Cryptographic Alternatives:** Mandate SFTP or HTTPS for all batch processing, server automation, and administrative system file transfers.
3.  **Firewall Restrictions:** Implement structural perimeter rules blocking egress/ingress actions on TCP port 20 and 21, establishing active alert triggers for unauthorized legacy attempts.
