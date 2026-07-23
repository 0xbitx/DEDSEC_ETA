<p align="center">
  <img src="https://github.com/user-attachments/assets/1dccf312-6838-455f-84fd-a2a76b7d11f1" width="300" height="300">
</p>

<h1 align="center">DEDSEC ETA</h1>
<h3 align="center">Email Threat Analyzer</h3>

---

## Overview

DEDSEC ETA is an automated email threat detection tool built for security operations workflows. It connects directly to a Gmail inbox via IMAP, ingests incoming messages, and applies a layered set of detection rules to surface phishing attempts, business email compromise indicators, and malicious payloads. The tool is designed to demonstrate practical competency in threat analysis, automation, and detection engineering — core competencies expected of a Security Operations Center (SOC) Analyst.

Every email is decomposed and inspected across nine independent detection layers. Results are scored, categorized by risk level, mapped to MITRE ATT&CK technique IDs, and presented in a structured table format suitable for triage or incident response handoff. Report generation produces a professional PDF, a machine readable JSON export, and IOC extracts in STIX 2.1, MISP, and CSV formats for every flagged email.

---

## Detection Layers

### Authentication Header Analysis (SPF / DKIM / DMARC)
The `Received-SPF` and `Authentication-Results` headers are parsed to determine whether the sending server passed or failed SPF, DKIM, and DMARC checks. A failed SPF check is a high confidence indicator that the sender address was spoofed. DKIM and DMARC failures indicate that either the message integrity could not be verified or the sending domain's policy was violated.

### Sender Spoofing Detection
Three impersonation vectors commonly exploited in business email compromise attacks are checked. **Reply-To mismatch** compares the `Reply-To` header domain against the `From` domain — attackers often set a legitimate looking `From` address while routing replies to an attacker controlled inbox. **Return-Path mismatch** compares the envelope `Return-Path` against the visible `From` domain. **Display name spoofing** checks whether the display name contains a known brand name while the actual email domain does not belong to that brand.

### Linguistic Analysis
The tool tokenizes email body text using NLTK and applies natural language processing techniques to identify anomalies. A curated corpus of legitimate English vocabulary flags misspelled or deliberately obfuscated words. The system also scans for urgency triggers, scare phrasing, requests for sensitive credentials, and impersonation of trusted brands such as PayPal, Microsoft, Apple, and financial institutions. Generic greetings that lack recipient personalization are flagged as mass phishing indicators.

### URL and Domain Inspection
All hyperlinks embedded in HTML and all plaintext URLs extracted from the message body are collected and normalized. Each domain is inspected against several threat feeds:

* **URL Shortener Detection** — reference list of over 40 known shortening services flagged because they obscure the true destination
* **Tunneling and Port Forwarding Services** — domains associated with ngrok, Cloudflare Tunnel, LocalTunnel, and similar platforms flagged as attacker infrastructure
* **Typosquatting Detection** — fuzzy string matching against trusted brand domains to identify lookalike domains

### Attachment Inspection and VirusTotal Lookup
File extensions are checked against known dangerous types: direct executables, script files, macro enabled Office documents, and compressed archives. Double extension patterns designed to deceive users are also detected. Every attachment payload is hashed using SHA256 and MD5, and the hash is queried against the VirusTotal API for multi vendor detection ratios. Results are cached locally to avoid redundant API calls.

### WHOIS Domain Age Check
The sender's domain is queried via WHOIS to determine its registration date. Domains registered less than 30 days ago are flagged as high risk — newly registered domains are overwhelmingly associated with phishing campaigns and disposable attacker infrastructure.

### MITRE ATT&CK Mapping
Every detection signal is tagged with the corresponding MITRE ATT&CK technique ID: T1566 (Phishing), T1566.001 (Spearphishing Attachment), T1566.002 (Spearphishing Link), T1557 (Adversary-in-the-Middle), T1027.011 (Obfuscated Files: URL Shortening), T1090.001 (Proxy: Internal Proxy), T1583.001 (Acquire Infrastructure: Domains), T1204.002 (User Execution: Malicious File), and T1027.002 (Obfuscated Files: Archive).

### Risk Scoring
Each detection layer contributes weighted points to a composite risk score. High confidence indicators such as SPF failures, tunneling domains, VirusTotal detections, and sender spoofing carry the most weight. The score maps to a three tier classification: **Safe**, **Suspicious**, or **Malicious**.

### Trusted Sender Whitelisting
A configurable whitelist of sender addresses and trusted domains prevents false positives from known legitimate sources. Emails matching entries in this list are silently skipped during scanning.

---

## Operational Workflow

