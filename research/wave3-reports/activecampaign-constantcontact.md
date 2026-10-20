Research complete. Below are the two structured reports, based only on each provider's official documentation (ActiveCampaign: developers.activecampaign.com plus its official Help Center; Constant Contact: developer.constantcontact.com). Read-only research; no accounts created, nothing submitted.

---

# ACTIVECAMPAIGN

**Headline verdict: ActiveCampaign has NO OAuth program for third-party apps that want to act on users' ActiveCampaign data.** There is no OAuth app registration, no ActiveCampaign authorization server, no client ID/secret for the ActiveCampaign API, and no consent flow in the official developer documentation. The word "OAuth2" appears extensively in ActiveCampaign's docs, but in the *opposite direction* (see §1). What exists for accessing ActiveCampaign data is (a) per-user **API key + account API URL** credentials for the API v3, and (b) **App Studio / CX Apps**, an in-product app platform where OAuth2 is something ActiveCampaign uses to connect *outbound* to *your* service.

## 1. What the "app" is called, and the recommended flavor

- **API-access direction (your SaaS connecting to users' ActiveCampaign accounts):** there is no app object at all. The credential is a per-user **API URL and API Key** pair, found in each account under Settings → Developer. ActiveCampaign's own Help Center states: "The API URL and Key can be considered your username and password, which third-party apps can use to access your ActiveCampaign account," and "Each user in your ActiveCampaign account has their own API URL and Key." API v3 requests are authenticated with that key (the API reference documents it as the `Api-Token` header) against the account-specific base URL.
- **App Studio direction:** the app object is a **CX App**, built in **App Studio** and defined by an **App Configuration** JSON file. Its `auth` section is defined, verbatim, as: "Defines how ActiveCampaign authenticates with the integrating system's API." Supported auth types are OAuth2, Basic Auth, Token Auth, and Session Auth — all describing how *ActiveCampaign* logs in to *your* API. In the OAuth2 config, *you* supply the external service's `authorization_base_url`, `client_id`, `client_secret`, `scopes`, `token_url`, and optional `refresh_url`; the redirect URL you register with that external authorization server is ActiveCampaign's own callback (`https://integration-layer-siphon.cluster.app-us1.com/auth/callback`). OAuth2 here makes your service the OAuth provider and ActiveCampaign the client — it does not give your SaaS delegated access to a customer's ActiveCampaign account.
- **Recommended flavor for a third-party AI agent/SaaS whose users connect their own ActiveCampaign accounts:** there is no OAuth flavor to pick. The realistic paths are (1) each customer copies their API URL and API Key from Settings → Developer and pastes them into your product, or (2) you build a CX App in App Studio if you want a presence inside ActiveCampaign's App Marketplace and your integration fits the CX App model (ActiveCampaign calling your API, using whatever auth — including OAuth2 — your API requires).

## 2. Setup steps in order

**Path A — API key access to customers' accounts (the "connect an account" substitute):**
1. Each customer signs in to ActiveCampaign → Settings (gear icon) → **Developer** and copies their **API URL** and **API Key**. *(Human only — the customer must do this inside their own account; there is no developer portal or consent screen.)*
2. The customer pastes the API URL + Key into your product; you store them and call that account's API v3 base URL with the key on each request (documented in the API reference as the `Api-Token` header). *(Agent-automatable: all the API plumbing.)*

**Path B — CX App in App Studio:**
1. Request a **developer sandbox account** (a scaled-down production account for testing). *(Human.)*
2. In the sandbox account: left nav **Apps → App Studio → Build a New App**; complete the form (app name, email, questions), agree to the Terms and Conditions, **Submit**. *(Human.)*
3. Write the **App Configuration** JSON (`$version`, `auth`, `api`, `data_intake`, `workflows`, `objects`). In `auth`, declare how ActiveCampaign will authenticate to *your* API (OAuth2 / Basic / Token / Session). For OAuth2 you enter your own service's authorization/token URLs and the client ID/secret *your service issued to the integration*. *(Agent-automatable — it is code/config; validation errors surface in the Configuration Toolbox and all must be resolved before testing or submitting.)*
4. Click **Test app in sandbox** and debug using the real-time logs in the editor. *(Human-triggered; agent can iterate on config.)*
5. Complete **App Details**: Description (name, description, homepage URL), Media (logo 200×200px, header image 980×268px), Metadata (intended audience public or private, category), and Developer Info (testing credentials so the review team can test end to end). *(Human supplies assets/credentials.)*
6. For a public release, prepare the **marketplace listing** (via the listing form) and a **help content guide** for users. *(Human.)*
7. **Submit for review** from App Studio Home or the Configuration Toolbox. *(Human.)*
8. After approval, click **Release App** from App Studio Home; choose a private release (your account only) or Marketplace release. *(Human.)*

## 3. Approval / review

- **Path A (API keys):** no review, no approval, no listing. A customer hands over their key and it works. There is nothing to submit.
- **Path B (CX Apps):** review is mandatory before release. Per the official Getting Started guide, App Studio lets developers "submit code for review, and release an approved app directly into the ecosystem." The Go Live Guide states: "Once your app has been submitted for review, our team will conduct a technical review of your app's config file. In addition to the technical review, our team will also test your app's functionality from end-to-end." If engineers find issues, they contact you to fix them. You can withdraw a submission in App Studio Home and re-submit. Review progress is visible via a progress bar on the app version in My Apps.
- **Pre-approval state:** before approval the app is testable only in your own sandbox account. Release options after approval: private ("accessible to you and any users of your ActiveCampaign account") or Marketplace ("accessible to all ActiveCampaign customers"). "You will receive a notification when the review is complete and approved for release."
- **Timeline:** no review duration is stated anywhere in the docs read. **Cost:** no fee is stated anywhere in the docs read (see §7).

## 4. Time / effort

- Provider-stated figures only: the official docs read state **no durations at all** — no review timeline, no setup-time estimates, and no token lifetimes (there are no ActiveCampaign-issued OAuth tokens whose lifetime could be stated). For Path B's outbound OAuth2, token lifetimes are whatever the *external* service (yours) issues.
- Concrete stated requirements that drive effort: logo 200×200px and header image 980×268px for the listing; the config must pass validation before the "Test app in sandbox" or submit actions are enabled; the review team needs working test credentials stored under App Details → Developer Info.

## 5. Blockers and nuances

- **The core blocker is the missing OAuth flow itself** (for the SaaS direction): no delegated consent, no scopes, no refresh tokens, no revocation short of resetting the key. Your onboarding is "ask the customer to find and paste their API URL and Key," and the key is effectively a full account password — ActiveCampaign itself says it "can be considered your username and password." There is no least-privilege option: the key is unscoped.
- **Per-user key fragility:** "If integrations are set up with a user's API credentials, deleting the user will cause integrations to break. Since there is no account default API, you must update affected integrations with another user's API credentials." If your champion leaves the customer's company and their user is deleted, your connection dies.
- **Key reset is all-or-nothing:** a compromised key is reset on the Developer Settings page ("Reset API Key"), and "you must update all 3rd party applications with your new API information" — one reset breaks every integration using that user's key simultaneously.
- **Per-account base URL:** every account has its own API URL (copied from its Developer settings); you must store and use each customer's URL rather than assuming a single global API host.
- **Direction confusion is the big trap in the docs:** third-party blogs (and a skim of the App Studio docs) make it sound like you "create an app and get OAuth client credentials" for ActiveCampaign. In the official docs, those client credentials are issued by the *external* service to ActiveCampaign, not by ActiveCampaign to you.
- **CX App privacy note:** "ActiveCampaign respects the privacy of its customers, and does not provide the Personally Identifiable Information (PII) of users who have installed your application. Any additional install tracking should be done through your API endpoint." Install counts are shown in the Version Management tab.
- **CX App redirect registration:** if your service is the OAuth provider for a CX App, the redirect URL to allow-list is ActiveCampaign's fixed callback on its own domain (see §1), which some providers' verification processes treat as a friction point.

## 6. Sources (official, actually read)

- https://developers.activecampaign.com/docs/configuration-specification-v2 (App Configuration sections; "auth: Defines how ActiveCampaign authenticates with the integrating system's API")
- http://developers.activecampaign.com/docs/configuration-walkthrough-1 (auth types incl. OAuth2 example config pointing at `api.example.com` endpoints; data intake/workflows model)
- https://developers.activecampaign.com/docs/auth (OAuth2 config properties; "the location where we will send the user to authorize access to your integration for their account on the service provider"; the fixed ActiveCampaign callback redirect URL)
- https://developers.activecampaign.com/docs/getting-started-and-guides (sandbox account, App Studio flow, App Details specs, submit-for-review, private vs Marketplace release)
- https://developers.activecampaign.com/docs/cx-app-go-live-guide (technical review + end-to-end testing, marketplace listing and help content, test credentials in Developer Info)
- https://developers.activecampaign.com/reference (API v3 Overview — REST/HTTP/JSON API; no OAuth authorization flow documented)
- https://help.activecampaign.com/hc/en-us/articles/207317590-Getting-started-with-the-API (official Help Center: API URL + Key model, per-user keys, user-deletion breakage, key reset) — note this is ActiveCampaign's official help domain rather than the developer domain.

## 7. Unverified

- Any CX App review timeline, SLA, or approval-rate information — not stated in the docs read.
- Any cost for App Studio, sandbox accounts, or Marketplace listing — not stated in the docs read.
- Any numeric caps (installs, users, apps per account) — not stated in the docs read.
- API v3 rate limits — not stated in the pages read in this pass.
- The API reference's authentication section text (the `Api-Token` header mechanics) could not be surfaced in the fetched Overview extract in this pass; the key-based model itself is confirmed by the official Help Center article above.
- Whether any partner-only or unreleased OAuth program for ActiveCampaign-as-provider exists outside the public documentation — nothing in the official docs read describes one; treat the documented surface as the whole program.

---

# CONSTANT CONTACT

**Headline finding: Constant Contact has a real, standard OAuth 2.0 program for third-party apps (V3 API), but every new application starts in a private, single-user sandbox state — you cannot onboard real external users until you phone Constant Contact support and complete an approval process whose criteria and timeline the docs do not publish.**

## 1. What the OAuth app is called, and the recommended flavor

- The object is an **application** registered in the **V3 API Developer Portal** under **My Applications**. Registering yields an **API Key** (this is the `client_id`) and, on demand, a **client secret** ("Generate Client Secret").
- Four flows are selectable at app creation: **Authorization Code Flow** (bundled with Implicit in one setting), **PKCE Flow**, and **Device Authorization Flow**. A second creation-time choice is the refresh-token method: **Rotating Refresh Tokens** ("recommended for use with most applications because they are most secure") or **Long Lived Refresh Tokens**.
- **Recommended flavor for a third-party AI agent/SaaS whose users connect their own accounts:** the **Authorization Code Flow with Rotating Refresh Tokens** and the `offline_access` scope. The docs position it for "traditional web applications (your application runs on a web server…) and is able to safely store the client secret" and call it "the most secure OAuth2 flow" because the access token passes directly to your server, never the browser.
- Alternatives: **PKCE Flow** for public clients (native/single-page apps that cannot safely store a secret; refresh tokens supported); **Device Flow** for input-constrained/browserless apps (refresh tokens supported); **Implicit Flow** for public clients — legacy only: the overview table says "Use only if your application already uses the Implicit flow or does not support Web Crypto," and refresh tokens are **not** supported there.
- Scopes (V3): `account_read`, `account_update`, `contact_data`, `campaign_data`, plus `offline_access` (required to receive a refresh token). Consent is all-or-nothing: "The account owner or authorized user must either grant access to all of your scopes or decline access to your application."

## 2. Setup steps in order

1. **Sign up for a V3 developer account** in the Developer Portal (My Applications → Sign up; email, password, name, organization, phone). A trial Constant Contact account is created for development/testing. *(Human only.)*
2. **Register the application:** My Applications → **New Application** → enter the app name (shown to users on the grant-access prompt) → choose the OAuth2 setting (Authorization Code/Implicit, PKCE, or Device) → choose the refresh-token method (Rotating recommended, or Long Lived) → **Create**. *(Human — both choices are made at creation; see §5.)*
3. **Get credentials:** click **edit** on the app; the **API Key (client_id)** shows in the Details view. If your flow needs a secret, click **Generate Client Secret** and copy it immediately — "The secret only appears once, when you first request to generate it. Regenerating the secret would require you to update your application." Click **Save**. *(Human.)*
4. **Register your redirect URI(s)** on the application (multiple are supported). *(Human.)*
5. **Build the authorization request:** GET `https://authz.constantcontact.com/oauth2/default/v1/authorize` with `client_id`, `redirect_uri`, `response_type=code`, `state` (required), and optional `scope` (space-delimited), `nonce`, `prompt=login`. *(Agent-automatable — code.)*
6. **User authorizes:** Constant Contact shows the login and the **Permission Request** screen listing your scopes; on grant it redirects to your `redirect_uri` with `code` and `state` (and optional `scope`) as query parameters. *(End user, human only.)*
7. **Exchange the code:** POST `https://authz.constantcontact.com/oauth2/default/v1/token` with `code`, `redirect_uri`, `grant_type=authorization_code`, Basic auth header `base64(client_id:client_secret)`, content type `application/x-www-form-urlencoded`. Response: `access_token` (JWT), `refresh_token` (only if `offline_access` was granted), `id_token`, `scope`, `expires_in`. *(Agent-automatable.)*
8. **Validate and use tokens:** decode/validate the access token's signature and claims against Constant Contact's JWKS (expected issuer `https://identity.constantcontact.com/oauth2/aus1lm3ry9mF7x2Ja0h8`, audience `https://api.cc.email/v3`, `cid` must equal your API key; `platform_user_id` uniquely identifies the user), then call the V3 API with `Authorization: Bearer {access_token}`. *(Agent-automatable.)*
9. **Refresh:** POST the same token endpoint with `grant_type=refresh_token` and the current refresh token (Basic auth required) to get a new access token (and, with rotating tokens, a new refresh token). *(Agent-automatable.)*
10. **Before onboarding real users:** complete the approval process in §3. *(Human — phone call to support.)*

## 3. Approval / review

- **Yes — approval is required before real external users can connect.** Both the Quick Start Guide and the OAuth2 Overview state it identically: "Applications you create are initially limited to private access only. You can only grant access to new applications using the Constant Contact user that created the application. In order to allow all Constant Contact users to grant access to your application, contact support by calling 866-289-2101 to begin the approval process."
- **Pre-approval cap:** effectively one user — the Constant Contact user who created the application is the only account that can grant it access. That is your entire test population until approval.
- **Process:** the documented process is a phone call to support to *begin* it. The docs read publish **no criteria, no checklist, no submission portal, and no timeline** for the approval.
- **Marketplace/listing:** no self-serve marketplace listing process is described in the developer documentation read (see §7).
- **Cost:** no fee is stated anywhere in the docs read (see §7).

## 4. Time / effort

Provider-stated figures only:

- **Access tokens:** "expire after 1440 minutes (86,400 seconds). They have a maximum life of 24 hours" (Authorization Code, PKCE, and Device flows). Implicit-flow tokens: "Expires after 480 minutes (28,800 seconds) with a maximum lifetime of 8 hours."
- **Refresh tokens:** "Refresh tokens are 42 characters long and expire after 180 days, if never used" — supported in Authorization Code, PKCE, and Device flows; not in Implicit.
- **Authorization codes:** "Authorization codes expire after 5 minutes (300 seconds)" and can be used once.
- **Approval wait:** not stated (the process starts with a support call; no duration given).
- **Setup durations:** none stated; app creation/registration is a single portal session in the Quick Start.
- Doc inconsistency worth flagging: the Authorization Code Flow page's example token response shows `expires_in: 28800` (8 hours) while its own text and the OAuth2 Overview state 86,400 seconds / 24 hours for that flow. Treat 24 hours as the documented maximum lifetime and code defensively to the `expires_in` you actually receive.

## 5. Blockers and nuances

- **The private-access gate is the launch blocker:** everything works in testing with your own user, which makes it easy to build the whole integration before discovering that no customer can authorize it until the support-driven approval completes, on an unpublished timeline.
- **Flow + refresh-method choices are creation-time settings:** you pick the OAuth2 setting and Rotating vs Long Lived refresh tokens in the New Application dialog. The docs recommend Rotating "for use with most applications because they are most secure." With rotating tokens, each refresh returns a new refresh token, so you must persist the replacement every time or strand the connection. If you choose wrong for your client type (e.g., Authorization Code for an app that cannot hold a secret), expect to re-register/reconfigure.
- **No `offline_access`, no refresh token:** it is a scope like any other and easy to omit; without it users must re-authenticate and reauthorize when the 24-hour access token expires.
- **Token endpoint is rate-limited:** "Constant Contact rate limits the Token endpoint. A 429 response will likely be returned if you attempt to refresh an access token before every V3 API request. Constant Contact recommends that you only send a refresh token request to get a new access token if your existing access token is expired or about to expire." The docs suggest treating a token as expired when its expiry is within 5 minutes (300 seconds).
- **Tokens are large JWTs with mandatory validation work:** access tokens are "very long in length (1,000–1,200 characters)" and the flow docs instruct you to validate signature and claims (issuer, audience, `cid` = your API key, `platform_user_id`) for every new access token — budget the JWKS fetch/cache and validation code.
- **Client secret is a one-time reveal:** shown only when generated; regeneration forces an app update everywhere the old secret is deployed.
- **Redirect URI rules:** "Wildcards are not supported except at the domain root level." Registration does not validate the URI ("During the application registration phase, Constant Contact does not return a 400 error for a `redirect_uri`"); a mismatch between the authorization request and the registered URIs returns a 400 at authorization time instead — misconfigurations surface late, with real users.
- **All-or-nothing consent:** users "must either grant access to all of your scopes or decline" — no partial grants; requesting fewer scopes is the only lever ("Account owners are more likely to authorize your application… when you request only the data that your application requires").
- **Legacy V2 is a different world:** the V2 API used different keys and flows and did not support the `scope` parameter; the current program is V3-only (V3 keys required). Don't mix V2 documentation into a V3 build.

## 6. Sources (official, actually read)

- https://developer.constantcontact.com/api_guide/getting-started/getting-started (Quick Start Guide: developer signup, New Application dialog, flow + refresh-method choices, API Key/client secret handling, private-access approval paragraph)
- https://developer.constantcontact.com/api_guide/oauth2-authorization/auth-overview (OAuth2 Overview: private-access gate, token definitions and lifetimes, flow comparison table, scopes, redirect/error behavior, token-endpoint rate limit)
- https://developer.constantcontact.com/api_guide/server_flow.html (OAuth2 Authorization Code Flow: authorize/token endpoints, parameters, 5-minute codes, token validation/JWKS, refresh procedure)

## 7. Unverified

- Approval criteria, checklist, and any timeline or SLA for the support-run approval process — the docs only give the phone number (866-289-2101) and say it "begin[s] the approval process."
- Any cost or fee for approval, listing, or API use — not stated in the docs read.
- Any marketplace/integration-directory listing process for V3 applications — not described in the developer pages read.
- Whether the flow and refresh-token-method settings can be changed on an existing application after creation, or require a new application — not stated in the pages read.
- Any numeric user/install cap after approval, and per-app rate limits beyond the token-endpoint warning — not stated.
- Whether the `expires_in: 28800` in the Authorization Code Flow example response is a documentation error or reflects actual issued tokens for some configurations — the docs contradict themselves (see §4).

---

**Cross-cutting note for the parent:** these two are opposite failure modes for the library. ActiveCampaign is a true dead-end case like Cricbuzz: there is no OAuth-as-provider program at all, and the honest guide is the API URL + Key paste flow (or building inside ActiveCampaign as a reviewed CX App, where OAuth points outbound at *your* API). Constant Contact has genuine, well-documented OAuth whose difficulty is concentrated in one place: the undocumented, phone-initiated approval that stands between a fully working private app (1 user: its creator) and any real customer connecting.