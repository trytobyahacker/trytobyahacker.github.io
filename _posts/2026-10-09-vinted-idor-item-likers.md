---
title: "Vinted: IDOR Exposes Identity of Item Likers + Iteraction via order_by_buyer_for_item"
date: 2026-10-09
categories: [Web, IDOR]
tags: [idor, vinted, broken-access-control, api, bugbounty, privacy]
---

## Summary

The `GET /api/v2/users` endpoint, when called with the `order_by_buyer_for_item` parameter, returns the identity of users who liked/favorited + Interaction a given item, without verifying that the requesting account owns that item.

Any authenticated user can substitute any `item_id` and retrieve who liked it, data that is **never exposed anywhere else in the product**. The public item page shows only an aggregate like count (e.g. ❤️ 26), never the identities behind it.

---
## Vulnerability Details

| Field | Value |
|-------|-------|
| **Type** | IDOR — Broken Object Level Authorization |
| **Endpoint** | `GET /api/v2/users?order_by_buyer_for_item={item_id}` |
| **Severity** | Medium |
| **Status** | Reported |


## Affected Endpoint

```http
GET /api/v2/users?per_page=100&search_text=&order_by_buyer_for_item={item_id}
Host: www.vinted.com
```
![idor](/1.png)

This endpoint is part of the legitimate "Mark as sold" flow, when a seller marks an item as sold off-platform, Vinted shows them a list of likely buyers (users who liked the item) so they can select who they sold it to.

---

## Root Cause

The backend correctly verifies that the request carries a valid authenticated session. An unauthenticated request returns:

```http
HTTP/2 401
{"code":100,"message":"Invalid authentication token","message_code":"invalid_authentication_token"}
```
![idor](/2.png)

However, it does **not** verify that the `item_id` passed in `order_by_buyer_for_item` belongs to the authenticated user. Any logged-in account can request this data for any item on the platform simply by manipulating the `item_id` in the query string.

---

## Confirmation via Native Vinted Notification

To verify the returned list genuinely reflects real item interactions, the following cross-check was performed:

1. Vinted sent a native push notification: **"amethyst1984 added your tt t5 to their favorites."**  
   The item (`tt t5`, `item_id: 10197071411`) belongs to the tester, this is ground truth.
![idor](/3.png)
![idor](/4-vinted.png)

3. The tester followed the legitimate UI flow that triggers the endpoint:  
   Notification → Inbox conversation → Details icon → Mark as sold → intercepted the request with a proxy.
![idor](/5-vinted.png)
![idor](/6-vinted.png)

5. The intercepted request:
```http
GET /api/v2/users?per_page=100&search_text=&order_by_buyer_for_item=10197071411
```

4. First user object in the response:
```json
{"id": 3166955222, "login": "amethyst1984", ...}
```
![idor](/7.png)

This confirms the endpoint's first result correctly corresponds to the real, verifiable user who liked the item, not a coincidental or generic listing.

---

## Steps to Reproduce

1. Log in with any Vinted account (Account A).
2. Identify an `item_id` belonging to a different seller (Account B) via `vinted.com/items/{item_id}`.
3. While authenticated as Account A, send:
```http
GET /api/v2/users?per_page=100&search_text=&order_by_buyer_for_item={item_id_of_B}
```
4. Observe `200 OK` with a list of users tied to that item, data the UI only exposes to the item's actual owner.
![idor](/8.png)
![idor](/9.png)

6. **Negative control:** repeat the same request unauthenticated (incognito) → `401 invalid_authentication_token`, confirming the break is in **authorization**, not authentication.

---

## Evidence: Tested Across Multiple Items

| item_id | Actual seller | Tester owns it? | Result |
|---------|--------------|-----------------|--------|
| 10197071411 | Tester (ground truth) | Yes | 200 OK — first entry = `amethyst1984`, confirmed via native notification |
| 10197071234 | Seller of "Jordan Essentials Hoodie" | No | 200 OK — distinct user list |
| 1019071411 | willie2210 | No | 200 OK — distinct user list |
| 10233200877 | torisyphrit94 | No | 200 OK — user list including seller herself |
| (unauthenticated, any item_id) | n/a | n/a | 401 — invalid_authentication_token |

---

## Impact

- Any authenticated user (free account) can discover **who liked any item** on the platform, information not exposed anywhere in the product UI.
- **Confirmed and verifiable**: cross-referenced against Vinted's own native notification, not an assumption.
- **Scriptable at scale**: the parameter accepts arbitrary item IDs with no rate limiting observed. An attacker could iterate platform-wide to harvest who-liked-what data for the entire marketplace.
- Enables targeted harassment (identifying users who showed quiet interest without engaging publicly), competitive intelligence abuse (tracking which users are interested in competitors' items), and large-scale behavioral profiling.

---

## Recommendation

- Add an ownership check on the `order_by_buyer_for_item` parameter: verify that the `item_id` belongs to the authenticated user's account before returning results.
- Apply rate limiting to this endpoint to prevent bulk enumeration.
- Audit other `order_by_*` and `filter_by_*` parameters across the API surface for the same class of missing authorization checks.
