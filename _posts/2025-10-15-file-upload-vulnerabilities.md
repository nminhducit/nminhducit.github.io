---
layout: post
title: "File Upload Vulnerabilities — From Direct Shell Uploads to Polyglot Bypasses"
date: 2025-10-15
last_modified_at: 2025-10-15
category: "Web Security"
reading_time: 10
author: "Nguyễn Minh Đức"

# SEO Optimization

description: "Technical walkthrough of exploiting and understanding File Upload vulnerabilities across six security-hardening levels. Covers extension bypass, MIME spoofing, .htaccess handler abuse, and polyglot image techniques."

excerpt: "A detailed technical report analyzing File Upload vulnerabilities from Level 1 to Level 6, including bypass methods, lessons learned, and secure mitigation strategies."

# Social Media Images

image: /assets/images/posts/file-upload-vulns.jpg
og_image: /assets/images/posts/file-upload-vulns-og.jpg
twitter_image: /assets/images/posts/file-upload-vulns-twitter.jpg


# Tags for better categorization

tags:
  - Web Security
  - File Upload
  - PHP Security
  - Malware Upload
  - Penetration Testing
  - Reverse Engineering
  - RCE exploitation

# Keywords for search engines (comma-separated)
keywords: "file upload vulnerability, php shell, extension bypass, MIME spoofing, .htaccess, polyglot image, web security"

# Schema.org Article Type
schema_type: TechArticle

# Reading difficulty level
audience: intermediate

# Language
lang: en

# Comments
comments: true
share: true

---

## Introduction

File upload vulnerabilities remain a prevalent issue in web applications, often leading to severe security breaches such as remote code execution (RCE). This technical report documents the process of testing and exploiting these vulnerabilities in the Cyber File Upload Workshop labs, spanning Levels 1 through 6. The labs simulate progressive hardening of a file upload feature, allowing for the demonstration of various bypass techniques in a controlled environment. The analysis is based on code reviews, manual testing, and proxy interception, emphasizing secure development practices.

## Executive Summary

This lab series explores **File Upload vulnerabilities** across six increasing security levels (Level 1 → Level 6).  

The objective was to gain **remote code execution (RCE)** by uploading and executing a crafted file on the web server.  

During early levels (1–3), unrestricted or weakly filtered uploads allowed direct PHP shell execution.  

Later levels introduced extension and MIME-type checks, progressively hardening the server — but multiple bypasses remained possible until Level 6.

**Outcome:** RCE achieved up to Level 5 using a combination of extension obfuscation, MIME spoofing, and `.htaccess` manipulation.  

Level 6 introduced stronger content validation that blocked server-side code execution.  

**Risk Level:** `Critical` — successful exploitation leads to **full server compromise**.

---

## Level-by-Level Analysis

The labs implemented incremental security checks. Each level's description includes the defensive logic, bypass technique, observed results, and lessons learned.

### Level 1

**Defensive Mechanism:** No validation on file type, extension, or content. Files are uploaded directly to a user-specific directory under `/upload/` and served publicly.

**Bypass Technique:** Direct upload of a server-side executable file (e.g., a simple PHP script).

**Results/Observations:** The file was stored at `/upload/<session_id>/shell.php` and accessible via URL. Accessing the URL executed the payload, confirming RCE. The server processed the file as PHP, outputting the marker string.

**Lessons Learned:** Absence of any checks allows trivial exploitation. Relying on client-side restrictions (if any) is ineffective, as they can be bypassed with proxies.

### Level 2

**Defensive Mechanism:** Client-side JavaScript check for allowed extensions (e.g., images only), but no server-side enforcement. Server accepts any uploaded file.

**Bypass Technique:** Intercepted the upload request via proxy and modified the filename extension to a server-executable format (e.g., .phtml).

**Results/Observations:** The file was uploaded successfully to `/upload/<session_id>/shell.phtml`. URL access triggered execution, displaying the embedded marker. No server rejection occurred.

**Lessons Learned:** Client-side validations are unreliable and easily circumvented. Server-side checks are essential for enforcing restrictions.

### Level 3

**Defensive Mechanism:** Server-side blacklist of extensions (e.g., blocking .php, .phtml via `in_array($extension, ['php', 'phtml'])`).

**Bypass Technique:** Used alternative executable extensions supported by the server (e.g., .php5 or .pht).

**Results/Observations:** File stored at `/upload/<session_id>/shell.php5`. Access via URL executed the code, confirming RCE. The blacklist did not cover all PHP variants.

**Lessons Learned:** Blacklists are incomplete and prone to omissions. Whitelists for allowed extensions are more secure, combined with content validation.

### Level 4

**Defensive Mechanism:** Server-side extension whitelist (e.g., allowing only .jpg, .png) via `end(explode('.', $filename))` and `in_array`.

**Bypass Technique:** Uploaded a .htaccess file to override MIME handlers, mapping non-executable extensions (e.g., .txt) to PHP execution.

**Results/Observations:** .htaccess was accepted and applied in the upload directory. Subsequent upload of shell.txt executed on access, enabling RCE. The server honored the handler override.

**Lessons Learned:** Allowing uploads of configuration files like .htaccess can undermine restrictions. Disable overrides in server config (e.g., AllowOverride None).

