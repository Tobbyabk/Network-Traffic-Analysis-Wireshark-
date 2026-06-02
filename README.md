## Objective

This project demonstrates hands-on network traffic analysis using Wireshark in a controlled lab environment. The primary goal was to filter and interpret captured traffic across key protocols — DNS, HTTP, and HTTPS — to identify normal baseline behaviour, detect anomalies, and recognise indicators of malicious activity. This exercise strengthens core SOC analyst skills by bridging the gap between raw packet data and actionable security insights.

---

## Skills Learned

- Deep understanding of network packet structure and how data flows across the OSI model
- Proficiency in applying Wireshark display filters to isolate specific protocols and traffic patterns
- Ability to reconstruct and analyse DNS queries and responses to detect suspicious lookups and potential exfiltration
- Understanding of HTTP request/response cycles and the ability to extract credentials, file transfers, and session tokens from unencrypted traffic
- Ability to analyse HTTPS/TLS handshakes, identify certificate details, and detect anomalous encrypted traffic patterns
- Development of critical thinking and pattern recognition skills for identifying malicious traffic in a SOC context
- Proficiency in documenting findings for escalation and reporting

---

## Tools Used

- **Wireshark** — primary tool for packet and protocol-level traffic analysis
- **Ubuntu Linux VM** — used for running network tools
- **PCAP**— captured packet ingested into wireshark

---

## Lab Setup
This investigation was carried out in a virtual environment using TryHackMe platform. The VM was launched on on TryHackMe and it took about 2 minutes for it to spin up. The capture packet required for this investigation was already provided in the VM and there was no need for a live-traffic capture. The Ubuntu version of the VM and the PCAP file are depicted in [Image 1] on the left and right side of the screenshot respectively.

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 12 08 47" src="https://github.com/user-attachments/assets/3edbb014-525e-4f6d-bf39-3ca0c4011fd6" />

> **Image 1 :** *(Screenshot — Ubuntu Version and PCAP files)*



###  Setting Up Wireshark for packet analysis

Launch Wireshark and navigate to File Menu, Click on open and select the PCAP file[DNS.pcap]. At this stage, all captured traffic will open and the lab is ready for analysis

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 13 15 15" src="https://github.com/user-attachments/assets/ec7cc12b-33c7-4e6d-9a39-6a5b7d71441a" />


> **Image 2:** *(Screenshot — Wireshark window showing loaded dns.pcap file)*


---

###  DNS Traffic Analysis

DNS (Domain Name System) translates human-readable domain names (e.g. `google.com`) into IP addresses. Every website visit, email, and many malware callbacks start with a DNS query. DNS is a critical protocol for SOC analysts because it can reveal suspicious domain lookups, command-and-control (C2) beaconing, and data exfiltration attempts.

**Wireshark filter to isolate DNS:**
```
dns
```
After applying dns as filter, the number of displayed packet reduced to 30370 as depicted in the Image 3 below

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 13 21 32" src="https://github.com/user-attachments/assets/2c17e0a3-57dc-43ed-bf17-e0c2f2d2e98a" />

> **Image 3:** *(Screenshot — Wireshark window showing apllied dns filter)*
>



DNS attacks typically flare up right after malware hits a system or a vulnerability gets exploited. Once inside, an attacker sets up a domain name to act as their Command and Control (C2) server.The malware then starts sending DNS queries to this server which is known as "DNS Tunnelling". These requests look strange and they are much longer than normal DNS queries and target weird subdomains

**What to look for:**
- **High query volume** to a single domain — potential DNS tunnelling or beaconing
- **Long subdomain strings** — potential DNS exfiltration
Firstly, I applied (dns contains "dnscat") as my filter because "dnscat" is a tool used to create an encrypted command-and-control (C&C) channel over the DNS protocol. I wanted to confirm if some dns queries were encoded and I got some hits after appying the filter as shown in Image 4.

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 13 55 12" src="https://github.com/user-attachments/assets/55e072ad-e24d-4d8a-ae3f-1cbce1ceb634" />

