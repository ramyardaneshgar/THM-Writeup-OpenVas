# THM-Writeup-OpenVas
Writeup for TryHackMe OpenVas Lab - Enterprise-grade vulnerability scanning, reporting, and continuous monitoring using Docker, OpenVAS/GVM, GMP, NVTs, and TLS.

By Ramyar Daneshgar

##  Objective  
The goal of this lab was to understand OpenVAS, part of the Greenbone Vulnerability Management (GVM) framework, and its utility in enterprise-grade vulnerability management pipelines. This walkthrough reflects how I explored its architecture, deployed it securely via Docker, executed scans, and interpreted vulnerability findings using real-world CVEs like MS17-010.

---

##  Task 1: Introduction — Setting the Context  
I began by familiarizing myself with the OpenVAS component and its integration into the larger **GVM** suite. It's essential to note that OpenVAS is not a standalone tool—it’s part of a broader **threat and vulnerability management architecture**. It ingests NVTs (Network Vulnerability Tests) from Greenbone’s feed, which is continuously updated.

###  Key Technical Concept  
- **NVTs:** Written in NASL (Nessus Attack Scripting Language), these are scripts that OpenVAS uses to check for known vulnerabilities.
- **Feed Updates:** Ensuring the feed is up-to-date is critical for identifying the latest threats (e.g., zero-day vulnerabilities, CVEs).

---

##  Task 2: GVM Framework Architecture  
I mapped out the **three-tier architecture**:
1. **Frontend** (GSA): Web interface I would interact with.
2. **Backend** (OpenVAS, OSPD): Where the scanning and management logic lives.
3. **Feeds**: Vulnerability databases (NVTs, SCAP, CERT, etc.).

###  Technical Insight  
Understanding this architecture was crucial because it explained why scanning might seem delayed—especially if the GSA or OSPD components are not responding or misconfigured. Knowing how components communicate over Unix sockets and TLS-enabled REST APIs helped me troubleshoot efficiently later on.

---

##  Task 3: Deploying OpenVAS via Docker  
Instead of compiling from source or installing via Kali’s repositories, I went with Docker:

```bash
apt install docker.io
docker run -d -p 443:443 --name openvas mikesplain/openvas
```

This deployment method **sandboxed OpenVAS** in a containerized environment, reducing dependency conflicts and environmental drift.


Docker’s bridge network is not secure by default. I ensured only localhost (`127.0.0.1`) had access to the dashboard, preventing exposure of the admin interface over the internet.

---

##  Task 4: Initial Configuration Wizard  
Once OpenVAS initialized, I logged in (`admin:admin`) at `https://127.0.0.1`.

From there:
- Navigated to **Scans > Tasks**
- Clicked the **Magic Wand** (basic wizard)
- Scoped target `127.0.0.1`

This allowed me to verify that OpenVAS was running correctly and could complete an internal scan.

###  Logic Behind Localhost Test  
Using localhost reduced external variables (network latency, firewall issues) and ensured that the OpenVAS scanning engine, feed integration, and reporting systems were functional before moving to remote targets.

---

##  Task 5: Scanning Infrastructure  
Here, I focused on remote infrastructure. I scoped the **DVWA machine** by:
- Creating a **New Target**
- Setting **IP = 10.10.148.71** (as per the deployment)
- Then created a **New Task**:
  - Name: `DVWA_Scan`
  - Scanner: Default
  - Scan Config: Full and fast
  - Target: Linked from above

Launched scan from the dashboard.

###  Technical Considerations  
- **Port scanning** via Nmap is embedded in OpenVAS’s default config, so I didn’t need to run it externally.
- **Credentialed vs. Non-Credentialed Scans**: This scan was unauthenticated. A real-world enterprise pipeline would also include authenticated scans to detect privilege-specific misconfigurations.

---

##  Task 6: Reporting & Monitoring  
After scan completion, I analyzed the report:

###  Breakdown:
- Host: `10.10.148.71`
- Ports Detected:
  - 445/tcp (SMB)
  - 135/tcp (RPC)
  - 3389/tcp (RDP)
  - general/tcp (TCP stack)
- Total Vulns: 5
- **High Severity:** MS17-010 (EternalBlue)

###  Continuous Monitoring
I configured:
- **Schedule**: `Daily at 12AM UTC`
- **Alert**: Email triggered when High CVSS vulnerabilities are detected

This automation mirrors what I would implement in a CI/CD pipeline for nightly scans or as part of a SOC alerting framework.

---

##  Task 7: Practical Vulnerability Analysis (Case 001)

This report simulated a SOC incident where OpenVAS had flagged issues on a vulnerable server. I reviewed the scan and identified the following:

---

###  Vulnerability 1: MS17-010 - EternalBlue (CVE-2017-0143 to 0148)

**Port:** 445/tcp  
**Severity:** High (CVSS 9.3)  
**Detection Logic:**  
OpenVAS sends a **crafted SMB transaction with FID = 0**. If the server responds abnormally, it confirms the presence of the SMBv1 flaw.

**Fix:** Patch the server using Microsoft’s KB4013389 or disable SMBv1 protocol entirely.

---

###  Vulnerability 2: DCE/RPC Enumeration (Port 135)

**Threat:** Medium  
**Description:** Enumerates all available DCOM/MSRPC services over TCP.  
**Risk:** Enables attackers to map out services for lateral movement or privilege escalation.

**Fix:** Restrict access via host-based firewall rules.

---

###  Vulnerability 3: Weak TLS Ciphers on RDP (3389)

**Threat:** Medium  
**Issue:** Accepts insecure ciphers like RC4.  
**Fix:** Reconfigure the RDP TLS policy to disallow RC4, MD5, and other weak suites.

---

###  Vulnerability 4: SHA-1 Cert (RDP)

**Threat:** Medium  
**Issue:** Uses SHA-1 signed certificate  
**Fix:** Replace with a SHA-2 (256+) certificate chain.

---

###  Vulnerability 5: TCP Timestamping

**Threat:** Low  
**Impact:** Leaks system uptime; can aid in fingerprinting or uptime-based attacks (e.g., reboot-sensitive exploits).  
**Fix:** Disable via `sysctl` or `netsh` on Windows.

---

##  Lessons Learned

###  Technical Takeaways  
1. **Docker Deployment Simplifies Testing**  
   Running OpenVAS in a Docker container removed setup friction and let me focus on scanning logic, not installation errors.

2. **Architecture Knowledge = Debugging Superpower**  
   Knowing how GVM components interact helped me debug scanning delays and web interface responsiveness.

3. **Real-World Vulnerabilities Were Not Just Theoretical**  
   The MS17-010 finding mirrored what actual ransomware campaigns (WannaCry, NotPetya) exploited. Detecting it felt tangible, not academic.

4. **Feed Updates Are Critical**  
   If feeds aren't synced, new vulnerabilities go undetected. I automated feed sync with cron for long-term ops.

5. **Reports Are Boardroom-Ready**  
   OpenVAS reports include CVSS scores, detection methods, and mitigation steps—valuable for both engineers and executives.

6. **Alerting & Scheduling = Enterprise-Ready**  
   Integration of these features makes OpenVAS suitable for 24/7 vulnerability lifecycle management, bridging the gap between reactive and proactive security postures.

