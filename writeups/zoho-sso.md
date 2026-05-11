---
layout: writeup
title: "Account Takeover via Invitation Link Abuse — Zoho Accounts SSO"
description: "A flaw in Zoho's invitation flow allowed anyone with a valid invite link to register as the target user without verifying email ownership, leading to full account impersonation across all Zoho services."
category: web
difficulty: medium
event: "Bug Bounty"
status: paid
bounty: 200
outcome_note: "Reported responsibly through Zoho's security disclosure program. Triaged as valid, fix deployed, bounty awarded."
date: 2025-11-03
---

## How It Started

This wasn’t something I was actively hunting for. It showed up during recon.

I was running a passive sweep on publicly exposed Zoho endpoints, collecting URLs from indexed sources. One of the URLs stood out immediately:

```
https://accounts.zoho.com/invitation/org/user/
<token>
```

Opening it led to a live invitation page for a user I didn’t know. No authentication required, no access control gate. Just a clean onboarding flow asking me to set a password.

![Burp Request]({{ '/assets/img/Zoho/Vulnerable_invite_link.png' | relative_url }})

That raised a simple question: *what actually proves I own this email address?*

---

## The Vulnerability

Zoho Accounts acts as the central identity provider for the entire Zoho ecosystem. It handles authentication and single sign-on across services like CRM, Desk, Books, and ManageEngine.

The invitation flow is supposed to onboard new users into an organization. Normally, that process should include some form of identity verification.

Here, it didn’t.

The application treated the invitation link itself as sufficient proof of identity. Once the link was opened, the only step required was setting a password. There was no email verification, no OTP, and no confirmation that the person completing the flow actually controlled the invited email address.

![Burp Request](/assets/img/Zoho/Unauthorised_userCreation.png)

In other words, the system trusted the link completely and never validated the user behind it.

---

## Exploitation

I decided to follow the flow exactly as intended.

1. Open the invitation link  
2. Observe the invited email address displayed on the page  
3. Click **Sign Up**  
4. Set a password of my choice  

That was it.

![Burp Request](/assets/img/Zoho/Unauthorised_access.png)
The account was created instantly, and I was automatically authenticated as the invited user. No verification step, no challenge, nothing in between.

The application redirected me into a Zoho service under that identity.

At this point, I wasn’t just inside an account — I was inside *someone else’s account*.

---

## Why This Matters

On the surface, this looks like a simple onboarding flaw. In reality, it’s much deeper.

Because Zoho Accounts is a centralized SSO system, authentication here is not scoped to a single application. It’s the entry point to everything tied to that organization.

Once the invitation is claimed, access propagates across services.

---

## Impact

| Risk | Detail |
|------|--------|
| Account takeover | Attacker can register as the invited user |
| Cross-service access | Access extends across all Zoho products via SSO |
| Identity spoofing | Actions performed under victim’s identity |
| Denial of service | Legitimate user cannot claim their account |

From an attacker’s perspective, this turns any exposed invitation link into a valid login.

And those links don’t have to be “stolen” in the traditional sense. If they’re indexed, leaked, or logged anywhere publicly, they become entry points.

---

## Root Cause

The system relied entirely on possession of the invitation link as proof of identity.

That assumption breaks the authentication model.

An invitation link is a delivery mechanism, not an identity verification factor. Without binding the process to the actual email owner, the flow becomes indistinguishable from account creation by an attacker.

---

## Fix

The fix required introducing a proper trust boundary into the flow.

After setting a password, the system now enforces email ownership verification through an OTP sent to the invited email address. Account activation cannot proceed without completing this step.

Additional protections observed after the fix:

- OTP-based verification tied to the invited email  
- Rate limiting on verification attempts  
- Proper enforcement of the activation flow  

With these controls in place, possession of the link alone is no longer enough.

---

## Disclosure Status

Reported responsibly. The issue was acknowledged and fixed by the Zoho security team.

![Burp Request](/assets/img/Zoho/zoho_reward.jpg)
---

## Closing Thoughts

Invitation flows are easy to overlook because they sit at the edge of authentication. They feel like onboarding logic, not security-critical infrastructure.

But in systems where invitations directly create authenticated identities, they *are* authentication.

In this case, a single missing verification step turned a convenience feature into an account takeover vector across an entire SaaS ecosystem.