> **Image 4:** *(Screenshot — DNS packet expanded in the packet details pane showing Query Name with dnscat)*
>

Then, I decided to use another filter (dns.qry.name.len > 200 and !mdns) to confirm it there was any long subdomain strings present as shown in Image 5. This filter will help isolate any query with a length over 200 and !mds will  Disable local link device queries in order to remove noise from the packets. After inpecting several packet details, a pattern of weird long subdomain strings was noticed and the main domain name of the attacker discovered was dataexfil[.]com.

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 14 09 11" src="https://github.com/user-attachments/assets/471d5249-7796-4faa-a7e6-5c0e530f621d" />

> **Image 5:** *(Screenshot — DNS packet expanded in the packet details pane showing Query Name with the main domain name)*
>



**SOC relevance:** Unusual DNS queries  —  Long DNS addresses with encoded subdomain addresses is a primary indicator of dns tunnelling and should be escalated for further investigation.

---

---

### Step 4 — HTTP Traffic Analysis

**What is HTTP?**
HTTP (HyperText Transfer Protocol) is an unencrypted protocol used for web communication. Because HTTP traffic is transmitted in plaintext, Wireshark can capture and display the full contents of requests and responses — including credentials, form data, cookies, and file transfers.

**Generating HTTP traffic:**
On the Windows VM, open a browser or Command Prompt and run:
```
curl http://testphp.vulnweb.com
curl http://neverssl.com
```
Or navigate to any `http://` website in a browser.

**Wireshark filter to isolate HTTP:**
```
http
```

**What to look for:**
- **GET requests** — client requesting a webpage or resource
- **POST requests** — client submitting data (login forms, file uploads) — POST bodies are visible in plaintext
- **HTTP 200 OK** — successful response
- **HTTP 401 Unauthorized / 403 Forbidden** — access denied — could indicate brute force or credential testing
- **HTTP 302 Redirect** — server redirecting client — can be used in phishing redirects
- **Credentials in plaintext** — visible in POST request body (e.g. `username=admin&password=password123`)

**Extracting HTTP content:**
In Wireshark, right-click a HTTP packet → Follow → HTTP Stream to see the full request and response in plain text.

> **Ref 6:** *(Screenshot — Wireshark HTTP filter applied showing GET and POST requests with destination IP, URL path, and HTTP method visible in the Info column)*

> **Ref 7:** *(Screenshot — HTTP stream follow showing a plaintext POST request body containing form data including username and password fields)*

**SOC relevance:** Any credentials, session tokens, or sensitive data transmitted over HTTP are visible to anyone on the network segment. HTTP is increasingly rare on legitimate sites — HTTP traffic to internal or unusual destinations should be investigated as a potential data exfiltration or C2 channel.

---

### Step 5 — HTTPS and TLS Traffic Analysis

**What is HTTPS/TLS?**
HTTPS (HTTP Secure) encrypts web traffic using TLS (Transport Layer Security). While the content of HTTPS traffic cannot be read directly, Wireshark can capture and analyse the TLS handshake — which reveals the encryption protocol version, cipher suites, and server certificate details — without decrypting the payload.

**Generating HTTPS traffic:**
On the Windows VM, open a browser and navigate to:
```
https://www.google.com
https://www.microsoft.com
```

**Wireshark filter to isolate TLS/HTTPS:**
```
tls
```
or to filter by port:
```
tcp.port == 443
```

**What to look for in the TLS Handshake:**
| Packet | Direction | Description |
|--------|-----------|-------------|
| Client Hello | Client → Server | Proposes TLS version and cipher suites |
| Server Hello | Server → Client | Selects TLS version and cipher suite |
| Certificate | Server → Client | Server presents its SSL certificate |
| Client Key Exchange | Client → Server | Establishes session key |
| Application Data | Both | Encrypted payload (not readable) |

**Analysing the TLS Certificate:**
Click the Certificate packet → Expand `Transport Layer Security` → `Handshake Protocol` → `Certificate` to see:
- **Subject** — who the certificate belongs to
- **Issuer** — the Certificate Authority that signed it
- **Validity period** — check for expired certificates
- **Subject Alternative Names (SANs)** — domains covered by the certificate

