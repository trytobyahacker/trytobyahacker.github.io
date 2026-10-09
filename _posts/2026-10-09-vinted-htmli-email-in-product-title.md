---
title: "Vinted: HTML Injection in Product Title Renders in Purchase Receipt Email"
date: 2026-10-09
categories: [Web, HTMLi]
tags: [htmli, vinted, email, phishing, bugbounty]
---

## Summary

The product title field in Vinted does not sanitize HTML tags before inserting the value into transactional emails sent to buyers. An attacker can create a listing with an HTML payload in the title, when a victim purchases the item, Vinted sends a purchase receipt email from a legitimate `@vinted.com` address with the attacker-controlled HTML rendered inside it.

The payload is invisible on the web UI (renders as plain text), meaning most researchers would dismiss it immediately. The vulnerability only manifests in the email triggered by an actual purchase.

---

## Vulnerability Details

| Field | Value |
|-------|-------|
| **Type** | HTML Injection → Email Phishing |
| **Location** | Product title field → purchase receipt email |
| **Trigger** | Victim purchases the malicious listing |
| **Email sender** | Legitimate `@vinted.com` address |
| **Severity** | Medium |
| **Status** | Reported |

---

## Discovery Method

This bug was found by creating two controlled accounts, attacker and victim, and completing a real purchase between them. The title on the web UI showed the payload as plain text, which would normally lead a researcher to dismiss it as unexploitable. Only by going through the full purchase flow and checking the receipt email was the injection confirmed.

Most hunters would miss this by only looking at the web UI.

---

## Payload

```html
<h1>pepepe</h1> or <a href="https://evil.com">Verify payment</a>
```

---

## Steps to Reproduce

1. Log in with an attacker-controlled Vinted account.
2. Create a new listing with the following payload in the **product title** field:
```html
<h1>pepepe</h1> or any htmli payload: <a href="https://evil.com">Verify payment</a>
```
3. Log in with a second (victim) Vinted account.
4. Purchase the malicious listing from the victim account.
5. Open the purchase receipt email received by the victim.
6. Observe that the HTML is rendered, `PEPEPE` or the "Verify payment" appears as a clickable link pointing to `https://evil.com`.

---

## Impact

The resulting phishing email:

- Originates from a **legitimate `@vinted.com` address**
- Is a genuine Vinted transactional email, increasing victim trust
- Contains attacker-controlled links rendered as legitimate-looking anchor text (e.g. "Verify payment", "Confirm your account")
- May bypass email security filters because it is legitimately sent through Vinted's own mail infrastructure

An attacker could direct victims to credential-harvesting pages, fake payment portals, or malware delivery sites. The attack scales easily by creating low-cost listings designed to attract buyers, targeting multiple Vinted users through a single legitimate email flow.

---

## Recommendation

- HTML-encode all user-supplied input (product title, description, username) before inserting into HTML email templates.
- Apply an allowlist of permitted characters for the title field on the **backend**, not only the frontend.
- Audit all other user-controlled fields that may be included in transactional emails.
