Research complete. Both reports below are based only on official documentation I read directly (Pipedrive: developers.pipedrive.com / pipedrive.readme.io docs hub and pipedrive.com Developer Agreement; Zoho: zoho.com/crm/developer/docs and zoho.com Marketplace developer help). Read-only; no accounts created, nothing submitted.

---

# PIPEDRIVE

## 1. What the OAuth app/object is called, and the recommended flavor

The object is an **"app"** created in the **Developer Hub** (inside a Pipedrive developer sandbox account, under *Settings > (company name) Developer Hub*). Saving a draft app issues the `client_id` and `client_secret` used for OAuth 2.0. There are two app types, chosen at creation:

- **Public app** — listed in the Pipedrive Marketplace with its own listing page; must pass Pipedrive's app approval process.
- **Private app** — not listed, no listing page, **no approval process**; once "live" it "can be shared with any user/company in Pipedrive via a direct, unlisted installation link."

**Recommended flavor for a third-party SaaS/AI agent whose users connect their own accounts:** a **Private app** if you distribute through your own product (users install via your unlisted link and run the standard OAuth flow; no review gate). Choose a **Public app** only if you want Marketplace discovery/listing, which triggers mandatory review. Both flavors use the same OAuth 2.0 authorization-code flow; scopes are fixed at registration in the "OAuth & access scopes" tab.

Alternatives/notes: Pipedrive also has a per-user **API token**, but the approval checklist explicitly penalizes it: "Your app does not collect, store, or unnecessarily use the API token of a user" — OAuth is the sanctioned path for multi-user apps.

## 2. Setup steps in order

1. **Request a developer sandbox account** at developers.pipedrive.com — required to access Developer Hub. *(Human-only: form request; Pipedrive may refuse or terminate sandbox access at its sole discretion per the Developer Agreement.)*
2. **Choose the app type (public or private) and save a draft app** in Developer Hub. Only two fields are required to save a draft: **app name** and **OAuth callback URL**; saving the draft generates your `client_id`/`client_secret`. *(Human-only, in the Developer Hub UI.)*
3. **Set OAuth & access scopes** in the registration form, mapped to the endpoints you'll call; complete the remaining registration tabs (general info, setup/installation, support & legal, review info) — required fields are validated when you send the app to review. *(Human-only.)*
4. **Implement the OAuth flow server-side** *(agent-automatable)*:
   - Redirect the user's browser to `GET https://oauth.pipedrive.com/oauth/authorize?client_id=…&redirect_uri=…&state=…` (the `/authorize` redirect step is only needed if you allow installation outside the Marketplace; Marketplace installs start from the listing).
   - User sees the consent dialog and clicks "Allow and install" or "Cancel"; Pipedrive redirects to your callback URL with `code` (or `error=user_denied`).
   - Exchange the code: `POST https://oauth.pipedrive.com/oauth/token` (form-urlencoded, HTTP Basic auth with `client_id:client_secret`, `grant_type=authorization_code`, `code`, `redirect_uri`). Response returns `access_token`, `refresh_token`, `scope`, `expires_in`, and `api_domain`.
   - Call the API as `GET https://{COMPANYDOMAIN}.pipedrive.com/api/v1/…` with `Authorization: Bearer <access_token>`.
   - Refresh via the same token endpoint with `grant_type=refresh_token`.
