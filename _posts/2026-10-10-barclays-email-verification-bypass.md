---
title: "Barclays: Email Verification Bypass via Password Reset"
date: 2026-10-10
categories: [Web, Authentication]
tags: [authentication, bypass, email-verification, password-reset, barclays, workday]
---

## Target
 
**Barclays** is one of the largest multinational banks in the world, headquartered in London, UK, with operations in over 40 countries and tens of millions of customers globally.
 
`barclays.wd3.myworkdayjobs.com` is Barclays' official external recruitment portal, powered by **Workday**, the enterprise HR platform used by most Fortune 500 companies. This is where candidates create accounts, submit job applications, and manage their recruitment process for all Barclays positions worldwide.
 
The email verification control on this portal exists to ensure that only legitimate applicants with valid, owned email addresses can create and access candidate accounts.
 
---
 
## Summary
 
Barclays' candidate portal enforces email verification before allowing login. This control can be bypassed entirely using the password reset flow, no verification link ever needs to be clicked.
 
| Field | Value |
|---|---|
| Target | barclays.wd3.myworkdayjobs.com |
| Category | Authentication / Email Verification Bypass |
| Status | Not accepted (disputed) |
| Platform | Workday ATS |
 
---
 
## Vulnerability Details
 
The registration flow on the Barclays candidate portal is designed to require email verification before login. An unverified account returns:
 
```json
{
  "message": "Verify your account before you sign in or request a verification email.",
  "showResendVerification": true,
  "type": "ERROR"
}
```
 
However, the password reset flow does not check the account's verification state. By completing a password reset, an attacker can transition an unverified account into an authenticated session, without ever clicking the verification link.
 
---
 
## Steps to Reproduce
 
1. Register a new account on the Barclays candidate portal using an email you control
2. Do **not** click the verification link sent to your inbox
3. Attempt to log in → blocked with the error above
   
![barclays1](/assets/img/posts/barclays1.png)

5. Navigate to **Forgot Password** and request a reset for the same account

![barclays2](/assets/img/posts/barclays2.png)

7. Complete the password reset using the link sent to your email
8. Log in using the new password, **authentication succeeds**
![barclays3](/assets/img/posts/barclays3.png)
10. Access Candidate Home and job application functionality

![barclays4](/assets/img/posts/barclays4.png)

---
 
## Vulnerable Request
 
```http
POST /en-US/External_Career_Site_Barclays/initiatereset?redirect=...  HTTP/2
Host: barclays.wd3.myworkdayjobs.com
Content-Type: application/x-www-form-urlencoded
X-Workday-Client: 2026.40.35
 
username=target@email.com&successRedirectUri=...&sessionSecureToken=...
```
 
---
 
## Expected vs Actual
 
**Expected:** Password reset should not modify the account's `email_verified` state. The login endpoint should still block an unverified account after a password reset.
 
**Actual:** Completing a password reset transitions the account to an authenticated state on next login, bypassing the verification requirement entirely.
 
---
 
## Proof of Concept
 
Two test accounts were used:
 
**Account A: Verified (control)**
- Registered → clicked verification link → logged in → accessed Candidate Home ✓
**Account B: Unverified (bypass)**
- Registered → skipped verification → login blocked ✓
- Completed password reset → logged in **without clicking verification link** → accessed Candidate Home ✓
Both accounts reached identical access levels.
 
---
 
## Timeline
 
| Date | Event |
|---|---|
| Submission | Reported to Barclays Responsible Disclosure program |
| Initial response | **"Thank you for your submission, we've determined this to be valid. The client will now address this for a fix."** |
| Final response | Report closed as **Not Valid** by the client |
 
The triager initially validated the report and informed the client. The client later returned and rejected it with the following justification.

![barclays5](/assets/img/posts/barclays5.png)

---
 
## Barclays' Response
 
> *"If a user is able to reset the password via receiving an email, that means the user has access to that mailbox, hence implicitly they're validating the email address, regardless of whether the user has validated the email address before or not. This is not a vulnerability; it is expected functionality."*

![barclays6](/assets/img/posts/barclays6.png)

## My Take
 
Barclays' argument has merit from a strict ownership perspective, receiving the reset email does prove mailbox access. However, their reasoning conflates two separate security controls:
 
1. **Email verification**: confirming the address exists and belongs to the registrant, triggering any downstream onboarding logic (welcome emails, data processing consent, account activation)
2. **Password reset**: a recovery mechanism that should not double as an activation flow
The issue is not whether the user owns the mailbox. The issue is that the `email_verified` flag is never explicitly set through the intended verification flow, yet the account reaches a fully verified state. Any server-side logic that depends on `email_verified = true` (audit logs, compliance checks, activation webhooks) would see a false state.
 
Similar reports have been accepted and rewarded by other programs, the decision varies by how strictly a program models account state consistency.
 
---
 
## Remediation
 
- The login endpoint should check `email_verified` status independently of how the password was last set
- Password reset should not implicitly change the account's verification state
- Authentication controls on login and verification should be consistent regardless of which flow the user came through
 
