# 🔐 VDP Security Writeup
## Vertical Privilege Escalation via Permission Wildcard Injection

> **🔴 Critical Severity** · Broken Access Control · Privilege Escalation · Client-Side Authorization Bypass

---

## 📌 Finding at a Glance

| Field | Details |
|---|---|
| **Program** | Vulnerability Disclosure Program (VDP) |
| **Vulnerability** | Broken Access Control |
| **Attack Type** | Vertical Privilege Escalation |
| **Root Cause** | Client-side authorization trust |
| **Affected Data** | `permissions[]` scope array |
| **Severity** | 🔴 **Critical** |
| **Impact** | Standard user → Administrative privileges |

---

# 🎯 Executive Summary

During a security assessment under the target's **Vulnerability Disclosure Program (VDP)**, a critical **Broken Access Control** vulnerability was identified in the application's authorization model.

The application relied on authorization information contained in the authentication response to determine the capabilities available to the user.

By intercepting the login response for a low-privileged account and modifying the `permissions[]` array to contain a wildcard (`"*"`), the client treated the user as having unrestricted permissions.

This resulted in **vertical privilege escalation**, allowing a standard user to access administrative functionality without legitimate administrative authorization.

### 🔥 Core Security Issue

> **Authorization decisions must never depend on untrusted client-side state.**

---

# 🧩 Vulnerability Details

| Category | Finding |
|---|---|
| **Vulnerability Type** | Broken Access Control / Privilege Escalation |
| **Affected Mechanism** | Client-side permission authorization |
| **Manipulated Object** | `permissions[]` |
| **Attack Vector** | HTTP response manipulation |
| **Required Account** | Low-privileged standard user |
| **Result** | Administrative access |

### Why This Is Critical

The application's trust boundary was placed on the **client** rather than the backend.

The client received permission information and used it to determine which privileged functionality should be available.

An attacker who can modify that response can therefore influence the authorization state presented to the application.

---

# 🧪 Proof of Concept

## Step 1 — Baseline Authentication

A standard, low-privileged user account was used to authenticate to the application.

The authentication response was intercepted using **Burp Suite Proxy**.

### Original Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 342

{
  "status": "success",
  "data": {
    "user_id": 10482,
    "username": "standard_user",
    "email": "user@example.com",
    "role": "user",
    "permissions": [
      "profile:read",
      "profile:write",
      "media:view"
    ]
  }
}
```

### 🔎 Observation

The response contained the user's permission scopes:

```text
profile:read
profile:write
media:view
```

These values were subsequently processed by the client application.

---

# ⚠️ Step 2 — Permission Wildcard Injection

The intercepted JSON response was modified by replacing the restricted permission list with a wildcard value.

### Modified Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 295

{
  "status": "success",
  "data": {
    "user_id": 10482,
    "username": "standard_user",
    "email": "user@example.com",
    "role": "user",
    "permissions": [
      "*"
    ]
  }
}
```

### 🔬 Key Difference

```diff
 "permissions": [
-  "profile:read",
-  "profile:write",
-  "media:view"
+  "*"
 ]
```

The important security flaw is that the application accepted the manipulated client-side authorization state.

---

# 🚨 Step 3 — Privilege Escalation Verification

After forwarding the modified response to the browser:

1. Administrative navigation elements became available.
2. Previously restricted functionality was unlocked.
3. Protected administrative endpoints became accessible.
4. Administrative access was successfully confirmed.

### Observed Administrative Paths

```text
/admin/dashboard
/admin/users
```

This demonstrated that a standard account could transition into an administrative authorization state through response manipulation.

---

# 🔥 Attack Flow

```text
┌──────────────────────┐
│ Standard User Login  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Authentication       │
│ Response Returned    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Burp Suite           │
│ Response Intercept   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ permissions[]        │
│ changed to "*"       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Client Accepts       │
│ Manipulated State    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Administrative UI    │
│ & Functionality      │
└──────────────────────┘
```

---

# 💥 Impact Analysis

## 🔴 1. Complete Authorization Bypass