1.  Configure a Gmail account with an application specific password. Credentials are encrypted at rest using an XOR cipher with Base64 encoding.
2.  Optionally place a VirusTotal API key in `virustotal.api` to enable hash based attachment lookups. The tool functions fully without it.
3.  Select a scan mode: target a specific sender, sweep the entire inbox, real-time monitor, or purge spam.
4.  Emails are fetched in configurable batches and processed concurrently via thread pool.
5.  Each analyzed email produces a structured terminal report. Reports (PDF + JSON + STIX + MISP + CSV) are generated automatically for every flagged email. The scan concludes with a summary dashboard.

---

## Sample Output

```
FROM:         someone@phishingsite.com
SUBJECT:      Verify Your Account Now
RISK:         Malicious
SCORE:        14

────────────────────────────────────────────────────────────
 Type               Value
────────────────────────────────────────────────────────────
 Auth Headers       SPF: Fail | DKIM: Fail | DMARC: Pass
 Keywords           verify, account, urgent
 URLs               http://bit.ly/fake-login
 Interesting Words  acount, verifcation, updat
 Attachment Flags   Executable file: invoice.exe
                    VT Detection: invoice.exe flagged by 12/70 vendors
 VirusTotal         invoice.exe: 12 malicious / 2 suspicious / 70 total
 Domain Age         phishingsite.com: 3 days old (created 2026-07-20)
 MITRE ATT&CK       T1566.001 - Spearphishing Attachment
                    T1027.011 - Obfuscated Files: URL Shortening
                    T1204.002 - User Execution: Malicious File
                    T1583.001 - Acquire Infrastructure: Domains
 Content Flags      Urgency or scare tactic detected
                    Sensitive info requested
                    Display name spoofing: 'PayPal Support' but domain is phishingsite.com
                    Possible Phishing
────────────────────────────────────────────────────────────
 Domain            Typosquat         Tunnel   Shorten
────────────────────────────────────────────────────────────
 [1] bit.ly        No                No       Yes
 [2] ngrok.io      No                Yes      No
────────────────────────────────────────────────────────────
```

---

## Reporting

Every scan produces multiple artifacts per flagged email, saved to `reports/`:

| Format | File Pattern | Purpose |
|---|---|---|
| PDF | `email_report_*.pdf` | Incident case file attachment, analyst handoff |
| JSON | `email_analysis_*.json` | SIEM/SOAR ingestion, custom dashboards |
| STIX 2.1 | `iocs_*.stix.json` | Threat intel sharing with ISACs and other SOCs |
| MISP | `iocs_*.misp.json` | Direct import into MISP instances |
| CSV | `iocs_*.csv` | Splunk, ELK, or Excel ingestion |

The **Scan Summary** dashboard prints aggregate metrics after every batch scan:

```
╭──────────────────────────┬──────────────────────────────────────╮
│ SCAN SUMMARY             │                                      │
├──────────────────────────┼──────────────────────────────────────┤
│ Total scanned            │ 247                                  │
│ Flagged                  │ 31  (12.5%)                          │
│ Malicious                │ 8                                    │
│ Suspicious               │ 23                                   │
│ Safe                     │ 184                                  │
│ Whitelisted (skipped)    │ 0                                    │
├──────────────────────────┼──────────────────────────────────────┤
│ Domain                   │ ngrok.io  (4)                        │
│ Keyword                  │ verify  (12)                         │
├──────────────────────────┼──────────────────────────────────────┤
│ Average risk score       │ 3.2                                  │
│ Reports generated        │ 31 PDF / 31 JSON / 31 IOC            │
╰──────────────────────────┴──────────────────────────────────────╯
```

---

## Detection Gallery

Real-world email analyses performed by DEDSEC ETA. Each entry shows the terminal output produced during automated triage, exactly as an analyst would see it during an investigation.

---

### 1. Legitimate Email (Safe)

A Normal email from a known sender passes all authentication checks and triggers no detection layers.

