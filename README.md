<p align="center">
  <img src="https://github.com/user-attachments/assets/1dccf312-6838-455f-84fd-a2a76b7d11f1" width="300" height="300">
</p>

<h1 align="center">DEDSEC ETA</h1>
<h3 align="center">Email Threat Analyzer</h3>

---

## Overview

DEDSEC ETA is an automated email threat detection tool built for security operations workflows. It connects directly to a Gmail inbox via IMAP, ingests incoming messages, and applies a layered set of detection rules to surface phishing attempts, business email compromise indicators, and malicious payloads. The tool is designed to demonstrate practical competency in threat analysis, automation, and detection engineering — core competencies expected of a Security Operations Center (SOC) Analyst.

Every email is decomposed and inspected across nine independent detection layers. Results are scored, categorized by risk level, mapped to MITRE ATT&CK technique IDs, and presented in a structured table format suitable for triage or incident response handoff. When report generation is enabled, each flagged email also yields a professional PDF report, a machine readable JSON export, and IOC extracts in STIX 2.1, MISP, and CSV formats.

---

## Detection Layers

### Authentication Header Analysis (SPF / DKIM / DMARC)
The `Received-SPF` and `Authentication-Results` headers are parsed to determine whether the sending server passed or failed SPF, DKIM, and DMARC checks. A failed SPF check is a high confidence indicator that the sender address was spoofed. DKIM and DMARC failures indicate that either the message integrity could not be verified or the sending domain's policy was violated. Results are displayed with color coded pass or fail badges and contribute significantly to the risk score.

### Sender Spoofing Detection
Three impersonation vectors commonly exploited in business email compromise attacks are checked:

* **Reply-To Mismatch**: The `Reply-To` header domain is compared against the `From` domain. Attackers often set a legitimate looking `From` address while routing replies to a different attacker controlled inbox.
* **Return-Path Mismatch**: The envelope `Return-Path` domain is compared against the `From` domain. A mismatch indicates that the visible sender differs from the actual mail origin.
* **Display Name Spoofing**: The display name portion of the `From` header is checked against a list of trusted brand names. If the display name contains a known brand but the actual email domain does not belong to that brand, it is flagged as impersonation.

### Linguistic Analysis
The tool tokenizes email body text and applies a combination of natural language processing techniques to identify anomalies common in phishing campaigns. A curated corpus of legitimate English vocabulary is used to flag misspelled or deliberately obfuscated words. The system also scans for urgency triggers, scare phrasing, requests for sensitive credentials, and impersonation of trusted brands such as PayPal, Microsoft, Apple, and financial institutions. Generic greetings that lack recipient personalization are similarly flagged, as these are strong indicators of mass phishing operations.

### URL and Domain Inspection
All hyperlinks embedded in HTML content and all plaintext URLs extracted from the message body are collected and normalized. Each domain is inspected against several threat feeds built directly into the tool:

* **URL Shortener Detection**: The tool maintains a reference list of over 40 known URL shortening services. Shortened links are flagged because they obscure the true destination and are heavily abused in phishing campaigns.
* **Tunneling and Port Forwarding Services**: Domains associated with services such as ngrok, Cloudflare Tunnel, LocalTunnel, and similar platforms are flagged. These services allow attackers to expose locally hosted phishing pages to the public internet behind ephemeral domains.
* **Typosquatting Detection**: Each extracted domain is compared against a set of high value target brands using fuzzy string matching. Domains that closely resemble legitimate services but contain minor character variations are identified as potential typosquatting attempts.

### Attachment Inspection and VirusTotal Lookup
File attachments are examined for properties that commonly indicate malicious intent. The tool cross references file extensions against known dangerous types: direct executables, script files, macro enabled Office documents, and compressed archives. Double extension patterns designed to deceive users about file type are also detected.

Beyond extension based checks, every attachment payload is hashed using SHA256 and MD5. The SHA256 hash is queried against the VirusTotal API to retrieve multi vendor detection ratios. Results are cached locally to avoid redundant API calls on repeat attachments. When one or more vendors flag a file as malicious, the detection count is surfaced in the terminal output and contributes weight to the risk score.

### WHOIS Domain Age Check
The sender's domain is queried via WHOIS to determine its registration date. Domains registered less than 30 days ago are flagged as high risk. Newly registered domains are overwhelmingly associated with phishing campaigns and disposable attacker infrastructure. The domain age and creation date are displayed in the terminal output and highlighted prominently in the PDF report.

