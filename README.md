## Objective

This project demonstrates hands-on network traffic analysis using Wireshark in a controlled lab environment. The primary goal was to filter and interpret captured traffic across key protocols — DNS and HTTP — to identify normal baseline behaviour, detect anomalies, and recognise indicators of malicious activity. This exercise strengthens core SOC analyst skills by bridging the gap between raw packet data and actionable security insights.

---

## Skills Learned

- Deep understanding of network packet structure and how data flows across the OSI model
- Proficiency in applying Wireshark display filters to isolate specific protocols and traffic patterns
- Ability to reconstruct and analyse DNS queries and responses to detect suspicious lookups and potential exfiltration
- Understanding of HTTP user-agent string field, by differenting non-standard/anomalous user-agent from normal ones 
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

### HTTP Traffic Analysis

HTTP (HyperText Transfer Protocol) is an unencrypted protocol used for web communication. Because HTTP traffic is transmitted in plaintext, Wireshark can display the full contents of requests and responses — including credentials, form data, cookies, and file transfers.
For http traffic analysis, another pcap file(user-agent.pcap) was loaded on wireshark as show in Image 6;

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 14 56 14" src="https://github.com/user-attachments/assets/67e264c8-a147-4793-82d5-092635d3de53" />

> **Image 6:** *(Screenshot — Wireshark window showing loaded user-agent.pcap file)*


**Wireshark filter to isolate HTTP:**
```
http
```

After applying http as filter, the number of displayed packet reduced to 48 as depicted in the Image 7 below;

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 15 00 25" src="https://github.com/user-attachments/assets/7f3b9008-e72c-485d-9e40-79cb741a587f" />

> **Image 7:** *(Screenshot — Wireshark window showing applied http filter)*

**What to look for:**
- **Analomous User-agents** — malicious user-agent masking as a normal/harmless user-agent

In this investigation, the "user-agent" field is the focal point used for spotting anomaly in HTTP traffic. Attackers usally craft some user-agents to make it look natural and bypass security. After laoding the pcap file in wireshark, "http.user_agent" filter was applied to discover anomalous user-agent as shown in Image 8.

<img width="1286" height="751" alt="Screenshot 2026-06-02 at 16 07 06" src="https://github.com/user-attachments/assets/0848b135-b81f-4ba2-8288-4f28ccde9bbf" />

> **Image 8:** *(Screenshot — Wireshark window showing applied http.user_agent filter)*

Then, I browsed through the user_agent column to discover any suspicious looking user-agent strings. While browsing through the user-agent column, I stumble on 3 different user-agent strings that caught my attention- they are;
- **Mozilla/5.0 (Windows; U; Windows NT 6.4; en-US) AppleWebKit/534.10 (KHTML, like Gecko) Chrome/8.0.552.237 Safari/534.10**

At a glance, it might look like a standard browser identifier. However, it contains a glaring logical error: it pairs a browser version from 2010 (Chrome/8.0) with a Windows OS preview from 2014 (Windows NT 6.4). A real user's browser would never organically produce this combination.
- **Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)**

The second weird user-agent string is the default User-Agent used by the Nmap Scripting Engine (NSE). Upon research, this indicates a network scanning was being conducted against the web server.
- **Wfuzz/2.4**

The last user-agent string discovered is the default User-Agent signature for Wfuzz, a popular web application vulnerability scanner and directory bruteforcing tool. This indicates an attacker performed automated-testing the web server to discover hidden files, directories, or login pages.
  

**SOC relevance:** Non-standard and custom user agent, audit tools (Nmap and Wfuzz) in the user-agent field info should be immediately flagged and escalated for further investigation

---


---

**Summary findings table:**

| Protocol | Observation | Severity | Action |
|----------|-------------|----------|--------|
| DNS | High-volume long subdomains address to single domain | High | Escalate — potential DNS tunnelling |
| HTTP | Non-standard user-agent strings | High | Escalate — data exposure risk and scanning detected |

---

## Key Takeaways

- DNS is the most valuable protocol for early threat detection — monitoring query patterns reveals C2 beaconing, tunnelling, and exfiltration before other indicators appear
- HTTP remains a significant risk in internal environments — Custom/anomalous user-agent strings could indicate network scanning 


---

## References

- [Wireshark Official Documentation](https://www.wireshark.org/docs/)
- [MITRE ATT&CK — T1071 Application Layer Protocol](https://attack.mitre.org/techniques/T1071/)
- [MITRE ATT&CK — T1568 DNS for C2](https://attack.mitre.org/techniques/T1568/)
- [RFC 1035 — DNS Protocol Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [TryHackMe — Wireshark Labs](https://tryhackme.com/room/wireshark101)
