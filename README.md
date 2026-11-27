# NETWORKWALKS-B084-FINAL-PROJECT-CYBERSECURITY
# Penetration Testing Report

**Web Application Assessment — Mediroza General Hospital Portal**

Final Project | Cybersecurity | NetworkWalks

| | |
|---|---|
| **Pentester Name** | Omar Moamen |
| **Program / Batch** | B084M — NetworkWalks Cybersecurity Internship |
| **Date** | 27 November 2026 |
| **Target** | `medirozahospital.com` (`199.188.201.16`) — a fictional training environment built for this internship module ("Mediroza General Hospital" and "Mediroza CMS" do not exist outside this exercise) |
| **Permission secured from client?** | Yes. The target is a purpose-built NetworkWalks lab environment provided for this assignment, not a real organization. |
| **Phases covered** | **Phase 1:** Reconnaissance & Footprinting<br>**Phase 2:** Directory & File Enumeration<br>**Phase 3:** Authentication Analysis & Exploitation<br>**Phase 4:** File-Level Analysis (encrypted document recovery) |

---

## 1. Liability Disclaimer

I performed these activities only against the NetworkWalks-provided lab target for this module. All material in this report is for education and training purposes only. Do not use anything here against systems you do not own or have explicit written permission to test. The instructor, the authors and NetworkWalks are not responsible for any misuse of this knowledge. Unauthorized access to computer systems is a crime in most jurisdictions, even where no damage results.

## 2. Introduction

This report documents the final-project assessment of a simulated hospital patient portal (`medirozahospital.com`), run as part of the NetworkWalks Cybersecurity internship. Unlike the Week 2 report, which was limited to passive footprinting and host discovery, this assignment went end-to-end: reconnaissance, directory/file enumeration, authentication-mechanism analysis, exploitation of a real injection flaw, and recovery of encrypted patient documents.

The assessment follows the natural order the target was actually explored in, including the dead ends — several early avenues (FTP, dotfile enumeration, a fake CMS version tag) led nowhere, and are documented as negative results rather than left out, because knowing what *doesn't* work is part of a real engagement.

**Scope and authorization:** all testing was carried out against the NetworkWalks-provided target for this module, within the tools and techniques taught in the course material (footprinting tools, Burp Suite, gobuster, John the Ripper / hashcat).

## 3. Tools Used

| Tool | Purpose | Notes |
|---|---|---|
| whois, nslookup, dnsrecon | Domain / DNS footprinting | Returned only the server IP and basic DNS records — no exploitable detail |
| whatweb, curl | Web technology / header fingerprinting | Surfaced the `Mediroza CMS 1.4.2` generator tag, later confirmed to be a decoy (no such CMS exists) |
| wafw00f | WAF detection | Used to understand why some automated tooling (sqlmap) behaved inconsistently |
| FTP client | Test anonymous FTP access | Port 21 open (Pure-FTPd); anonymous login explicitly disabled by server banner — dead end |
| robots.txt (manual review) | Identify disallowed paths | Revealed `/patient`, `/staff`, and `/old` — paths the site wanted hidden from search engines, not from a human reader |
| gobuster | Directory/file brute-forcing | Used against `/patient/` and `/patient/reports/`; mostly returned uniform 403s (server-wide block, not a real finding) |
| Burp Suite (Repeater / Proxy) | Manual request manipulation and response analysis | Used to compare staff vs. patient login behavior and to craft/test SQL injection payloads |
| John the Ripper / hashcat | Password cracking | Cracked 2 of 3 PDF report passwords quickly; the third required diagnosing a hash-parsing bug before it would crack |
| pdf2john, exiftool | PDF hash extraction / metadata review | Used to identify PDF encryption revision (RC4-128, Standard V2.3) |
| sqlmap | Automated SQL injection testing | Inconclusive due to WAF interference and dynamic page content; manual Burp testing was the reliable method |

## 4. Activities Performed

### 4.1 Reconnaissance & Footprinting

Standard footprinting tools (`whois`, `nslookup`, `dnsrecon`, `whatweb`, `curl -I`, `wafw00f`) were run against `medirozahospital.com`. Combined, they returned only the server's public IP address and confirmation that a web portal existed — no version-specific, directly exploitable information came out of this phase.

*(Screenshot: [`reconnaissaince tools.png`](screenshots/reconnaissaince%20tools.png))*

**Dead end — fake CMS fingerprint.** Every page carried `<meta name="generator" content="Mediroza CMS 1.4.2">`. A vulnerability search for "Mediroza CMS" returned nothing — it isn't a real product. This was treated as a planted decoy rather than a real fingerprint, and no further time was spent chasing CVEs for it. **Lesson:** verify a disclosed product/version actually exists before researching vulnerabilities for it.

