---
title: "SecurityTrails: Payment Bypass on $1,500/mo Plan + Permanent API Key Activation"
date: 2026-10-09
categories: [Web, BusinessLogic]
tags: [payment-bypass, securitytrails, business-logic, api, bugbounty, p3]
---

## Summary

The SecurityTrails checkout flow for the Business plan ($1,500/month) failed to validate whether a payment was successfully completed before upgrading the account. Submitting a payment with insufficient funds caused the backend to upgrade the account to Business tier anyway, granting full premium access indefinitely, including a permanently active API key that renewed its quota automatically every month without any further payment.

**Bounty:** $750 — P3  

**Plan affected:** Business ($1,500/mo), 65,000 queries/month, commercial use, DSL access, associated domains, consulting services

![plan](/sec1.png)
---

## Vulnerability Details

| Field | Value |
|-------|-------|
| **Type** | Payment Bypass + Business Logic Flaw |
| **Location** | Checkout flow — Business plan upgrade |
| **Impact** | Free permanent access to $1,500/mo plan + auto-renewing API quota |
| **Severity** | P3 |
| **Bounty** | $750 |

![bounty](/bounty.png)

---

## Discovery

Found by attempting a real $1,500 purchase with an account that had insufficient funds, not to bypass anything, just to test the payment flow out of curiosity.

> *"What do I have to lose? Let me try this payment flow."*

The payment failed due to insufficient balance. The account was upgraded to Business tier anyway.

Most researchers would never test this flow because the cost acts as a natural deterrent. The barrier here was psychological, not technical.

![insufi](/insufi.png)
---

## What the Business Plan Grants

| Feature | Professional ($500/mo) | Business ($1,500/mo) |
|---------|----------------------|---------------------|
| Queries/month | 20,000 | 65,000 |
| Historical DNS | ✓ | ✓ |
| Reverse WHOIS | ✓ | ✓ |
| DSL (Domain Specific Language) | ✗ | ✓ |
| Associated domains | ✗ | ✓ |
| Consulting services | ✗ | ✓ |
| Commercial use | ✓ | ✓ |
| Rate limit | 5 req/sec | 5 req/sec |

![quota](/quota.png)
---

## Steps to Reproduce

1. Create or log in to a SecurityTrails account.
2. Navigate to the plan upgrade page and select **Business ($1,500/mo)**.
3. Initiate the payment with a card or account with **insufficient balance**.
4. The payment fails client-side / at the payment processor.
5. Observe that the **account is upgraded to Business tier anyway**.

![plan](/plan.png)

7. Generate an API key, it activates with full Business quota (65,000 queries/month, commercial use).
8. Wait 30+ days, the API key **auto-renews its quota without any payment**, indefinitely.

![invoice](/invoice.png)
---

## Impact

- **Free permanent Business plan access**, $1,500/month of value with zero payment.
- **API key remains active for 42+ days** and auto-renews monthly, confirmed by observing quota refresh without any billing event.
- **Automatable at scale**, the flow can be scripted: create account → trigger failed payment → extract API key → use/sell quota → repeat.
- API key grants **commercial use**, DSL access, 65,000 DNS/WHOIS queries/month, and associated domain lookups, capabilities valuable to threat intelligence and OSINT tooling.
- No interaction required after initial bypass, the account maintains its premium state indefinitely.

---

## Root Cause

The backend upgraded the account plan based on the **initiation** of a payment request rather than a confirmed successful payment callback from the payment processor. The payment processor's failure response was not validated server-side before granting premium access.

---

## Recommendation

- Upgrade account tier **only** upon receiving a confirmed success webhook/callback from the payment processor, never on payment initiation.
- Implement server-side verification of payment status before modifying account entitlements.
- Add a periodic billing check that downgrades accounts when no valid payment is on record for the current billing cycle.
- Audit API key activation to require a verified active subscription, not just account tier state.

---

## Timeline

| Date | Event |
|------|-------|
| 2026-XX-XX | Vulnerability discovered during checkout flow testing |
| 2026-XX-XX | Reported to SecurityTrails security team |
| 2026-XX-XX | Triaged as P3 |
| 2026-XX-XX | $750 bounty awarded |
