---
name: api-survey
description: Surveys payment processor API references to map a named capability (e.g. authorisation, capture, onboarding) to concrete endpoints, then summarizes endpoints, flows, request/response fields, auth schemes, and API design per provider. Use when comparing PSP APIs, mapping product functions to processor docs, or researching flows across Stripe, Adyen, Antom, Airwallex, Checkout.com, Worldpay, Worldline, or similar documentation sites.
---

# API survey

## Instructions

1. **Anchor the capability**  
   Take the user’s function name (e.g. `authorise`, `onboarding`) and expand it to **industry terms** used in docs: *authorization*, *payment*, *capture*, *session*, *customer*, *payment method*, *KYC*, *account*, *webhook*, etc. Search both the user’s wording and these synonyms in each doc site.

2. **Use the reference doc set**  
   Treat the following as the default catalog to walk unless the user supplies a different list. Open the relevant sections (often Payments, Sessions, Customers, Balances, Transfers, Webhooks, Onboarding / Accounts).

   | Provider | API / reference base |
   |----------|----------------------|
   | Stripe | https://docs.stripe.com/api |
   | Adyen | https://docs.adyen.com/api-explorer/ |
   | Antom | https://docs.antom.com/ac/ams/api |
   | Airwallex | https://www.airwallex.com/docs/api |
   | Checkout.com | https://api-reference.checkout.com/ |
   | Worldpay | https://docs.worldpay.com/access/products/payments/ |
   | Worldline | https://docs.connect.worldline-solutions.com/documentation/api/ |

3. **Per provider, collect**  
   - **Endpoints**: HTTP method + path (and resource name in their model).  
   - **Flows**: How calls chain (e.g. session → confirm → capture; onboarding → KYC → payout). Note **idempotency** and **webhook** touchpoints if documented.  
   - **Inputs and outputs**: Main path/query/body fields and response objects; link or name the canonical schema object (not every optional field unless the user asks).  
   - **Authorization**: Scheme (e.g. `Authorization: Bearer`, API key header/query, HMAC signature, client credentials); where keys live (dashboard, merchant vs platform); test vs live.  
   - **API design**: REST vs RPC-style paths, versioning (URL vs header), idempotency keys, pagination style, error envelope (HTTP codes + error body shape), webhooks vs polling.

4. **Use tools**  
   Fetch public doc pages or API reference as needed. If a site is heavy JS or blocked, say so and fall back to official OpenAPI links or PDFs when the docs expose them. Do **not** invent endpoints; mark **unknown** when the doc was not retrieved.

5. **Deliver the summary**  
   Use the output template below **once per provider** that was in scope. Keep each provider section parallel so the user can compare columns mentally.

## Output template

Repeat for each provider surveyed:

```markdown
### [Provider name]

- **Related endpoints**  
  - `[METHOD] /path` — one-line purpose.
- **Flow**  
  - Ordered steps and how this capability fits (including webhooks if applicable).
- **Inputs and outputs**  
  - Request: key fields / objects.  
  - Response: key fields / status values.
- **Authorization**  
  - Scheme and where credentials come from.
- **API design notes**  
  - Versioning, idempotency, errors, pagination, notable conventions.
```

## Examples

**User capability:** “card authorisation only, no capture yet”

- Map to: *Payment*, *Authorization*, *Auth-only*, *hold*, *pre-auth* per scheme.  
- For each doc set, list endpoints that create or consult an authorised payment without capture, and exclude pure capture/settlement unless needed for context.

**User capability:** “merchant onboarding”

- Map to: *onboarding*, *account*, *legal entity*, *KYB*, *merchant*, *Connect* (Stripe-style), *marketplace*.  
- Summarise account-creation and verification flows separately from payment acceptance APIs.

## Additional resources

- For a longer URL list or org-specific doc bases, extend the table in a project-local copy of this skill or add [reference.md](reference.md) beside `SKILL.md` with extra providers only.
