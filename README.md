
<p align="center">
<img src="https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExdGtjeHZtZDk0OGt1aHZ4eGhuZm41anVrYnRzbzI1cWd3NThjazY2ZSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/d4966VVu321IfzpMAz/giphy.gif", width="300", height="300">
</p>

<h1 align="center">DEDSEC_ETA</h1> 

<h4 align="center">DEDSEC_ETA is a tool that detects phishing attempts, analyzes suspicious emails, and provides real-time threat detection.</h4>

### DESCRIPTION
DEDSEC_ETA (Email Threat Analyzer) is a Linux-based tool designed to inspect and analyze email content pulled directly from Gmail inboxes. It helps identify phishing attacks, spam, malicious attachments, spoofed domains, shortened or tunneled URLs, and other suspicious elements.

Built for ethical hackers, cybersecurity learners, and security awareness teams, DEDSEC_ETA performs advanced analysis using a combination of:

  Natural Language Processing (NLP) via the nltk library to detect misspelled words, social engineering language, and risky keywords often used in phishing emails

  URL parsing and heuristic checks to identify shortened links (e.g. bit.ly, t.ly), port-forwarding tunnels (e.g. ngrok.io, cloudflare.com), and typosquatting domains through fuzzy matching

  Attachment scanning for suspicious or potentially dangerous file types like .exe, .js, .docm, or password-protected archives

  Email header analysis (SPF, DKIM, DMARC) to detect spoofed senders or misconfigured domains

  Tabulated CLI output for clean, human-readable summaries

  Real-time monitoring using IMAP IDLE for on-the-fly detection of threats as emails arrive

### FEATURES
 * Detects suspicious URLs and domain redirects

 * Identifies shortened links (e.g. bit.ly, t.ly, etc.)

 * Flags misspelled words that may indicate social engineering

 * Scans for executable and macro-enabled attachments

 * Detects spoofed brand names and scare tactics

 * Highlights phishing-related keywords

 * Detects typosquatting using fuzzy domain matching

 * Flags port-forwarding and tunneling links (e.g. ngrok, Cloudflare)

 * Supports real-time scanning using IMAP IDLE

 * Cleans spam mailbox automatically

 * Supports configurable trusted senders and domains

 * Offers a CLI-based interactive menu with tabulated output

### OUTPUT:
~~~
FROM:         someone@phishingsite.com
SUBJECT:      Verify Your Account Now
RISK:         Malicious
SCORE:        8

────────────────────────────────────────────────────────────────────────────
 Type               Value
────────────────────────────────────────────────────────────────────────────
 Keywords           verify, account
 URLs               http://bit.ly/fake-link
 Misspelled Words   acount, verifcation
 Content Flags      Urgency or scare tactic detected
                    Sensitive info requested
 Domain             bit.ly
 Typosquat          Looks like: paypal.com
 Tunnel             Port Forwarding/Tunnel
────────────────────────────────────────────────────────────────────────────
~~~

### INSTALLATION 
    * git clone https://github.com/0xbitx/DEDSEC_ETA.git
    * cd DEDSEC_ETA
    * pip3 install requests tldextract nltk beautifulsoup4 tabulate pyspellchecker idna
    * chmod +x dedsec-eta

    to run:
    * ./dedsec-eta
    
    or
    
    * sudo apt install ./dedsec-eta.deb
    * dedsec-eta
    
### TESTED ON FOLLOWING
* Kali Linux 
* Parrot OS
* Ubuntu
  
## Support

If you find my work helpful and want to support me, consider making a donation. Your contribution will help me continue working on open-source projects.

**Bitcoin Address: `36ALguYpTgFF3RztL4h2uFb3cRMzQALAcm`**
                         

### DISCLAIMER
                                       TO BE USED FOR EDUCATIONAL PURPOSES ONLY

The use of the DEDSEC_ETA is COMPLETE RESPONSIBILITY of the END-USER. Developers assume NO liability and are NOT responsible for any misuse or damage caused by this program. 