### MITRE ATT&CK Mapping
Every detection signal is tagged with the corresponding MITRE ATT&CK technique ID. This bridges the gap between raw detection output and the threat intelligence framework used by enterprise SOCs. Mapped techniques include T1566 (Phishing), T1566.001 (Spearphishing Attachment), T1566.002 (Spearphishing Link), T1557 (Adversary-in-the-Middle), T1027.011 (Obfuscated Files: URL Shortening), T1090.001 (Proxy: Internal Proxy), T1583.001 (Acquire Infrastructure: Domains), T1204.002 (User Execution: Malicious File), and T1027.002 (Obfuscated Files: Archive).

### Risk Scoring
Each detection layer contributes weighted points to a composite risk score. High confidence indicators such as SPF failures, tunneling domains, VirusTotal detections, and sender spoofing carry the most weight. Supplementary signals like keyword matches, content flags, and misspelling volume contribute incrementally. The final score maps to a three tier classification: **Safe**, **Suspicious**, or **Malicious**.

### Trusted Sender Whitelisting
A configurable whitelist of sender addresses and trusted domains prevents false positives from known legitimate sources. Emails matching entries in this list are silently skipped during scanning, reducing noise and allowing the analyst to focus on genuinely unknown or suspicious senders.

---

## Operational Workflow

1.  The analyst configures a Gmail account with an application specific password. Credentials are encrypted at rest using an XOR cipher with Base64 encoding and stored in a local configuration file.
2.  A VirusTotal API key can optionally be placed in a `virustotal.api` file to enable hash based attachment lookups. The tool functions fully without it.
3.  On execution, the tool presents a command line interface with scan options: target a specific sender, sweep the entire inbox, generate reports with IOC exports, or purge the spam folder.
4.  Emails are fetched in configurable batches and processed concurrently using a thread pool for efficiency.
5.  Each analyzed email produces a structured terminal report with risk classification, score, authentication results, keywords, URLs, attachment flags, VirusTotal detections, domain age, and MITRE ATT&CK tags.
6.  When report generation is enabled, each flagged email yields a PDF report, a JSON export, and IOC extracts in STIX 2.1, MISP, and CSV formats saved to a `reports/` directory. The scan concludes with a summary dashboard showing aggregate statistics.

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
 Auth Headers       SPF: fail, DKIM: fail, DMARC: pass
 Keywords           verify, account, urgent
 URLs               http://bit.ly/fake-login
 Interesting Words  acount, verifcation, updat
 Attachment Flags   Executable file: invoice.exe
                    VT Detection: invoice.exe flagged by 12/70 vendors
 VirusTotal         invoice.exe: 12 malicious / 2 suspicious / 70 total
 Domain Age         phishingsite.com: 3 days old (created 2026-07-20)
 MITRE ATT&CK       SPF Fail: T1566.001 (Spearphishing Attachment)
                    Shortened URL: T1027.011 (Obfuscated Files)
                    Executable Attachment: T1204.002 (Malicious File)
                    VT Detection: T1204.002 (Malicious File)
                    Domain Age: T1583.001 (Acquire Infrastructure)
 Content Flags      Urgency or scare tactic detected
                    Sensitive info requested
                    Display name spoofing: 'PayPal Support' but domain is phishingsite.com
                    Possible Phishing
────────────────────────────────────────────────────────────
 Domain            Typosquat         Tunnel   Shorten
────────────────────────────────────────────────────────────
 [1] bit.ly        NOT               NOT      Yes
 [2] ngrok.io      NOT               Yes      NOT
────────────────────────────────────────────────────────────
```

**Scan Summary (displayed after batch completion):**

```
==================================================
           SCAN SUMMARY
--------------------------------------------------
   Total scanned:           247
   Flagged:                  31  (12.5%)
     Malicious:               8
     Suspicious:             23
     Safe:                  184
   Whitelisted (skipped):     0
--------------------------------------------------
   Top flagged domains:
     ngrok.io  (4)
     bit.ly    (3)
   Top keywords:
     verify    (12)
     urgent    (9)
--------------------------------------------------
   Average risk score:       3.2
   Reports generated:        31 PDF / 31 JSON / 31 IOC
