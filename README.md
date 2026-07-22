<p align="center">
  <img src="https://github.com/user-attachments/assets/1dccf312-6838-455f-84fd-a2a76b7d11f1" width="300" height="300">
</p>

<h1 align="center">DEDSEC ETA</h1>
<h3 align="center">Email Threat Analyzer</h3>

---

## Overview

DEDSEC ETA is an automated email threat detection tool built for security operations workflows. It connects directly to a Gmail inbox via IMAP, ingests incoming messages, and applies a layered set of detection rules to surface phishing attempts, business email compromise indicators, and malicious payloads. The tool is designed to demonstrate practical competency in threat analysis, automation, and detection engineering, core competencies expected of a Security Operations Center (SOC) Analyst.

Every email is decomposed and inspected across multiple dimensions: sender reputation, linguistic patterns, embedded URLs, domain characteristics, and attachment properties. Results are scored, categorized by risk level, and presented in a structured table format suitable for triage or incident response handoff.

---

## Detection Layers

### Linguistic Analysis
The tool tokenizes email body text and applies a combination of natural language processing techniques to identify anomalies common in phishing campaigns. A curated corpus of legitimate English vocabulary is used to flag misspelled or deliberately obfuscated words. The system also scans for urgency triggers, scare phrasing, requests for sensitive credentials, and impersonation of trusted brands such as PayPal, Microsoft, Apple, and financial institutions. Generic greetings that lack recipient personalization are similarly flagged, as these are strong indicators of mass phishing operations.

### URL and Domain Inspection
All hyperlinks embedded in HTML content and all plaintext URLs extracted from the message body are collected and normalized. Each domain is inspected against several threat feeds built directly into the tool:

* **URL Shortener Detection**: The tool maintains a reference list of over 40 known URL shortening services. Shortened links are flagged because they obscure the true destination and are heavily abused in phishing campaigns.
* **Tunneling and Port Forwarding Services**: Domains associated with services such as ngrok, Cloudflare Tunnel, LocalTunnel, and similar platforms are flagged. These services allow attackers to expose locally hosted phishing pages to the public internet behind ephemeral domains.
* **Typosquatting Detection**: Each extracted domain is compared against a set of high-value target brands using fuzzy string matching. Domains that closely resemble legitimate services but contain minor character variations are identified as potential typosquatting attempts.

### Attachment Inspection
File attachments are examined for properties that commonly indicate malicious intent. The tool cross references file extensions against known dangerous types: direct executables, script files, macro enabled Office documents, and compressed archives. Double extension patterns designed to deceive users about file type are also detected. When an archive attachment includes password protection in its encoded payload, it is flagged, as password protected archives are frequently used to bypass automated malware scanners at the email gateway.

### Risk Scoring
Each detection layer contributes weighted points to a composite risk score. The scoring model emphasizes high confidence indicators such as tunneling domains and URL shortener counts while also accounting for supplementary signals like keyword matches and content flags. The final score maps to a three tier classification: **Safe**, **Suspicious**, or **Malicious**. This triage oriented output enables a SOC Analyst to quickly prioritize which emails warrant deeper investigation.

### Trusted Sender Whitelisting
A configurable whitelist of sender addresses and trusted domains prevents false positives from known legitimate sources. Emails matching entries in this list are silently skipped during scanning, reducing noise and allowing the analyst to focus on genuinely unknown or suspicious senders.

---

## Operational Workflow

1.  The analyst configures a Gmail account with an application specific password.
2.  Credentials are encrypted at rest using an XOR cipher with Base64 encoding and stored in a local configuration file.
3.  On execution, the tool presents a command line interface with scan options: target a specific sender, sweep the entire inbox, or purge the spam folder.
4.  Emails are fetched in configurable batches and processed concurrently using a thread pool for efficiency.
5.  Each analyzed email produces a structured report containing sender details, subject, risk classification, composite score, flagged keywords, extracted URLs, misspelled or anomalous words, attachment warnings, content flags, and a per domain breakdown with typosquatting, tunneling, and shortening indicators.

---

## Sample Output

```
FROM:         someone@phishingsite.com
SUBJECT:      Verify Your Account Now
RISK:         Malicious
SCORE:        8

────────────────────────────────────────────────────────────
 Type               Value
────────────────────────────────────────────────────────────
 Keywords           verify, account
 URLs               http://bit.ly/fake-link
 Interesting Words  acount, verifcation
 Content Flags      Urgency or scare tactic detected
                    Sensitive info requested
 Domain             bit.ly
 Typosquat          Looks like: paypal.com
 Tunnel             Port Forwarding/Tunnel
────────────────────────────────────────────────────────────
```

---

## Technology Stack

The tool is written in Python and leverages well established libraries for each analysis domain. `nltk` and `pyspellchecker` handle natural language processing and typo detection. `BeautifulSoup4` parses HTML email bodies for URL extraction. `tldextract` normalizes domain names for reliable matching. `tabulate` generates the structured terminal output tables. The `imaplib` and `smtplib` standard library modules manage Gmail connectivity, and `concurrent.futures` enables multi threaded email processing for scan performance.

---

## Installation

```bash
git clone https://github.com/0xbitx/DEDSEC_ETA.git
cd DEDSEC_ETA
pip3 install requests tldextract nltk beautifulsoup4 tabulate pyspellchecker idna
chmod +x dedsec-eta
```

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

This tool has been tested and verified on the following Linux distributions:

*   Kali Linux
*   Parrot OS
*   Ubuntu

---

## SOC Relevance

This project demonstrates several competencies directly applicable to a SOC Analyst role:

*   **Threat Detection Engineering**: Designing and implementing multi layered detection logic that combines signature based rules with behavioral heuristics.
*   **Email Security Analysis**: Understanding the anatomy of phishing emails, including social engineering tactics, URL obfuscation techniques, and malicious attachment patterns.
*   **Automation and Scripting**: Building tools that reduce manual triage effort through automated ingestion, analysis, and structured reporting.
*   **Security Operations Mindset**: Prioritizing actionable output, minimizing false positives through whitelisting, and presenting findings in a format that supports rapid decision making during incident response.
*   **Adversary Technique Awareness**: Recognizing real world attacker methodologies such as typosquatting, tunneling services for C2 or phishing infrastructure, and social engineering language patterns.

---

## Disclaimer

This tool is intended exclusively for educational purposes and authorized security assessments. The end user assumes complete responsibility for its use. The developer assumes no liability and is not responsible for any misuse or damage caused by this program. Always obtain explicit permission before analyzing email accounts that you do not own.