A standard user can bypass the application's intended permission model and obtain high-privileged functionality.

## 🔴 2. Vertical Privilege Escalation

The vulnerability allows movement from:

```text
Standard User
     ↓
Administrative User
```

without legitimate server-side authorization.

## 🔴 3. Administrative Functionality Exposure

Based on the assessment, privileged functionality included administrative dashboards, user-management functionality, and restricted data.

## 🔴 4. Broken Trust Boundary

The fundamental security issue is the application's reliance on information controlled by the user agent.

```text
❌ Client decides permissions

Browser
   ↓
"permissions": ["*"]
   ↓
Application trusts client state
   ↓
Admin functionality
```

A secure design should instead be:

```text
Browser
   ↓
Authenticated Request
   ↓
Backend Authorization
   ↓
Server-side Role / Permission Check
   ↓
Allow / Deny
```

---

# 🛠️ Remediation

## 1. 🔒 Enforce Authorization Server-Side

The backend must independently validate authorization for **every protected API request**.

Do not trust:

- HTTP response modifications
- LocalStorage
- SessionStorage
- Client-side JavaScript state
- Unverified JSON permission objects

The server should determine the user's permissions from trusted session state or securely validated authentication claims.

---

## 2. 👤 Implement Robust RBAC

Use a server-side **Role-Based Access Control (RBAC)** model.

For example:

```text
User
 │
 ├── Role: user
 │      ├── profile:read
 │      ├── profile:write
 │      └── media:view
 │
 └── Role: admin
        ├── users:manage
        ├── settings:manage
        └── admin:access
```

The backend should resolve these permissions from trusted authorization data rather than accepting them from the browser.

---

## 3. 🚫 Reject Unauthorized Wildcards

Wildcard permission expansion should never be granted simply because the client submits:

```json
{
  "permissions": ["*"]
}
```

If wildcard scopes are supported internally, they should only be assigned through trusted server-side role definitions and validated authorization logic.

---

## 4. 🧪 Test Every Privileged Endpoint

Authorization should be tested independently at the API layer.

For every administrative endpoint, verify:

```text
Valid Admin  →  ✅ Allowed
Standard User →  ❌ Denied
Modified Client State → ❌ Denied
Expired Session → ❌ Denied
Invalid Token → ❌ Denied
```

---

# 📊 Risk Summary

| Security Control | Result |
|---|---|
| Server-side authorization | ❌ Insufficient |
| Client-side permission trust | 🔴 Vulnerable |
| Vertical privilege escalation | 🔴 Confirmed |
| Admin functionality exposure | 🔴 Confirmed |
| Wildcard permission handling | 🔴 Unsafe |
| Overall Severity | 🔴 **Critical** |

---

# 🧠 Root Cause

The root cause is a **broken authorization trust model**.

The application treated permission information delivered to the client as authoritative instead of deriving and enforcing authorization decisions on the server.

### Insecure Model

```text
Server → permissions → Browser
                         ↓
                    Client decides
                         ↓
                     Access granted
```

### Secure Model

```text
Browser → Request → Server
                     ↓
              Authenticate User
                     ↓
              Resolve Role/ACL
                     ↓
             Authorize Request
                     ↓
               Allow / Deny
```

---

# 🏁 Conclusion

This finding demonstrates a critical **vertical privilege escalation** caused by trusting client-controlled authorization state.

A low-privileged user was able to manipulate the `permissions[]` value in the authentication response and introduce a wildcard permission. The application subsequently exposed administrative functionality.

The primary remediation is straightforward:

> **Treat the client as untrusted and enforce authorization decisions on the server for every privileged operation.**

Client-side permission information may be useful for controlling UI presentation, but it must **never be the security boundary**.

---

## 🛡️ Security Takeaway

> **Hiding an admin button is not authorization.**

The real security control must exist at the backend API and resource level, where every request is independently authenticated and authorized.

---

### ⚠️ Responsible Disclosure

This writeup is intended for authorized vulnerability research and responsible disclosure. Any sensitive identifiers, endpoints, account information, or target-specific details should be redacted before public publication.
