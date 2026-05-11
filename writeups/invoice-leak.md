---
layout: writeup
title: "Unauthenticated Invoice Access — Taxify (Bolt)"
description: "Customer invoices were publicly accessible via a predictable URL parameter — no login required. Swapping the UUID returned financial documents belonging to other users."
category: web
difficulty: easy
event: "Bug Bounty"
status: duplicate
outcome_note: "Marked as duplicate — another researcher had reported the same issue before me. Duplicate status confirms the vulnerability was valid and already known to the team."
date: 2025-06-05
---

## Overview

This one was found during a routine session with the Taxify (now Bolt) platform — back when the company was still transitioning its branding. The vulnerability itself is simple, but the data it exposed made it worth reporting: customer invoices containing names, trip history, and payment amounts were accessible to anyone with a link and a UUID to guess.

---

## Discovery

I was authenticated as a regular user and navigated to my invoice history. Each invoice had a "View" link that opened a PDF. I intercepted the request in Burp Suite and saw the URL structure:

```
GET /?s=3f7a1bc2-94d8-4e21-b3cc-XXXXXXXXXX
```

A UUID as the only parameter. No session cookie required. No token in the header. I opened the link in a private browser with no active session — the invoice loaded fine.

The application was treating knowledge of the UUID as proof of authorization. It wasn't.

---

## The Problem with "Security Through Obscurity"

UUIDs are 128-bit values — there are 2¹²² possible v4 UUIDs, which sounds enormous. The reasoning is usually: *"The ID is so hard to guess, we don't need auth."*

That reasoning has a few holes:

1. **Links get shared.** Invoices get forwarded to accountants, copied into expense reports, pasted into Slack. Once a URL is in the wild, it's in browser history, email threads, and server logs.
2. **Referrer headers.** If a user clicks a link from an invoice to an external site, the invoice URL may appear in the `Referer` header of that outbound request.
3. **Log exposure.** Any server, proxy, or analytics tool the URL passes through may log it — including third-party services.

The UUID was the only thing standing between anyone on the internet and a customer's financial data.

---

## Exploitation

The access control check was effectively nonexistent. The full flow:

1. Request my own invoice URL while authenticated
2. Copy the URL
3. Open it in an unauthenticated browser — invoice loads
4. Replace the UUID with one from a different invoice (obtained via enumeration, link sharing, or log leakage)
5. View another user's invoice with no credentials

 ![PII Leak through Unauthorised access to Invoice via IDOR]({{ '/assets/img/Zoho/Invoice.png' | relative_url }})

I didn't attempt large-scale enumeration — that wasn't necessary to prove the vulnerability. Confirming that the endpoint returned data without authentication, and that swapping identifiers returned different users' data, was sufficient.

---

## Data Exposed Per Invoice

- Customer full name
- Business/billing details
- Trip origin and destination
- Fare breakdown and payment amount
- Date and time of trip

For a driver or frequent business traveler, that's a meaningful privacy violation. For enterprise accounts, it could expose internal travel patterns or financial data.

---

## Impact

Beyond the immediate privacy issue, there are a few downstream risks worth flagging:

**GDPR exposure** — Taxify operated heavily in the EU. Unauthenticated access to customer financial records is exactly the kind of thing that draws a regulator's attention.

**Phishing surface** — An attacker with access to real trip data could craft highly convincing phishing emails: *"There was an issue with your Bolt invoice #XXXX from [correct date] for [correct amount]..."*

**Scalability** — While bruteforcing UUIDs in the traditional sense isn't feasible, any UUID that leaks through a referrer header, analytics script, or forwarded email becomes immediately exploitable.

---

## Fix Recommendations

The core fix is simple: **require authentication before serving invoice data, regardless of whether the requester knows the UUID.**

```python
# Pseudocode
@require_login
def get_invoice(uuid):
    invoice = Invoice.get(uuid)
    if invoice.user_id != current_user.id:
        return 403
    return send_pdf(invoice)
```

Additional hardening worth considering:

- **Signed, expiring URLs** — Generate invoice links that include an HMAC signature and a short TTL (e.g., 15 minutes). After that, the link is dead.
- **Access logs** — Track invoice views and alert on unusual access patterns (e.g., one IP requesting many different invoice UUIDs)
- **Strip URLs from referrer headers** — Use a `Referrer-Policy: no-referrer` header on pages that link to invoices