5. **Build the mandatory installation/uninstallation flows** and test with the "Install & test" button in Developer Hub. *(Mixed: code is agent-automatable; testing requires the human's sandbox.)*
6. **Public apps only: click "Send to review"**, supplying a demo video explaining permission/scope usage and key functionality, fully functional test accounts for Pipedrive's team, and an available contact person. *(Human-only.)*
7. **After approval, publish manually**: approved apps are "unlisted from the Marketplace by default"; go to "…" next to the app in Developer Hub and click "Publish." *(Human-only.)*

## 3. Approval / review

- **Private apps: no review at all.** They skip the approval process entirely and can be shared with any Pipedrive company via the unlisted install link once live. This is the zero-gate route for a SaaS that brings its own users.
- **Public apps: review is mandatory before Marketplace listing.** "Each new public app submitted to the Pipedrive Marketplace has to pass our approval process." Reviewers test the app end to end (installation flows and actual functionality). Approval checklist requirements include: OAuth 2.0 as primary auth, refreshing the access token after its 60-minute lifetime, requesting only necessary scopes (users can only **accept or deny all scopes** — no partial consent), polished install/uninstall flows, Terms of Service and Privacy Policy pages, a pricing page link, 3–5 listing images, demo video, and working test accounts for the review team. Material changes to an approved app must be resubmitted for approval (per the Developer Agreement).
- **Stated review timeline (verbatim banner on the approval-process page):** "Due to a surge in demand and ongoing vacations, public app reviews may take up to **21 business days**." No standard/baseline review duration is stated elsewhere in the docs I read.
- **Pre-approval caps:** A **draft private app** "can be shared with users in the same Pipedrive company" only; it must be live to share cross-company. The **developer sandbox is capped at 5 authorized developer users** ("The maximum number of Authorized Developer Users that can access the sandbox is five (5)" — Developer Agreement). I found **no stated install/user cap** for draft public apps beyond testing in your own sandbox — flagged as unverified below.
- **Cost:** No listing fee or review fee is stated anywhere in the docs or Developer Agreement sections I read — unverified (see §7).

## 4. Time / effort (provider-stated figures only)

- Public app review: "up to **21 business days**" (current banner; docs give no other figure).
- Authorization code lifetime: **5 minutes** ("the `authorization_code` expires in 5 minutes").
- Access token lifetime: **60 minutes** ("access_token expires after 60 minutes").
- Refresh token: "will expire if it isn't used in **60 days**. Each time `refresh_token` is used, its expiry date is reset back to 60 days" — the same refresh token is re-issued on refresh with the window extended.
- No provider-stated setup-duration estimates for registration or implementation.

## 5. Blockers and nuances

- **App type is a one-way door:** "Do pick your app type carefully, as it cannot be changed later on Developer Hub." Private ↔ public cannot be converted; you'd create a new app (new credentials, users re-install).
- **Exactly one callback URL per app:** "we allow only one callback URL per app." Plan tenant/environment routing accordingly.
- **All-or-nothing consent:** users "accept or deny all scopes"; there is no granular consent, so over-scoping directly hurts conversion, and reviewers check that you request only necessary scopes.
- **Every user installs individually, per company:** "Every user in a company needs to install or set up their own apps and integrations," and users with multiple companies must install per company; the access token is bound to the granting user **and** the company they were logged into when authorizing.
- **Refresh-token inactivity cliff:** if the refresh token isn't used within 60 days it expires, and "the whole installation process needs to be started from the beginning" (user must reinstall).
- **Deleting your app uninstalls it from all existing customers** and removes it from the Marketplace.
- Store access tokens with room to grow: Pipedrive recommends "a minimum of varchar (768) for storing the tokens" as token length fluctuates.
- Scope or name changes on an approved app "may have to go through the app approval process again."
- Token exchange must happen after the user has created/logged into an account in *your* app (docs warn about the ordering during install).

## 6. Sources (official, actually read)

- https://pipedrive.readme.io/docs/marketplace-oauth-api (OAuth 2.0 overview)
- https://pipedrive.readme.io/docs/marketplace-oauth-authorization (authorize/token/refresh, lifetimes)
- https://pipedrive.readme.io/docs/developer-hub (draft apps, statuses, private vs public)
- https://pipedrive.readme.io/docs/marketplace-creating-a-proper-app (app types, characteristics)
- https://pipedrive.readme.io/docs/marketplace-registering-the-app (registration form, one callback URL, scopes)
- https://pipedrive.readme.io/docs/marketplace-app-approval-process (checklist, "up to 21 business days" banner, publish step)
- https://www.pipedrive.com/en/developer-agreement (sandbox 5-user max, Marketplace approval for material changes)

## 7. Unverified

- Any fee for Marketplace listing/review or revenue share — not stated in the docs I read.
- A baseline (non-surge) public-app review duration — only the 21-business-day banner figure is stated.
- Whether draft public apps have a numeric install cap before approval — not stated; docs only describe installing/testing in your own sandbox.
- Sandbox-account approval turnaround time — not stated.
- Whether a private app has any install ceiling — no cap is stated in the docs read.

---

# ZOHO CRM

## 1. What the OAuth app/object is called, and the recommended flavor

The object is a **client** (the token docs also call it a "connected app") registered in the **Zoho API Console / Developer Console** (api-console.zoho.com), which issues a **Client ID** and **Client Secret**. Zoho Accounts is the OAuth 2.0 server for all Zoho services, including CRM. Client types:

- **Server-based Application** — authorization code flow, secret kept server-side. Zoho's CRM docs describe web-based apps as "chosen when your application is used by multiple users and requires user intervention during authorization."
- **Client-based (JavaScript) Application** — implicit / authorization code with PKCE, browser-only.
- **Mobile-based Application** — authorization code with PKCE.
- **Non-browser Application** — device authorization flow.
- **Self Client** — pre-generated authorization code for your own account's back-end jobs/scripts; no redirect URL or consent redirect flow; **own-account only**, so it is *not* a third-party flavor.

**Recommended flavor for a third-party SaaS/AI agent:** **Server-based Application** with `access_type=offline` (required to receive a refresh token), requesting granular `ZohoCRM.*` scopes at authorization time (e.g. `ZohoCRM.modules.ALL`, `ZohoCRM.users.READ`). CRM has a real, open OAuth program — any user can register a client self-service and any Zoho user can authorize it; there is no OAuth-client approval step in the CRM developer docs.

## 2. Setup steps in order

1. **Register the client** in the Zoho API Console: choose **Server-based Application**, enter Client Name, Homepage URL, and Authorized Redirect URI(s), click Create; receive Client ID and Client Secret. *(Human-only: console UI, requires a Zoho account.)*
2. **Build the authorization URL and redirect the user** *(agent-automatable)*: `https://accounts.zoho.com/oauth/v2/auth?scope={scopes}&client_id={id}&response_type=code&access_type=offline&redirect_uri={uri}`. The user logs in, picks the **organization** (Production/Sandbox/Developer environment) the grant applies to if they have several, and Accepts or Rejects (`error=access_denied` on reject). *(Consent itself is human-only, per user and per organization.)*
3. **Exchange the grant token immediately** *(agent-automatable)*: POST to `{accounts-server}/oauth/v2/token` with `grant_type=authorization_code`, `code`, `client_id`, `client_secret`, `redirect_uri`. You must use the **domain-specific accounts URL** returned in the redirect (`accounts-server` parameter, e.g. `https://accounts.zoho.eu` for EU users). Response: `access_token`, `refresh_token`, `api_domain`, `token_type: Bearer`, `expires_in: 3600`.
4. **Call the CRM API** at the returned `api_domain` with header `Authorization: Zoho-oauthtoken {access_token}`. *(Agent-automatable.)*
5. **Refresh** with `grant_type=refresh_token` against the same DC's token endpoint before the 1-hour access-token expiry; reuse valid access tokens rather than minting new ones (throttles below). *(Agent-automatable.)*
6. **Optional — enable Multi-DC** for the client in the console (Settings → enable Multi DC) if your users span data centers, noting each DC has its **own unique client secret**. *(Human-only console toggle; secrets are human-managed.)*
7. **Optional, separate track — Zoho Marketplace listing:** if you also build a CRM **extension** (Sigma/Zoho Developer Console), you can publish it privately via an install URL, or submit it for Marketplace review (see §3). This is not required to use OAuth from your own SaaS. *(Human-only.)*

## 3. Approval / review

- **OAuth clients: no review or approval is described anywhere in the Zoho CRM developer docs** — registration is instant self-service, and external users in other organizations can authorize your client. I found no pre-approval user/install cap for OAuth clients (token-level caps in §5 are the real constraints). Flagged as a docs-absence finding, not an affirmative policy statement.
- **Zoho Marketplace (extensions only):** publishing an extension privately needs no review (you get a shareable install URL). Listing publicly requires Zoho's review: submit via the "List your extension in the Zoho Marketplace" checkbox + submission form; status shows "Waiting for approval." **Stated timeline (Zoho Marketplace help): "The review process usually takes 24-48 hours"**; the developer FAQ likewise says "It would take 48 hours for reviewing." One Zoho community post about *telephony* extensions cites 3 weeks to 1 month — a different extension category and a community (non-docs) source, so treat it as unverified.
- **Cost:** No listing or review fee is stated in the docs I read. (The Marketplace FAQ mentions developer payouts are processed monthly once the sum is at least $100 — a payout threshold, not a fee.)

## 4. Time / effort (provider-stated figures only)

- Access token: "Each access token is valid for **one hour**" (`expires_in: 3600`).
- Grant token (authorization code): "a one-time use token and valid for **three minutes, by default**" (Token Validity page). See the inconsistency flagged in §5.
- Refresh token: "Refresh tokens do not expire until a user revokes them."
- Throttles (Token Validity page): max **10 grant tokens per client ID in 10 minutes**; max **10 access tokens from a refresh token in 10 minutes**; max **15 active access tokens per refresh token** (the 16th invalidates the oldest); max **20 refresh tokens per user** (generating the 21st invalidates the first created, "irrespective of whether the first refresh token is in use or not").
- Marketplace extension review: "usually takes **24-48 hours**" (if you take that optional track).
- No provider-stated setup/implementation duration estimates.

## 5. Blockers and nuances

- **The 20-refresh-token trap:** every re-consent mints a *new* refresh token; once a user has 20, the **oldest is silently invalidated** — an integration that prompts re-authorization repeatedly can kill its own (or another session's) stored token.
- **Tokens are organization- and environment-specific:** a grant is for one chosen org in one environment (Production/Sandbox/Developer); tokens from one cannot be used against another.
- **Data-center fragmentation:** Zoho runs separate DCs (US .com, EU .eu, IN .in, AU .com.au, JP .jp, CN .com.cn, SA .sa, CA zohocloud.ca). Registration, grant, and token calls must all happen in the user's DC (read `location`/`accounts-server` from the redirect) unless you enable Multi-DC — and **each DC holds a unique client secret**, so a DC-mismatched secret yields `invalid_client`.
- **Grant-token lifetime is stated inconsistently across Zoho's own CRM docs:** 3 minutes "by default" (Token Validity), "valid only for two minutes" (Authorization Request page note), and "valid only for one minute in the redirection-based flow" (Access & Refresh Tokens error page). Build for the shortest (exchange immediately).
- **Self Client is a dead end for multi-user products** — it generates codes only for your own account/orgs from inside the console; don't mistake it for the third-party path.
- `access_type=online` returns **no refresh token**; forgetting `access_type=offline` means re-consent every hour.
- Scopes are requested per authorization (not fixed at registration) and are granular (`ZohoCRM.modules.<module>.READ/CREATE/UPDATE/DELETE/ALL`, `ZohoCRM.users.*`, `ZohoCRM.settings.*`, `ZohoCRM.org.*`); the token is usable "only for the operations defined in the scope." Dummy homepage/redirect values are permitted at registration if you don't have a domain yet (per SDK registration docs), but the runtime `redirect_uri` must exactly match the registered one (`ERROR_invalid_redirect_uri` otherwise).
- API calls use the nonstandard header scheme `Zoho-oauthtoken`, not `Bearer`, despite `token_type: Bearer` in the token response.
- CRM API usage credits/rate limits apply per org plan, but the CRM OAuth docs I read did not state figures — unverified here.

## 6. Sources (official, actually read)

- https://www.zoho.com/crm/developer/docs/api/v2/auth-request.html (authorization request, org selection, self-client)
- https://www.zoho.com/crm/developer/docs/api/access-refresh.html (token exchange, DC domains, `Zoho-oauthtoken` header)
- https://www.zoho.com/crm/developer/docs/api/v6/token-validity.html (grant/access/refresh token lifetimes and caps)
- https://www.zoho.com/crm/developer/docs/javascript-sdk/v2/clientapp.html (client registration, client types; also the identical C#/Ruby SDK registration pages)
- https://www.zoho.com/developer/help/marketplace/publish-extension.html and https://www.zoho.com/developer/help/extensions/publish-extension.html (private publish vs Marketplace review, "24-48 hours")
- https://www.zoho.com/developer/help/extensions/faqs.html (review "48 hours", payout threshold)
- Zoho OAuth client-type/flow comparison (Server-based, Client-based, Mobile, Non-browser, Self Client) via Zoho's OAuth 2.0 user guide on zoho.com

## 7. Unverified

- Whether Zoho imposes any review, verification, or cap on OAuth clients used across many external organizations — no such step appears in the CRM developer docs read; absence of documentation, not a stated guarantee.
- Any fee for client registration or Marketplace listing — none stated.
- Which grant-token lifetime (1, 2, or 3 minutes) applies in a given flow — Zoho's own pages conflict.
- Exact paid-plan/API-credit limits affecting connected orgs — not in the OAuth docs read.
- The 3-weeks-to-1-month review figure for telephony extensions comes from a Zoho community post, not official docs.
- Whether Self Client grant-token duration options (chosen from a dropdown at generation) have a stated maximum — the docs say you choose the duration but the pages I read don't enumerate the options.