### Level 5

**Defensive Mechanism:** Server-side check of client-supplied MIME type (e.g., via $_FILES['file']['type']) against a whitelist (images only).

**Bypass Technique:** Intercepted the request and spoofed the Content-Type header to match the whitelist (e.g., image/jpeg), while uploading an executable file.

**Results/Observations:** File stored at `/upload/<session_id>/shell.php`. URL access executed the payload. The server trusted the manipulated MIME type.

**Lessons Learned:** Client-controlled data like $_FILES['type'] is untrustworthy. Use server-side MIME detection libraries like finfo for validation.

### Level 6

**Defensive Mechanism:** Server-side MIME validation using `finfo_file` on the temporary file, whitelisting image types (image/jpeg, image/png, image/gif).

**Bypass Technique:** Created a polyglot file starting with valid image magic bytes (e.g., GIF89a header) but containing embedded server-side code.

**Results/Observations:** File uploaded to `/upload/<session_id>/shell.php` and executed on access, as the server processed it as PHP despite the image MIME. RCE was achieved, allowing root file reads.

**Lessons Learned:** Even robust MIME checks can be bypassed with crafted files. Re-encode uploads server-side to strip embedded code.

## Technical Analysis Overview

Common weaknesses in the code samples include:
- Reliance on client-supplied data (e.g., MIME types, filenames).
- Storage of uploads in webroot with execution privileges.
- Lack of filename sanitization, enabling path traversal or extension bypasses.

Potential attack vectors:
- Direct PHP execution: Upload and access a .php file in unprotected levels.
- Extension bypass: Use variants like .php5 or case-sensitive mismatches (.PHP).
- Content-Type spoofing: Alter headers in requests for MIME-whitelisted uploads.
- Polyglot images: Files valid as images but executable as scripts.
- EXIF metadata abuse: Embed code in image metadata, exploitable if parsed.
- .htaccess override: Change server handlers for arbitrary extensions.
- LFI chaining: Upload malicious files and include them via LFI vulnerabilities.
- Log poisoning: Inject code into logs and access via inclusion.

These vectors succeed because:
- $_FILES['type'] is client-controlled and forgeable.
- finfo_file checks only magic bytes, allowing polyglots with valid headers followed by code.
- Unsanitized filenames enable executable extensions or traversals.

## Reproduction (Safe, Lab-Only)

Reproduce in a controlled lab environment using harmless markers (e.g., echo "test";) instead of shells. Clean up after each test.

- **Level 1:** Upload a file with PHP marker and access the returned URL to verify output.
- **Level 2-3:** Modify extension to server-accepted variants (e.g., .phtml) and check execution via URL.
- **Level 4:** Upload .htaccess (if allowed) to remap handlers; follow with a marker file under a different extension and access.
- **Level 5:** Proxy-intercept upload, alter Content-Type to whitelist match, upload marker, and verify execution.
- **Level 6:** Craft an image-valid file with embedded marker, upload, and access to confirm output. Note: Use safe markers only.

Safety note: All actions were performed in permitted lab settings; delete test files post-experiment.

## Impact & Severity

In production, these vulnerabilities enable RCE, allowing attackers to exfiltrate data, move laterally, or pivot to internal systems. Severity: **Critical** (CVSS 9.8) due to unauthenticated exploitation and full system compromise potential.

## Remediation

Implement these specific actions:
- Validate inputs securely: Use server-side content analysis (e.g., getimagesize() or imagecreatefromstring()) alongside MIME checks.
- Prevent execution: Store uploads outside webroot; serve via read-only handlers if public.
- Sanitize filenames: Randomize names, strip traversal characters (../), whitelist safe extensions.
- Re-encode images: Use GD/Imagick to resave uploads, removing metadata.
- Harden server: Set Content-Security-Policy headers, disable .htaccess (AllowOverride Off), restrict permissions (no execute).
- Scan uploads: Integrate antivirus/malware scanners and WAF rules for shell patterns.
- Monitor: Alert on suspicious uploads or executions.

## Detection & Forensics

Indicators in logs:
- Unusual multipart POST requests.
- New files with executable extensions in upload directories.
- Requests to /upload/* with query parameters.

Investigation queries:
- Find files: `find /upload -type f -mtime -1`.
- Check access logs: grep for 200 responses to uploaded paths.
- Compare timestamps: Match session IDs with file creation times.

## Proof and Evidence

Collect:
- Proxy logs of request/response for successful uploads.
- URLs and response bodies showing marker execution.
- Relevant code snippets (e.g., upload validation logic).
- Directory permissions and ownership for /upload.

## Appendix: Secure Upload Pipeline Checklist

- Whitelist extensions + validate content with server-side checks + re-encode images.
- Store outside webroot with random filenames and non-executable permissions.
- Strip metadata and scan for embedded scripts.
- Harden server: Disable .htaccess, restrict handlers, update libraries.
- Test with monitoring for anomalies.

---

## Final Note

File upload vulnerabilities are among the most severe web flaws — a single misconfiguration can open a door to full compromise.

Every developer and blue team engineer should treat file handling as untrusted code and design pipelines that neutralize executable payloads before they ever touch disk.

---

*Last updated: October 15, 2025*