```
╭────────────────────────────────────────────────────────────────────────╮
│ ╭──────────┬───────────────────────────────────────╮                   │
│ │ FROM:    │ niki@cloudns.net                      │                   │
│ ├──────────┼───────────────────────────────────────┤                   │
│ │ SUBJECT: │ How's your ClouDNS experience so far? │                   │
│ ├──────────┼───────────────────────────────────────┤                   │
│ │ RISK:    │ Safe                                  │                   │
│ ├──────────┼───────────────────────────────────────┤                   │
│ │ SCORE:   │ 1                                     │                   │
│ ╰──────────┴───────────────────────────────────────╯                   │
├────────────────────────────────────────────────────────────────────────┤
│ ╭───────────────────┬────────────────────────────────────────────────╮ │
│ │ Type              │ Value                                          │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ Keywords          │ account                                        │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ URLs              │ https://www.linkedin.com/company/cloud-dns-ltd │ │
│ │                   │ https://www.facebook.com/cloudns/              │ │
│ │                   │ https://twitter.com/ClouDNS                    │ │
│ │                   │ https://www.youtube.com/c/CloudnsNet           │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ Interesting Words │ cloudns, dns, iskar, rasheva, str              │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ Auth Headers      │ SPF: Pass | DKIM: Pass | DMARC: Pass           │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ Domain Age        │ 6073 days (created: 2009-12-06)                │ │
│ ├───────────────────┼────────────────────────────────────────────────┤ │
│ │ MITRE ATT&CK      │ T1566 - Phishing                               │ │
│ ╰───────────────────┴────────────────────────────────────────────────╯ │
├────────────────────────────────────────────────────────────────────────┤
│ ╭──────────────────┬─────────────┬──────────┬───────────╮              │
│ │ Domain           │ Typosquat   │ Tunnel   │ Shorten   │              │
│ ├──────────────────┼─────────────┼──────────┼───────────┤              │
│ │ [1] linkedin.com │ No          │ No       │ No        │              │
│ ├──────────────────┼─────────────┼──────────┼───────────┤              │
│ │ [2] facebook.com │ No          │ No       │ No        │              │
│ ├──────────────────┼─────────────┼──────────┼───────────┤              │
│ │ [3] twitter.com  │ No          │ No       │ No        │              │
│ ├──────────────────┼─────────────┼──────────┼───────────┤              │
│ │ [4] youtube.com  │ No          │ No       │ No        │              │
│ ╰──────────────────┴─────────────┴──────────┴───────────╯              │
╰────────────────────────────────────────────────────────────────────────╯

```

