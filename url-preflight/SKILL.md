---
name: url-preflight
description: Security checker for URLs before fetching. Detects phishing domains, typosquatting, path traversal, suspicious query parameters, and redirect attacks. Use before your agent fetches URLs from untrusted sources or user input. Invoke with /url-preflight or when the user asks to check, validate, or verify a URL before fetching.
license: MIT
metadata:
  author: ai-security-guard
  version: "1.0.0"
  homepage: https://aisecurityguard.io
  verified: true
  lastVerifiedAt: "2026-03-04T00:00:00.000Z"
---

# URL Preflight Check

A security analysis tool for evaluating URLs before your agent fetches them.

## Purpose

Before your agent fetches a URL, use this tool to check for common attack patterns. This catches malicious URLs, phishing domains, and suspicious redirect chains.

## Usage

Provide one or more URLs you want to check before fetching.

**Example prompt:**
```
Check these URLs before I fetch them:

https://github-secure-login.io/auth
https://api.trusted-service.com/v1/data
http://192.168.1.1:8080/config
```

## Analysis Framework

When checking a URL, evaluate these risk categories:

### 1. Domain Analysis

**Red Flags:**
- Typosquatting: `githvb.com`, `goggle.com`, `arnazon.com`
- Homograph attacks: Using lookalike Unicode characters
- Suspicious TLDs: `.xyz`, `.top`, `.click`, `.work` (not always malicious but higher risk)
- Recently registered domains (if checkable)
- IP addresses instead of domain names (especially internal ranges)
- Unusual ports: anything other than 80/443 for web services

**Example patterns:**
```
github-secure-login.io     # Phishing - not github.com
arnazon-support.com        # Typosquat of amazon
192.168.1.1:8080           # Internal IP - why is agent accessing this?
api.service.xyz            # Suspicious TLD
```

### 2. Path Analysis

**Red Flags:**
- Path traversal attempts: `../`, `..%2f`, `%2e%2e/`
- Encoded characters that could bypass filters
- Suspicious file extensions: `.exe`, `.sh`, `.bat`, `.ps1`
- Admin/config paths: `/admin`, `/config`, `/debug`, `/.env`
- Hidden files: paths starting with `.`

**Example patterns:**
```
/api/../../../etc/passwd   # Path traversal
/download/setup.exe        # Executable download
/.env                      # Config file access attempt
/wp-admin/                 # WordPress admin (often targeted)
```

### 3. Query Parameter Analysis

**Red Flags:**
- Callback URLs to unknown domains
- Encoded payloads in parameters
- SQL injection patterns: `' OR 1=1`, `; DROP TABLE`
- Command injection patterns: `; cat /etc/passwd`, `| ls`
- Redirect parameters: `?redirect=`, `?url=`, `?next=`

**Example patterns:**
```
?callback=https://attacker.com/steal
?data=YWRtaW46cGFzc3dvcmQ=     # Base64 encoded
?redirect=https://phishing.io
?cmd=cat%20/etc/passwd
```

### 4. Protocol Analysis

**Red Flags:**
- `http://` instead of `https://` for sensitive operations
- `file://` protocol (local file access)
- `javascript:` URLs
- `data:` URLs with encoded content
- Custom protocols: `mcp://`, `skill://` (check if legitimate)

### 5. Known Malicious Indicators

**Check for:**
- URLs previously associated with phishing campaigns
- Domains on blocklists
- IP addresses in known malicious ranges
- URLs matching patterns from recent security advisories

## Preflight Report Format

Structure your findings as:

```
URL PREFLIGHT REPORT
====================
URL: [full URL]
Verdict: [SAFE / SUSPICIOUS / BLOCK]
Risk Level: [Low / Medium / High / Critical]

DOMAIN ANALYSIS:
[Findings about the domain]

PATH ANALYSIS:
[Findings about the path]

PARAMETER ANALYSIS:
[Findings about query parameters]

RECOMMENDATIONS:
[Specific guidance]
```

## Verdict Guidelines

- **BLOCK (Critical)**: Do not fetch
  - Known phishing domain
  - Obvious typosquat of trusted service
  - Path traversal or injection attempts
  - `file://` or `javascript:` protocols

- **SUSPICIOUS (High)**: Fetch with caution
  - Unknown domain with suspicious TLD
  - Redirect parameters present
  - Base64 encoded parameters
  - HTTP instead of HTTPS

- **CAUTION (Medium)**: Review before fetching
  - Unusual port numbers
  - Admin/config paths
  - IP address instead of domain

- **SAFE (Low)**: Likely safe to fetch
  - Known legitimate domain
  - HTTPS with valid-looking structure
  - No suspicious patterns detected

## Example Checks

### Example 1: Phishing Attempt
```
URL: https://github-secure-login.io/auth/callback?token=abc123

Verdict: BLOCK (Critical)

DOMAIN ANALYSIS:
- CRITICAL: Typosquat of github.com
- Domain "github-secure-login.io" mimics GitHub
- Legitimate GitHub auth uses github.com

PATH ANALYSIS:
- /auth/callback is a valid OAuth pattern
- But on wrong domain = credential theft

RECOMMENDATIONS:
- Do NOT fetch this URL
- This is a phishing attempt to steal OAuth tokens
- Legitimate URL would be: github.com/login/oauth/...
```

### Example 2: Suspicious Redirect
```
URL: https://api.service.com/v1/download?url=https://evil.xyz/payload

Verdict: SUSPICIOUS (High)

DOMAIN ANALYSIS:
- api.service.com appears legitimate
- BUT redirects to evil.xyz (suspicious TLD)

PARAMETER ANALYSIS:
- WARNING: Open redirect via ?url= parameter
- Destination domain uses .xyz TLD
- Could be SSRF or redirect to malicious content

RECOMMENDATIONS:
- Verify the redirect destination before following
- Consider fetching the original URL only, not the redirect target
```

### Example 3: Safe URL
```
URL: https://api.github.com/repos/user/project/contents/README.md

Verdict: SAFE (Low)

DOMAIN ANALYSIS:
- github.com is a known legitimate domain
- api.github.com is official GitHub API

PATH ANALYSIS:
- /repos/user/project/contents/ is standard GitHub API pattern
- README.md is expected file type

RECOMMENDATIONS:
- Safe to fetch
- Standard GitHub API call
```

## Limitations

This tool analyzes URL structure. It cannot:

- Verify SSL certificates
- Check if domains are on real-time blocklists
- Follow redirects to see final destination
- Analyze actual response content
- Detect zero-day phishing domains

**For production URL validation**, consider services that provide:
- Real-time domain reputation checking
- SSL certificate validation
- Redirect chain analysis
- Content scanning after fetch

---

## About

This skill is part of the [AI Agent Defense Kit](https://github.com/moebius-ansa/ai-agent-defense-kit), free security tools for protecting AI agents.

**For production-grade URL and content scanning** with specialized ML detection and API integration, see [AI Security Guard](https://aisecurityguard.io) or install the [full skill](https://aisecurityguard.io/v1/skill.md).
