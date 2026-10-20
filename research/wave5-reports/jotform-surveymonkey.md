Research complete. Both reports are below, sourced only from official provider documentation I actually read (Jotform: api.jotform.com/docs; SurveyMonkey: developer.surveymonkey.com portal pages). One methodological note: SurveyMonkey's API v3 reference (developer.surveymonkey.com/api/v3/) is a JavaScript-rendered page whose body text could not be extracted by my reader (only the site cookie notice was returned), so any detail that lives only in that reference is flagged under Unverified rather than asserted from third-party sources.

---

# REPORT 1 — JOTFORM

## 1. What the app/object is called, and the recommended flavor

**There is no OAuth app object, and Jotform has no OAuth 2.0 program for third-party API access in its official API documentation.** There is no client ID/client secret registration, no authorize/token endpoints, no redirect URI registration, and no scope system in the Jotform API docs. The connection object is a **per-user API Key**, created by each Jotform user in their own account (My Account → API section).

The docs state the model plainly: "To get started using Jotform API you need a valid API key," and the API "is written for Jotform users and 3rd party developers who would like to serve Jotform users" (Zapier is given as the example third-party user of the API).

**Recommended flavor for a third-party SaaS/AI agent:** per-user API key collection. Two documented variants:

- **(a) Ask each connecting user to create an API key in their Jotform account settings and paste it into your product** (manual key handoff). You then call the API as that user.
- **(b) JavaScript SDK login (the closest thing to a consent flow):** embed Jotform's JS SDK, call `JF.login()` to open a sign-in pop-up in an iframe; after sign-in "he will be asked to allow permissions to access to her account." Your code then calls `JF.getAPIKey()` to retrieve **that user's API key** and stores it server-side for future calls ("you need to get the user's API key and save it in your servers"). Note what is handed over: the user's full API key, not a scoped, expiring token.

Authentication of calls, per the docs, is by API key sent as a query parameter (`?apiKey=...`), as an HTTP header (`APIKEY: {myApiKey}`), or via the JS SDK (`JF.initialize({apiKey: ...})`).

## 2. Setup steps, in order

Developer side:

1. Build your integration against the REST API at `https://api.jotform.com` (EU accounts: `https://eu-api.jotform.com`; HIPAA: `https://hipaa-api.jotform.com`). *(Automatable — code only; no developer registration step exists.)*
2. Implement one of the connection flows in §1 (manual key paste, or JS SDK `JF.login()` + `JF.getAPIKey()`). *(Automatable, except the end user's sign-in/permission step in the SDK flow, which is human-only for the end user.)*
3. Store each connected user's API key server-side and send it on every call as header or query parameter. *(Automatable.)*

End-user side (per connected account):

4. User signs in to Jotform → My Account → API section → "Create a new API Key" (docs' Getting Started steps 1–3). *(Human-only — only the account owner can create keys; not needed if using the JS SDK login flow, which retrieves an existing key after sign-in.)*

There is no app registration, no developer portal enrollment, and no credential issuance step for the integrator at all.

## 3. Approval / review

- **No review or approval of any kind is required** before external users can connect. The official API docs describe no app submission, vetting, or listing process for API access, and no App Directory/marketplace listing for API integrations.
- **Pre-approval caps:** none (there is no approval state). The operative limits are per-API-key daily call limits tied to the *connected user's* Jotform plan (see §4).
- **Cost/fees:** no fee for API access is stated in the API docs; limits scale with the user's existing Jotform plan.
- **Stated review timeline:** not applicable — no review exists. (No verbatim quote available; the docs simply contain no review process.)

## 4. Time / effort (provider-stated figures only)

- **Access token lifetime:** not applicable — no tokens are issued. **API key expiry: the docs state no expiration for API keys** (keys persist until managed in the user's account settings; see §7).
- **Authorization code lifetime:** not applicable — no authorization-code flow exists.
- **Refresh token behavior:** not applicable — no refresh tokens.
- **Review waits:** none.
- **Stated usage limits (per API key, per day), verbatim from the docs' FAQ:** "API Keys are limited to: 1000 requests per day for the **starter** plan; 10000 requests per day for the **bronze** plan; 50000 requests per day for the **silver** plan; 100000 requests per day for the **gold** plan." Jotform Enterprise "has no limits to the number of daily API calls."
- **API capability note (docs FAQ):** "Jotform API v1 is mostly read only. You can get information about the Jotform user and forms. You can also download existing form data."

## 5. Blockers and nuances

- **The credential you receive is the user's full API key, not a scoped token.** There is no documented scope system for API keys, so a connected key should be treated as broad access to that account. The JS SDK "allow permissions" prompt does not translate into documented, enforceable permission subsets for the stored key.
- **No OAuth one-way doors, but a structural one:** because there is no OAuth, there is also no token refresh, no scoped consent, and no per-integration revocation handle exposed to you — your connection lives and dies with the user's key, which the user manages in their own account settings.
- **Region endpoints:** the docs publish separate base URLs for standard, EU, and HIPAA usage, so an integration must call the endpoint matching where the connected account's data resides rather than assuming one global host.
- **Plan-dependent ceilings follow the end user, not you:** a free/starter-plan user's key is capped at 1,000 calls/day regardless of your product's needs; heavy users must be on higher Jotform plans (or Enterprise) for the integration to scale.
- **JS SDK flow constraints:** `JF.login()` runs an iframe pop-up sign-in; after login the SDK auto-initializes with the key, and `JF.getAPIKey()` is how you persist access. This is a browser-mediated flow, so server-side/headless onboarding still falls back to manual key paste.
- JSONP is supported (`callback` parameter), per the docs — relevant only to legacy browser integrations.

## 6. Sources (official, actually read)

- https://api.jotform.com/docs/ — Jotform API documentation (Overview, Getting Started, Authentication incl. JS SDK `JF.login`/`JF.getAPIKey`, FAQ with daily API call limits). Primary source for everything above.
- http://developers.jotform.com/integrations/ — Jotform Developers integrations documentation (read via official search excerpt): its integration examples initialize with `JF.connect({ apiKey: apiKey, ... })`, consistent with the key-based model; this is Jotform's separate widgets/integrations platform, not an OAuth program.

## 7. Unverified (docs did not state)

- Whether API keys ever expire or are rotated automatically — the API docs state no key lifetime.
- Whether individual API keys can be created with restricted permissions (e.g., read-only vs. full access) — the docs page read describes key creation but does not describe per-key permission scoping.
- A documented revocation procedure/flow for a key held by a third-party app (docs describe key creation in account settings; no revocation endpoint or flow is documented).
- Any OAuth 2.0, OpenID Connect, client-credentials, or app-registration offering anywhere in Jotform's developer platform — none appears in the official API documentation read; if Jotform offers OAuth for some other surface (e.g., Enterprise SSO for logging into Jotform itself), it is not documented as a third-party API authorization mechanism in the sources above.
- Exact behavior of keys across the standard/EU/HIPAA endpoints (the docs list the endpoints but do not explicitly state whether a key is bound to one region).

---

# REPORT 2 — SURVEYMONKEY

## 1. What the app/object is called, and the recommended flavor

The object is an **App** created in the SurveyMonkey API Developer Portal (developer.surveymonkey.com → My Apps). At creation you choose an app type: **Public App** or **Private App**.

**Recommended flavor for a third-party SaaS/AI agent whose users connect their own accounts: a Public App.** Per SurveyMonkey's official comparison table, Public Apps are the only type that outside users can connect: "How many accounts can access the app — Unlimited" (vs. Private: "One SurveyMonkey account or one Team"), and for Public Apps "SurveyMonkey account needs to be an enduser of the app" with the end user purchasing their own SurveyMonkey account. The official Public App approval checklist also confirms the connection mechanism is OAuth sign-in by end users: "Your app should have at least five OAuth sign ins in the first three months."

**Alternative:** a Private App — for a single account or one Team (internal automation, an organization automating its own surveys/data export). It requires no approval, but it cannot be offered to arbitrary outside customers and is not eligible for the App Directory.

## 2. Setup steps, in order

1. Sign in to the SurveyMonkey Developer Portal and create an app in My Apps, choosing **Public App**. *(Human-only.)*
2. Configure the app and select the scopes it needs — the FAQ directs developers to the API Documentation for scopes: "Public app developers should understand which scopes they'll need. You can read more about scopes in our API Documentation." *(Human-only, in the portal.)*
3. Build the OAuth sign-in integration so end users can authorize your app (the approval checklist presupposes live OAuth sign-ins). *(Automatable — code; each end user's authorization is human-only for that user.)*
4. Ensure all communication is over HTTPS with a valid SSL certificate (checklist requirement). *(Automatable/infrastructure.)*
5. Create the app listing: "Visit the listings tab in the My Apps view to create an app listing and submit it for approval." First listing in English. *(Human-only.)*
6. Submit for SurveyMonkey approval and launch only after approval (see §3). *(Human-only.)*
7. Ongoing: reach/maintain "at least five OAuth sign ins in the first three months" and provide easily-accessible app support (checklist criteria). *(Human-only.)*

Private App path instead: create a Private App in My Apps → no approval step → use it against the one account/Team it belongs to.

## 3. Approval / review

- **Public Apps: yes, approval is required before launching.** Verbatim from the official comparison table: "**Requires approval before launching** — Public Apps: Yes; Private Apps: No." And: "Public apps can be listed on SurveyMonkey's App Directory after meeting certain criteria and receiving approval."
- **What triggers it:** launching a Public App / publishing it to the App Directory. You submit via the listings tab in My Apps.
- **App Approval Checklist (official, verbatim criteria):** "Your app should have unique value and be attracting new users regularly"; "Your app should NOT replicate functionality already offered by SurveyMonkey's survey tools"; "Your app's name should NOT be a generic term (i.e. 'Analytics'), include versioning (i.e. 'App V3.0'), and/or include the word 'SurveyMonkey'"; "Your app should have at least five OAuth sign ins in the first three months"; "You must state if users will need a free or paid account from your service to use your app"; "Your team must provide easily-accessible app support"; "All communication should be over HTTPS using a valid SSL certificate."
- **Discretion clause (verbatim):** "SurveyMonkey retains the right to reject an app based on criteria not listed above. Approval of your app by SurveyMonkey does not guarantee its compliance with the criteria above."
- **Pre-approval caps / limits:** Public Apps: "API call limits — 500,000/day**" ("\*\*Limits are subject to change"). Private Apps: "starting at 500 calls/day (contact sales)" to increase. Private Apps are hard-capped in reach: one SurveyMonkey account or one Team.
- **Cost/fees:** no listing or approval fee is stated on the official pages read. For Private Apps needing more calls, the FAQ says only: "If you're building a Private app and think you need a higher API limit you should reach out to our sales team" (no price stated).
- **Stated review timeline:** none — the official pages state no approval duration (see §7).

## 4. Time / effort (provider-stated figures only)

- **Access token lifetime:** not stated on the official portal pages read (the API v3 reference has the token documentation; its body could not be extracted — see §7). **No figure is asserted here.**
- **Authorization code lifetime:** not stated on the pages read. **No figure asserted.**
- **Refresh token behavior/expiry:** not stated on the pages read. **No behavior asserted.**
- **Review wait:** not stated.
- **Provider-stated figures that do appear:** Public App checklist activity threshold — "at least five OAuth sign ins in the first three months"; API call limits — Public "500,000/day", Private "starting at 500 calls/day".

## 5. Blockers and nuances

- **App-type choice is a one-way door in practice:** a Private App can never serve outside customers (one account or one Team, no App Directory eligibility); a Public App cannot launch at all until SurveyMonkey approves it. Pick wrong and you rebuild/re-submit.
- **Revocation is coarse and all-or-nothing.** FAQ, verbatim: "You can refresh your app's credential in the settings for your app in the developer portal. This will refresh your secret, revoking all access to your app." Rotating the app secret revokes access for *every* connected user at once — there is no per-user/per-token revocation described on the official pages read. Treat secret rotation as a break-glass, all-customers event.
- **Approval is discretionary beyond the published checklist** (see the verbatim discretion clause in §3), and approval "does not guarantee compliance" — listing risk persists after launch.
- **You may not clone SurveyMonkey or use its name:** apps replicating SurveyMonkey's own survey functionality are ineligible, and the app name may not be generic, contain versioning, or include "SurveyMonkey."
- **Economics differ by type:** for Private Apps the *developer* purchases the SurveyMonkey accounts; for Public Apps the *end user* purchases their own account — your customers need their own SurveyMonkey accounts (with a plan sufficient to be an end user of the app) to connect.
- **Call-limit asymmetry:** a private/internal deployment starts at 500 calls/day, so polling-heavy designs will hit the ceiling fast; the official private-app guidance recommends webhooks instead of polling, client-side caching, batching changes, and Bulk API endpoints to stay within limits.

## 6. Sources (official, actually read)

- https://developer.surveymonkey.com/ — SurveyMonkey API Developer Portal home.
- https://developer.surveymonkey.com/build-a-public-app/ — "Creating a Public App": Public/Private comparison table ("Requires approval before launching: Yes", unlimited accounts, 500,000/day) and the full App Approval Checklist, including the listings-tab submission step and the rejection-discretion note.
- https://developer.surveymonkey.com/build-a-private-app/ — "Creating a Private App": comparison table (one account or one Team, starting at 500 calls/day, no approval) and call-usage tips (webhooks over polling, caching, bulk endpoints).
- https://developer.surveymonkey.com/faq/ — Developer FAQ: scopes pointer to the API Documentation, purchasing more API calls via sales, and token revocation via credential refresh ("This will refresh your secret, revoking all access to your app").
- https://developer.surveymonkey.com/api/v3/#scopes — API v3 reference (attempted; see §7).

## 7. Unverified (docs did not state / could not be verified from official text in this session)

- **Access token lifetime, authorization code lifetime, and refresh-token behavior for the OAuth flow.** The API v3 reference at developer.surveymonkey.com/api/v3/ is where SurveyMonkey documents authentication and token expiration/revocation, but the page is JavaScript-rendered and my reader returned only the site's cookie notice, so I could not read those figures from official text. Third-party sources variously describe the flow (authorization-code exchange at api.surveymonkey.com/oauth/token, long-lived tokens, a 90-day token for some plans), but per the official-docs-only rule none of that is asserted here.
- The enumerated scope list and which scopes are required vs. optional (the FAQ points to the API Documentation's scopes section, same extraction problem).
- The exact OAuth endpoint URLs and request parameters as documented in the v3 reference.
- Public App approval duration / typical review wait — not stated on the portal pages read.
- Any fee for App Directory listing or approval — none stated on the pages read.
- Whether changing an app's scopes regenerates/invalidate existing tokens — not stated on the official pages read.
- The specific SurveyMonkey plan required for Private App developers/end users — the comparison table renders this as "Sign In to View Plan," so the plan name is not publicly stated on the page.