| Tool Output |
|----------|
| ![image-analysis](https://github.com/user-attachments/assets/89276838-8b6d-4719-9a97-fedcb17b3540) |

| Report Output |
|----------|
| Report |
| [email_report_niki_at_cloudns.net_20260724_030609.pdf](https://github.com/user-attachments/files/30319381/email_report_niki_at_cloudns.net_20260724_030609.pdf) |
| Json Report |
| [email_analysis_niki_at_cloudns.net_20260724_030609.json](https://github.com/user-attachments/files/30319383/email_analysis_niki_at_cloudns.net_20260724_030609.json) |

---

### 2. Phishing Attempt with SPF, DKIM, DMARC Fail via Tunnel URL (my own advanced spear phishing tool)

A credential harvesting email impersonating Gmail. SPF, DKIM, DMARC fails, a Tunnel URL is detected, and urgency language triggers content flags.

```
╭───────────────────────────────────────────────────────────────────────╮
│ ╭──────────┬────────────────────────────────────╮                     │
│ │ FROM:    │ anonymous.service.mailer@gmail.com │                     │
│ ├──────────┼────────────────────────────────────┤                     │
│ │ SUBJECT: │ Security Alert                     │                     │
│ ├──────────┼────────────────────────────────────┤                     │
│ │ RISK:    │ Malicious                          │                     │
│ ├──────────┼────────────────────────────────────┤                     │
│ │ SCORE:   │ 7                                  │                     │
│ ╰──────────┴────────────────────────────────────╯                     │
├───────────────────────────────────────────────────────────────────────┤
│ ╭───────────────────┬───────────────────────────────────────────────╮ │
│ │ Type              │ Value                                         │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ Keywords          │ reset, account                                │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ URLs              │ https://myaccount.google.com/notifications    │ │
│ │                   │ mix-solaris-istanbul-signal.trycloudflare.com │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ Interesting Words │ didn, https, llc, usa                         │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ Auth Headers      │ SPF: Unknown | DKIM: Unknown | DMARC: Unknown │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ Domain Age        │ 11302 days (created: 1995-08-13)              │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ Content Flags     │ Possible spoofed brand name                   │ │
│ │                   │ Sensitive info requested                      │ │
│ │                   │ Possible Phishing                             │ │
│ ├───────────────────┼───────────────────────────────────────────────┤ │
│ │ MITRE ATT&CK      │ T1566 - Phishing                              │ │
│ │                   │ T1557 - Adversary-in-the-Middle               │ │
│ │                   │ T1566 - Phishing                              │ │
│ │                   │ T1090.001 - Proxy: Internal Proxy             │ │
│ │                   │ T1566 - Phishing                              │ │
│ ╰───────────────────┴───────────────────────────────────────────────╯ │
├───────────────────────────────────────────────────────────────────────┤
│ ╭───────────────────────┬─────────────┬──────────┬───────────╮        │
│ │ Domain                │ Typosquat   │ Tunnel   │ Shorten   │        │
│ ├───────────────────────┼─────────────┼──────────┼───────────┤        │
│ │ [1] google.com        │ No          │ No       │ No        │        │
│ ├───────────────────────┼─────────────┼──────────┼───────────┤        │
│ │ [2] trycloudflare.com │ No          │ Yes      │ No        │        │
│ ╰───────────────────────┴─────────────┴──────────┴───────────╯        │
╰───────────────────────────────────────────────────────────────────────╯
```
| Spear Phishing Tool Output |
|----------|
| ![image-analysis](https://github.com/user-attachments/assets/68608280-cc38-4a6f-a016-915c2b0b08e2) |

| Tool Output |
|----------|
| ![image-analysis](https://github.com/user-attachments/assets/929fa574-a0c7-4d98-985a-be3c8609618c) |

| Report Output |
|----------|
| Report |
| [email_report_anonymous.service.mailer_at_gmail.com_20260724_023403.pdf](https://github.com/user-attachments/files/30319230/email_report_anonymous.service.mailer_at_gmail.com_20260724_023403.pdf) |
| Json Report |
| [email_analysis_anonymous.service.mailer_at_gmail.com_20260724_023403.json](https://github.com/user-attachments/files/30319238/email_analysis_anonymous.service.mailer_at_gmail.com_20260724_023403.json) |

---

---

### 3. Malicious Attachment Detected by VirusTotal

An invoice themed email carries a password protected ZIP containing an executable. VirusTotal flags the payload across 15 vendors.

```

```

---

### 4. Phishing Link via Shorten URL

An attacker hosts a phishing page behind an cloudflare tunnel, shorten and distributes the link via email. The shorten service is detected and flagged.

```

```

---



## Technology Stack

| Package | Purpose |
|---|---|
| `nltk`, `pyspellchecker` | Natural language processing, typo detection |
| `BeautifulSoup4` | HTML email body parsing, URL extraction |
| `tldextract` | Domain name normalization |
| `tabulate` | Terminal output table formatting |
| `fpdf` | PDF report generation |
| `python-whois` | Domain registration age verification |
| `imaplib`, `smtplib` | Gmail IMAP and SMTP connectivity |
| `concurrent.futures` | Multi threaded email processing |
| `hashlib` | Attachment checksum computation |
| `ipaddress` | IP validation for IOC export |
| `json`, `csv` | Structured data export |

---

## Installation

```bash
git clone https://github.com/0xbitx/DEDSEC_ETA.git
cd DEDSEC_ETA
pip3 install requests tldextract nltk beautifulsoup4 tabulate pyspellchecker idna fpdf python-whois
chmod +x dedsec-eta
```

To enable VirusTotal attachment lookups, place your API key in a file named `virustotal.api` at the project root:

```bash
echo "your-api-key-here" > virustotal.api
```

The tool functions fully without a VirusTotal key. Attachment hashing still occurs and the hashes are included in all reports for manual lookup.

To run:

```bash
./dedsec-eta
```

Alternatively, install via the Debian package:

```bash
sudo apt install ./dedsec-eta.deb
dedsec-eta
```

---

## Supported Platforms

*   Kali Linux
*   Parrot OS
*   Ubuntu

---

## SOC Relevance

This project demonstrates several competencies directly applicable to a SOC Analyst role:

*   **Threat Detection Engineering** — Designing multi layered detection logic combining signature based rules, header authentication checks, behavioral heuristics, and external threat intelligence feeds.
*   **Email Security Analysis** — Full understanding of phishing anatomy including SPF/DKIM/DMARC authentication, Reply-To and Return-Path spoofing, display name impersonation, social engineering patterns, URL obfuscation, and malicious attachment characteristics.
*   **Malware Triage** — Computing cryptographic hashes for suspicious attachments and cross referencing against VirusTotal for multi vendor detection ratios.
*   **Threat Intelligence** — Mapping detections to MITRE ATT&CK technique IDs and exporting IOCs in STIX 2.1 and MISP formats for sharing with ISACs and threat intel platforms.
*   **Automation and Scripting** — Reducing manual triage effort through automated ingestion, analysis, structured reporting, and IOC extraction.
*   **Incident Documentation** — Generating professional PDF reports, structured JSON exports, and multi format IOC extracts for case management and audit trails.
*   **Security Operations Mindset** — Prioritizing actionable output, minimizing false positives through whitelisting, and presenting scan summary dashboards with aggregate metrics.

---

## Disclaimer

This tool is intended exclusively for educational purposes and authorized security assessments. The end user assumes complete responsibility for its use. The developer assumes no liability and is not responsible for any misuse or damage caused by this program. Always obtain explicit permission before analyzing email accounts that you do not own.