==================================================
```

---

## Reporting

When report generation is enabled, DEDSEC ETA produces multiple artifacts for every flagged email, all saved to a `reports/` directory within the project root.

**PDF Report.** A professionally formatted document with a dark header banner, color coded verdict block (green for Safe, amber for Suspicious, red for Malicious), full email metadata, and nine numbered analysis sections: Authentication Headers, Sender Spoofing Analysis, Content Analysis, URL and Domain Analysis, Attachment Analysis, VirusTotal Results, MITRE ATT&CK Mapping, and Scoring Methodology. The layout uses section headers, alternating row fills, PASS/FAIL status badges, and a domain intelligence table to produce a report ready for inclusion in an incident case file or for handoff to a senior analyst.

**JSON Export.** A machine readable export structured for ingestion into SIEM platforms, case management systems, or custom dashboards. The schema includes analysis metadata, verdict with numeric score, authentication results, spoofing indicators, detection results (keywords, URLs, misspellings, content flags, attachment warnings, domain analysis), attachment hashes and VirusTotal vendor statistics, domain age, and MITRE ATT&CK technique tags.

**IOC Exports.** Three threat intelligence sharing formats are generated for every email classified as Suspicious or Malicious:

* **STIX 2.1** — A standards compliant bundle containing an identity object for the tool, an indicator object for the phishing email, and observed data SCOs for every extracted IP, domain, URL, and file hash.
* **MISP** — A MISP compatible event JSON with typed attributes ready for import into a MISP instance.
* **CSV** — A flat IOC table with `type`, `value`, and `context` columns suitable for Splunk, ELK, or Excel ingestion.

---

## Technology Stack

The tool is written in Python and leverages well established libraries for each analysis domain. `nltk` and `pyspellchecker` handle natural language processing and typo detection. `BeautifulSoup4` parses HTML email bodies for URL extraction. `tldextract` normalizes domain names for reliable matching. `tabulate` generates the structured terminal output tables. `fpdf` produces professional PDF reports. `python-whois` queries domain registration data for age verification. The `imaplib` and `smtplib` standard library modules manage Gmail connectivity. `concurrent.futures` enables multi threaded email processing. `hashlib` computes attachment checksums for VirusTotal lookups. `ipaddress` validates extracted IPs for IOC export. `json` and `csv` handle structured data export in multiple formats.

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

This tool has been tested and verified on the following Linux distributions:

*   Kali Linux
*   Parrot OS
*   Ubuntu

---

## SOC Relevance

This project demonstrates several competencies directly applicable to a SOC Analyst role:

*   **Threat Detection Engineering**: Designing and implementing multi layered detection logic that combines signature based rules, header authentication checks, behavioral heuristics, and external threat intelligence feeds.
*   **Email Security Analysis**: Understanding the full anatomy of phishing emails including SPF/DKIM/DMARC authentication, Reply-To and Return-Path spoofing, display name impersonation, social engineering language patterns, URL obfuscation techniques, and malicious attachment characteristics.
*   **Malware Triage**: Computing cryptographic hashes for suspicious attachments and cross referencing them against VirusTotal to determine multi vendor detection ratios — a core Tier 1 SOC workflow.
*   **Threat Intelligence**: Mapping detections to MITRE ATT&CK technique IDs and exporting IOCs in STIX 2.1 and MISP formats. This demonstrates familiarity with the frameworks and data formats used by threat intel teams and ISACs.
*   **Automation and Scripting**: Building tools that reduce manual triage effort through automated ingestion, analysis, structured reporting, and IOC extraction.
*   **Incident Documentation**: Generating professional PDF reports, structured JSON exports, and multi format IOC extracts suitable for case management, stakeholder communication, and audit trails.
*   **Security Operations Mindset**: Prioritizing actionable output, minimizing false positives through whitelisting, presenting scan summary dashboards with aggregate metrics, and delivering findings in formats that support rapid decision making during incident response.
*   **Adversary Technique Awareness**: Recognizing real world attacker methodologies such as typosquatting, tunneling services for C2 or phishing infrastructure, newly registered domains, and social engineering language patterns — all mapped to the MITRE ATT&CK framework.

---

## Disclaimer

This tool is intended exclusively for educational purposes and authorized security assessments. The end user assumes complete responsibility for its use. The developer assumes no liability and is not responsible for any misuse or damage caused by this program. Always obtain explicit permission before analyzing email accounts that you do not own.