**Detecting suspicious TLS traffic:**
- **Self-signed certificates** — legitimate sites use trusted CA-signed certificates
- **Expired certificates** — potential misconfiguration or attacker using old infrastructure
- **Unusual cipher suites** — old or weak ciphers (e.g. TLS 1.0, RC4) indicate outdated or malicious servers
- **TLS on non-standard ports** — C2 traffic often uses HTTPS on ports other than 443 to avoid detection

> **Ref 8:** *(Screenshot — Wireshark TLS filter applied showing Client Hello, Server Hello, Certificate, and Application Data packets in the capture)*

> **Ref 9:** *(Screenshot — TLS Certificate packet expanded in the packet details pane showing Subject, Issuer, Validity dates, and Subject Alternative Names)*

> **Ref 10:** *(Screenshot — Wireshark showing TLS Application Data packets — demonstrating that payload is encrypted and not readable, unlike HTTP)*

**SOC relevance:** While HTTPS protects data in transit, encrypted traffic is increasingly used by malware to hide C2 communications. SOC analysts should flag TLS traffic with self-signed certificates, unusual cipher suites, or connections to newly registered domains — even if the payload cannot be read.

---

### Step 6 — Identifying Suspicious Traffic and Building a Summary Report

After capturing all protocol traffic, apply combined filters to identify anomalies:

**Filter for DNS + suspicious domains:**
```
dns && dns.qry.name contains "xyz"
```

**Filter for unencrypted HTTP POST (potential credential leak):**
```
http.request.method == "POST"
```

**Filter for TLS with self-signed certificates:**
```
tls.handshake.type == 11
```

**Filter for DHCP anomalies (multiple offers):**
```
dhcp.option.dhcp == 2
```

> **Ref 11:** *(Screenshot — Wireshark with combined filters applied, showing a filtered view of suspicious DNS queries and HTTP POST requests isolated from normal traffic)*

**Summary findings table:**

| Protocol | Observation | Severity | Action |
|----------|-------------|----------|--------|
| DNS | NXDOMAIN responses to random-looking domains | Medium | Investigate domain reputation |
| DNS | High-volume subdomains to single domain | High | Escalate — potential DNS tunnelling |
| DHCP | Multiple DHCP Offers from different MACs | High | Escalate — potential rogue DHCP |
| HTTP | Plaintext POST with credential data | High | Escalate — data exposure risk |
| HTTPS | Self-signed certificate on port 443 | Medium | Investigate server and destination |
| HTTPS | TLS 1.0 detected | Low | Flag for remediation |

> **Ref 12:** *(Screenshot — Completed Wireshark capture summary showing packet statistics — total packets, protocol breakdown, and capture duration — via Statistics → Protocol Hierarchy)*

---

## Key Takeaways

- DNS is the most valuable protocol for early threat detection — monitoring query patterns reveals C2 beaconing, tunnelling, and exfiltration before other indicators appear
- HTTP remains a significant risk in internal environments — plaintext credentials and data are easily captured by any analyst (or attacker) on the same network segment
- HTTPS traffic analysis focuses on the TLS handshake, not the payload — certificate anomalies and unusual cipher suites are the primary indicators of malicious encrypted traffic
- DHCP anomalies are often overlooked but can indicate serious man-in-the-middle attacks — rogue DHCP servers are a classic network pivot technique
- Every finding should be documented with packet reference numbers, timestamps, source/destination IPs, and a clear escalation recommendation

---

## References

- [Wireshark Official Documentation](https://www.wireshark.org/docs/)
- [MITRE ATT&CK — T1071 Application Layer Protocol](https://attack.mitre.org/techniques/T1071/)
- [MITRE ATT&CK — T1568 DNS for C2](https://attack.mitre.org/techniques/T1568/)
- [RFC 1035 — DNS Protocol Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [RFC 2131 — DHCP Protocol Specification](https://www.rfc-editor.org/rfc/rfc2131)
- [TryHackMe — Wireshark Labs](https://tryhackme.com/room/wireshark101)