**Dead end — FTP.** Port scanning showed FTP (21/tcp) open, running Pure-FTPd. Connecting showed the banner explicitly states `This is a private system - No anonymous login`, and a login attempt confirmed this (`530 Login authentication failed`). No credentials were available, so this path was correctly abandoned rather than brute-forced.

*(Screenshot: [`nmap scanning.png`](screenshots/nmap%20scanning.png))*

### 4.2 Directory & File Enumeration

**`robots.txt` review.** Rather than guessing paths blind, `robots.txt` was checked first, since it is a public file that site owners use to tell search engines which paths *not* to index. It disallowed three paths: `/patient`, `/staff`, and `/old`. This is a textbook case of `robots.txt` backfiring as a security control — it doesn't block access, it only blocks *search-engine indexing*, so it effectively handed over a map of the most sensitive areas of the site to anyone willing to read a public file.

*(Screenshot: [`robots.txt.png`](screenshots/robots.txt.png))*

#### Finding 2: Exposed Database Backup File via robots.txt

Browsing to the disallowed `/old` path directly (bypassing the robots directive, which only advises crawlers and enforces nothing) exposed a database backup file: **`mediroza_db_backup_2019.sql`**. The file was reachable with no authentication and no access control of any kind.

The dump's own header is explicit about what it contains:

```
-- mediroza_db_backup_2019.sql
-- Mediroza General Hospital - internal database backup
-- Generated by Mediroza CMS 1.4.2 backup module
-- Host: localhost    Database: mediroza_hr
-- WARNING: contains confidential staff and shareholder records
-- Backup date: 2019-08-27 02:14:03
```

It contains two full tables:
- **`staff`** — 30 records: full name, job title, department, email, phone number, national ID number, monthly salary, and date joined.
- **`shareholders`** — 10 records: shareholder name, ownership percentage, shares held, and share class.

Both tables are reproduced in full in [`data/staff.md`](data/staff.md) and [`data/shareholders.md`](data/shareholders.md), since displaying the actual exposed contents — not just describing them — is what demonstrates real impact for this kind of finding.

*(Screenshot: [`sql dump.png`](screenshots/sql%20dump.png))*

**Directory brute-forcing with gobuster.** `gobuster` was run against `/patient/` to look for further undocumented endpoints. It found `reports` (301 → `/patient/reports/`), but direct requests to that path, and to every dotfile path tested (`.git/HEAD`, `.svn/entries`, `.htaccess`, etc.), returned an identical 403 response (same body, same `Content-Length: 1242`) — a blanket server-level rule blocking those path classes outright, not evidence of an actual `.git` repository or a reachable reports directory. **Dead end, correctly ruled out** rather than chased further.

*(Screenshot: [`gobuster more directories.png`](screenshots/gobuster%20more%20directories.png))*

### 4.3 Authentication Analysis & Exploitation

