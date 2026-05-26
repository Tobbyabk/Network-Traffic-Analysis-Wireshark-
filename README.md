## Objective

This project demonstrates hands-on network traffic analysis using Wireshark in a controlled lab environment. The primary goal was to capture, filter, and interpret live and simulated network traffic across key protocols — DNS, DHCP, HTTP, and HTTPS — to identify normal baseline behaviour, detect anomalies, and recognise indicators of malicious activity. This exercise strengthens core SOC analyst skills by bridging the gap between raw packet data and actionable security insights.

---

## Skills Learned

- Deep understanding of network packet structure and how data flows across the OSI model
- Proficiency in applying Wireshark display filters to isolate specific protocols and traffic patterns
- Ability to reconstruct and analyse DNS queries and responses to detect suspicious lookups and potential exfiltration
- Ability to interpret DHCP handshakes and identify rogue DHCP server activity or IP conflicts
- Understanding of HTTP request/response cycles and the ability to extract credentials, file transfers, and session tokens from unencrypted traffic
- Ability to analyse HTTPS/TLS handshakes, identify certificate details, and detect anomalous encrypted traffic patterns
- Development of critical thinking and pattern recognition skills for identifying malicious traffic in a SOC context
- Proficiency in exporting packet captures (PCAPs) and documenting findings for escalation and reporting

---

## Tools Used

- **Wireshark** — primary tool for packet capture and protocol-level traffic analysis
- **Windows 10 VM** — endpoint generating DNS, DHCP, HTTP, and HTTPS traffic
- **Kali Linux VM** — used for simulating attack traffic and running network tools
- **nslookup / dig** — for generating and validating DNS queries during analysis
- **curl / browser** — for generating HTTP and HTTPS traffic for capture
- **ipconfig /release & /renew** — for triggering DHCP handshake captures
- **Microsoft Sentinel / Splunk** (optional integration) — for ingesting PCAP findings into SIEM for correlation

---

## Lab Setup

```
┌─────────────────────────────────────────────────┐
│                  Lab Environment                 │
│                                                 │
│  ┌──────────────┐        ┌──────────────────┐   │
│  │ Windows 10   │        │   Kali Linux     │   │
│  │ (Target VM)  │◄──────►│   (Attacker VM)  │   │
│  │ 192.168.1.10 │        │  192.168.1.20    │   │
│  └──────┬───────┘        └──────────────────┘   │
│         │                                       │
│  ┌──────▼───────┐        ┌──────────────────┐   │
│  │    Router/   │        │   DNS Server     │   │
│  │   Gateway    │◄──────►│   192.168.1.1    │   │
│  │ 192.168.1.1  │        └──────────────────┘   │
│  └──────────────┘                               │
│                                                 │
│         Wireshark capturing on eth0             │
└─────────────────────────────────────────────────┘
```

---

## Steps

---

### Step 1 — Setting Up Wireshark and Starting a Capture

Launch Wireshark and select the active network interface (typically `eth0` or `Ethernet`). Click the blue shark fin icon to begin capturing live traffic. At this stage, all network activity on the selected interface is being recorded in real time.

> **Ref 1:** *(Screenshot — Wireshark interface selection screen showing available network adapters with packet activity indicators)*

**Key actions:**
- Select the correct interface based on where traffic is expected
- Confirm packets are flowing by watching the packet count increment
- Leave capture running before triggering protocol-specific traffic in the steps below

---

### Step 2 — DNS Traffic Analysis

**What is DNS?**
DNS (Domain Name System) translates human-readable domain names (e.g. `google.com`) into IP addresses. Every website visit, email, and many malware callbacks start with a DNS query. DNS is a critical protocol for SOC analysts because it can reveal suspicious domain lookups, command-and-control (C2) beaconing, and data exfiltration attempts.

**Generating DNS traffic:**
Open Command Prompt on the Windows VM and run:
```
nslookup google.com
nslookup suspicious-domain.xyz
nslookup microsoft.com
```

**Wireshark filter to isolate DNS:**
```
dns
```

**What to look for:**
- **Query (Type A)** — client asking for an IP address for a domain
- **Response** — server returning the resolved IP
- **High query volume** to a single domain — potential DNS tunnelling or beaconing
- **NXDOMAIN responses** — domain does not exist — could indicate malware trying to reach a C2 server
- **Long subdomain strings** — potential DNS exfiltration (e.g. `verylongbase64string.maliciousdomain.com`)

> **Ref 2:** *(Screenshot — Wireshark DNS filter applied, showing query and response packets with domain names visible in the Info column)*

> **Ref 3:** *(Screenshot — DNS packet expanded in the packet details pane showing Query Name, Query Type A, and resolved IP address in the Answers section)*

**SOC relevance:** Unusual DNS queries — especially to newly registered domains, domains with random-looking names, or excessive NXDOMAIN responses — are a primary indicator of malware activity and should be escalated for further investigation.

---

### Step 3 — DHCP Traffic Analysis

**What is DHCP?**
DHCP (Dynamic Host Configuration Protocol) automatically assigns IP addresses to devices on a network. A typical DHCP exchange follows a four-step process known as **DORA**: Discover → Offer → Request → Acknowledge.

**Generating DHCP traffic:**
On the Windows VM, open Command Prompt as Administrator and run:
```
ipconfig /release
ipconfig /renew
```

**Wireshark filter to isolate DHCP:**
```
dhcp
```
or for older Wireshark versions:
```
bootp
```

**What to look for:**
- **DHCP Discover** — client broadcasting to find a DHCP server (source: `0.0.0.0`, destination: `255.255.255.255`)
- **DHCP Offer** — server responding with an available IP address
- **DHCP Request** — client formally requesting the offered IP
- **DHCP Acknowledge** — server confirming the IP lease
- **Multiple DHCP Offers** from different sources — potential **rogue DHCP server** attack
- **DHCP starvation** — large number of Discover packets from the same MAC — attacker attempting to exhaust the IP address pool

> **Ref 4:** *(Screenshot — Wireshark DHCP filter showing the four-packet DORA sequence — Discover, Offer, Request, Acknowledge — with source and destination IP addresses visible)*

> **Ref 5:** *(Screenshot — DHCP Acknowledge packet expanded showing assigned IP address, subnet mask, default gateway, lease time, and DNS server in the packet details pane)*

**SOC relevance:** A rogue DHCP server on the network can redirect traffic through an attacker-controlled gateway, enabling man-in-the-middle attacks. Multiple Discover packets from a single MAC address in rapid succession indicate a DHCP starvation attempt.

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