Two login forms exist: `/staff/login.php` and `/patient/login.php`. Comparing them directly (as instructed by the module's guidance to "analyze any authentication page you face") surfaced real differences in how each was built.

| | Staff login | Patient login |
|---|---|---|
| Error on bad credentials | Generic: "Invalid username or password" | Specific: **"Username not found"** |
| CSRF token present | No | No |
| Behavior under `admin'--` | Rejected normally | **Authenticates successfully, no password needed** |
| Behavior under malformed SQL payload | N/A (not injectable) | **Raw `mysqli_query()` syntax error reflected in the page** |

#### Finding 3: Username Enumeration (Patient Login)

The patient login form distinguishes "Username not found" from a wrong-password case, which lets an attacker determine which usernames exist on the system without knowing any password (CWE-204, Observable Response Discrepancy). The staff login form does not have this problem — it uses a single generic error for both cases, showing inconsistent input-handling discipline between the two forms in the same application.

#### Finding 4: SQL Injection — Authentication Bypass (Patient Login)

Submitting `admin'--` as the username (with the password field left as anything) logged in successfully with no valid password supplied at all:

```
POST /patient/login.php
username=admin'--
password=test
```
→ `HTTP/2 302 Found`, `Location: portal.php`

This works because the application concatenates the username field directly into a SQL query rather than using parameterized queries; the injected `'--` closes the string literal and comments out the rest of the query (including the password check). This is a **critical authentication bypass** — it grants full account access with zero knowledge of any real credential.

*(Screenshot: [`burp suite analyzing login.php.png`](screenshots/burp%20suite%20analyzing%20login.php.png))*

#### Finding 5: Error-Based SQL Injection / Verbose Error Disclosure

Testing a deliberately malformed condition (e.g. `admin' AND 1=2--`) returned the raw backend error directly in the page's UI alert box:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the manual that
corresponds to your MySQL server version for the right syntax to use near
''' AND 1=2--' at line 1
```

This confirms the injection point is live beyond the simple auth-bypass case, and also confirms `display_errors` is enabled in production — handing back the exact backend language (PHP/mysqli), on top of the injection itself. The combination (unsanitized query + verbose errors shown to the end user) is itself a disclosure issue independent of the injection.

#### Finding 6: Broken Access Control — Admin Account Sees All Patients' Records

Once authenticated as `admin` (via Finding 4), the patient portal's "My Reports" page listed **all three patients' lab reports** (S. Dlamini, P. Reddy, E. Thompson) under the one account, each linked via a predictable, sequential `download.php?id=N` parameter. A patient-facing account should only ever see its own records; this shows the portal's authorization logic is not actually tied to patient identity, it just returns whatever the session happens to have access to.

*(Screenshot: [`patient directory.png`](screenshots/patient%20directory.png))*

### 4.4 File-Level Analysis: Encrypted Lab Reports

Each of the three pathology report PDFs is password-protected. `exiftool` and `pdf2john` identified the encryption as PDF **Standard V2.3, 128-bit (RC4)** — an older, comparatively weak PDF encryption scheme.

#### Finding 7: Weak Passwords on Patient Lab Reports

Hashes were extracted with `pdf2john` and cracked with John the Ripper / hashcat using the `rockyou.txt` wordlist:

| Report | Patient | Password |
|---|---|---|
| `patient_report_1.pdf` | S. Dlamini | `123456` |
| `patient_report_2.pdf` | P. Reddy | `password` |
| `patient_report_3.pdf` | E. Thompson | `!@#$%^&` |

The first two cracked almost instantly, confirming these are among the most common passwords in existence — entirely inadequate for protecting pathology/lab results.

*(Screenshots: [`nw tools pdf 1 password.png`](screenshots/nw%20tools%20pdf%201%20password.png), [`succesful hashcat.png`](screenshots/succesful%20hashcat.png), [`patient report 1.png`](screenshots/patient%20report%201.png), [`patient report 2.png`](screenshots/patient%20report%202.png), [`patient report 3.png`](screenshots/patient%20report%203.png))*

**Dead end / tooling pitfall worth documenting — report 3.** Unlike the first two, `patient_report_3.pdf`'s hash would not crack against the same wordlist, even after billions of candidates across multiple wordlists, rule sets, and masks. The cause turned out to be a **hashcat hash-parsing bug**, not a stronger password: the PDF's permissions field in the extracted hash was `4294967292`, which is `2^32 − 4` — i.e. the signed value `-4` had been clamped/misread as an unsigned 32-bit integer, silently corrupting the hash before cracking ever began. Every attempt up to that point was guaranteed to fail regardless of wordlist quality. Converting the field back to its correct signed representation fixed the hash, and `rockyou.txt` cracked it on the very next attempt (`!@#$%^&`). **Lesson:** when a crack fails against a wordlist that already worked on near-identical files, question the hash/tooling before scaling up the attack (bigger lists, GPU, more rules) — a corrupted input will exhaust any wordlist with zero chance of success.

## 5. Risk Analysis / Impact

| # | Finding | Evidence | Impact | Risk Level |
|---|---|---|---|---|
| 1 | `robots.txt` discloses sensitive paths | `/patient`, `/staff`, `/old` listed as `Disallow` | Hands an attacker a map of sensitive directories instead of hiding them | Medium |
| 2 | Exposed database backup (`mediroza_db_backup_2019.sql`) | Reachable, unauthenticated, under `/old` | Full disclosure of 30 staff records (PII, national IDs, salaries) and 10 shareholder records (ownership structure) | **Critical** |
| 3 | Username enumeration on patient login | "Username not found" vs. generic staff error | Lets an attacker build a list of valid patient usernames pre-attack | Medium |
| 4 | SQL injection — authentication bypass | `admin'--` logs in with no password | Full, credential-free account takeover | **Critical** |
| 5 | Error-based SQL injection / verbose errors | Raw `mysqli_query()` error reflected to the user | Confirms deeper injectability; discloses backend stack | High |
| 6 | Broken access control on patient reports | Admin session saw all 3 patients' reports via `download.php?id=` | Any authenticated session can view other patients' confidential medical records | High |
| 7 | Weak PDF passwords on lab reports | `123456`, `password`, `!@#$%^&` | Trivial to crack; defeats the purpose of encrypting medical documents at rest | High |
| 8 | Fake/decoy CMS generator tag | `Mediroza CMS 1.4.2` (non-existent) | No direct impact; noted as a lesson in verifying fingerprints before researching them | Informational |
| 9 | Inconsistent input handling between staff and patient login | Staff form rejects injection cleanly; patient form does not | Shows uneven security discipline across the same application | Medium |
| 10 | No CSRF token on either login form | Reviewed form source | Login requests could be forged cross-site, though limited impact on a login (vs. a state-changing) form | Low |
| 11 | FTP anonymous login (tested, not exploitable) | `530 Login authentication failed`, banner explicitly disables anonymous access | None — documented as a closed lead | N/A |
| 12 | Dotfile/VCS paths (`.git`, `.svn`, etc.) | Uniform 403 across all tested dotfile paths | None — blanket server rule, not an actual exposed repo | N/A |

## 6. Recommendations

1. **Never store backups in a web-reachable path.** `mediroza_db_backup_2019.sql` should never have been under a directory the web server could serve at all — not even one disallowed in `robots.txt`, which provides zero actual access control. Backups belong outside the web root, with their own access restrictions.
2. **Fix `robots.txt` usage.** Do not list sensitive paths in `robots.txt` to "hide" them from search engines — this only tells engines not to index them, while publishing their existence to anyone who reads the file. Sensitive paths need real authentication/authorization, not an advisory file.
3. **Use parameterized queries everywhere.** Both the authentication bypass and the error-based injection stem from the same root cause: user input concatenated directly into SQL. Prepared statements/parameterized queries (e.g. PHP PDO with bound parameters) eliminate this entire class of bug.
4. **Disable verbose error output in production.** `display_errors` should be off; errors should be logged server-side, not rendered to the end user.
5. **Fix broken access control on patient reports.** Report access must be checked against the authenticated patient's own identity/ID, not merely "is there a valid session" — regardless of which account is logged in.
6. **Normalize generic, identical error messages for all login forms** (both username-not-found and wrong-password cases), consistently across staff and patient portals.
7. **Enforce a real password policy on anything protecting medical records**, including document-level encryption passwords — `123456` and `password` should be rejected outright by any password-setting flow.
8. **Add CSRF tokens to all forms**, including login, as defense in depth.

## 7. Conclusion

This assessment moved from passive footprinting all the way to full account takeover and exposure of confidential HR/shareholder data, and it did so largely through *compounding* small issues rather than one single catastrophic bug: a `robots.txt` entry that leaked a path, a backup file left reachable under that path, an authentication form that didn't sanitize its input, and weak passwords protecting the files that form ultimately granted access to. Several promising-looking leads (FTP, the fake CMS tag, dotfile enumeration) were tested and correctly ruled out rather than chased indefinitely — recognizing a dead end is as much a part of the process as finding a real vulnerability.

The most important technical lesson from this project was procedural rather than purely technical: when `patient_report_3.pdf` resisted cracking far longer than its siblings, the instinct to "just throw more compute/wordlist at it" was wrong — the real fix was questioning the tooling itself, which uncovered a hash-corruption bug rather than a genuinely stronger password.

## 8. Evidence Collected

| Screenshot | What it shows |
|---|---|
| ![`reconnaissaince tools.png`](screenshots/reconnaissaince%20tools.png) | whois/nslookup/dnsrecon/whatweb/curl/wafw00f output |
| ![`nmap scanning.png`](screenshots/nmap%20scanning.png) | Port scan showing FTP (21) open |
| ![`robots.txt.png`](screenshots/robots.txt.png) | `robots.txt` content disclosing `/patient`, `/staff`, `/old` |
| ![`sql dump.png`](screenshots/sql%20dump.png) | The exposed `mediroza_db_backup_2019.sql` file under `/old` |
| ![`gobuster more directories.png`](screenshots/gobuster%20more%20directories.png) | gobuster results against `/patient/` |
| ![`burp suite analyzing login.php.png`](screenshots/burp%20suite%20analyzing%20login.php.png) | Burp request/response showing the `admin'--` auth bypass |
| ![`burp suite map.png`](screenshots/burp%20suite%20map.png) | Burp site map of the application |
| ![`patient directory.png`](screenshots/patient%20directory.png) | Admin session listing all three patients' reports |
| ![`patient report 1.png`](screenshots/patient%20report%201.png) / ![`2`](screenshots/patient%20report%202.png) / ![`3`](screenshots/patient%20report%203.png) | The three recovered pathology reports |
| ![`nw tools pdf 1 password.png`](screenshots/nw%20tools%20pdf%201%20password.png) | PDF hash extraction / first crack |
| ![`succesful hashcat.png`](screenshots/succesful%20hashcat.png) | Successful hashcat run on `patient_report_3.pdf` after the signed/unsigned int32 fix |

Raw evidence: [`evidence/mediroza_db_backup_2019.sql`](evidence/mediroza_db_backup_2019.sql) — the exposed backup file in full.

Full exposed data tables: [`data/staff.md`](data/staff.md) · [`data/shareholders.md`](data/shareholders.md)
