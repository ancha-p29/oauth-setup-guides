# Teamwork

## 1. What the OAuth app is called / recommended flavor
- The object is an **“app”** created in the **Teamwork.com Developer Portal** (`https://{yourSiteName}.teamwork.com/developer`, or via developer.teamwork.com login). Creating it generates a `client_id` and `client_secret` (Credentials tab) used in Teamwork’s OAuth 2.0 implementation, which Teamwork calls the **App Login Flow**.
- There is only one third-party flavor — there are no separate internal/public app types in the docs. The relevant distinctions are app **status**: **“In development”** vs **“Published”** (after verification; see §3).
- Access is controlled by **Product Scopes**, not granular OAuth scopes: “you must choose at least one Product Scope if you want to allow your app to access your Teamwork.com data,” and “All APIs published on this portal allow you to access and update any data in your Teamwork.com account.” So scope selection is per-product (e.g. Teamwork/Projects, Desk, CRM, Spaces), and within a chosen product the app’s access is broad.
- For personal use/testing only, the docs say a per-user **API key / Basic (Bearer) auth** is acceptable; for an app others use, the docs direct you to the App Login Flow.
- Recommended flavor for a third-party AI agent/SaaS: a single Developer Portal app using the App Login Flow, with only the product scopes it actually needs (verification explicitly checks you requested “no more than is necessary”).

## 2. Setup steps in order
1. **Be a user in the owner company** of a Teamwork account — portal access requires this: “to access the Teamwork.com Developer Portal, you must be a user in the owner company. Users from other companies inside your Teamwork.com account will not have access.” *(Human: account/login prerequisite.)*
2. **Set up Publisher Details first** (prompted on first portal login; shared by all your apps): Name (shown on consent screen), Logo URL, Website, Privacy URL, Terms URL, VAT number, contact Email, registered Address (“used by Teamwork.com when validating your app”). *(Human: real company/legal details; agent can draft/assemble but cannot supply them.)*
3. **Create the app**: Name (shows on consent screen), Icon, Description, **Product(s)** (at least one mandatory), **Redirect URLs** (at least one mandatory). *(Human in portal UI; agent can prepare all values.)*
4. **Open the app → Credentials tab**: read `client_id`, `client_secret`, `redirect_uri`; you can manage scopes and add redirect URIs and optional **Origins** there. *(Human must retrieve/vault the secret; agent consumes it in code.)*
5. **Implement the App Login Flow** *(agent-automatable code)*:
   - Direct the user to the launchpad login URL. Generic (any user’s site): `https://www.teamwork.com/launchpad/login/?redirect_uri=XXX&client_id=YYY`; site-specific: `https://{yourSiteName}.teamwork.com/launchpad/login?redirect_uri=XXX&client_id=YYY`. An optional `state` parameter is passed through.
   - User authenticates on Teamwork’s side (“This step does not require any work from your side”); Teamwork redirects to your `redirect_uri` with a `code` parameter (plus `state` if sent).
   - POST JSON to `https://www.teamwork.com/launchpad/v1/token.json` with `code`, `client_id`, `client_secret`, `redirect_uri` (content-type `application/json`). Response contains the `access_token` plus basic installation info, including the `apiEndPoint` URL to use for that installation.
   - Call APIs with `Authorization: Bearer {token}`; optionally GET `https://www.teamwork.com/launchpad/v1/userinfo.json` for user/installation info (useful when one user has multiple Teamwork accounts).
6. **Develop/test** against your own account (the only account available pre-verification).
7. **Request verification** to go live (see §3), then status flips to “Published.”

## 3. Approval / review
- **Yes — review is required before real (external) users can connect.** Pre-verification, “your app will only have access to the data in the selected Teamwork.com account,” and an “In development” app “can only be used by the Teamwork.com account it was created under.” So the pre-approval cap is effectively **one account: your own**.
- Process: “Once you are ready to publish your app, reach out to **integrations@teamwork.com** to kick off the verification process. We will check the details on this checklist, and send the app to our QA team for testing. **Please allow up to 5 working days to get a response from our team.**” (The App Login Flow guide phrases it as: “Our team will guide you through the approval process.”) Note the 5 working days is stated for a *response*, not a guaranteed full approval time.
- What QA checks (verification checklist): app sanity (clear/honest name & description, visible icon; at least one product scope but no more than necessary; redirect URIs entered, no more than necessary, https only), OAuth 2.0 implemented via the App Login Flow credentials, working consent screen, API best practices (paging; rate-limit handling), no impact to user’s Teamwork data on initial sign-up, branding-guideline compliance, user-permissions/privacy testing, and that the app enhances the Teamwork experience and involves some interaction with Teamwork. You must provide a way for Teamwork to test the app.
- On approval: “Once the QA team approve your app, we will set it to published!” and the Developer Portal status changes to “Published,” making it “available to everyone.”
- **Cost:** no fee is mentioned anywhere in the docs read — unverified whether any fee exists.
- There is no self-serve marketplace listing flow described; publication is this email-triggered verification.

## 4. Time / effort (provider-stated figures only)
- Setup/portal configuration time: **not stated** in the docs.
- Code expiry (affects flow design, not project time): the returned `code` “expires in **15** minutes.”
- Verification waiting: “Please allow **up to 5 working days** to get a response from our team.” No total approval duration beyond first response is stated.
- Rate-limit recovery (from the verification page’s best-practices section): on HTTP 429, “After **60 seconds**, we'll let you back in!”

## 5. Blockers and nuances
- **Single-account dev cap** is the big gate: nothing external works until Teamwork verifies the app.
- **Permanent token, no refresh flow documented:** on success “the resulting payload will contain a **permanent access token**.” No refresh token, rotation, or token expiry for the access token is documented on the pages read — plan storage/security accordingly.
- **15-minute code window:** exchanging late returns 401; the docs also list payload mistakes (sending body as URL params, wrong values) as 400 causes, and non-POST as 405.
- **Per-installation API endpoint:** the token response gives an `apiEndPoint` you must store and use; Teamwork has multiple data-center regions and says the App Login Flow “will handle everything” — don’t hardcode one site’s API base for all installs. Multi-account users are a real case; use `userinfo.json`.
- **Redirect URI rules:** Credentials tab says redirect URIs “must be secure URIs with **https** only unless they are using `localhost`.” The flow guide also mentions `customprotocolapp://` and Chrome-extension redirect URLs as passable redirect URIs; treat https (or localhost) as the safe documented default. If your app has allowed **origins**, an additional `origin` header is required on the token request.
- **Coarse scopes:** product-level only — no read-only or per-resource scopes are documented. Choosing a product effectively grants API access/update to that product’s data (still bounded by the authorizing user’s own permissions per the product API guides: users only access “the information that is accessible through their profile”).
- **Scope changes re-trigger consent/re-auth:** “adding new products will require you to re-authenticate to access APIs under the new scopes”; a consent screen appears the first time a user logs in and again “unless the app scopes are changed.”
- **Publisher is a one-way-ish door:** any user can set the publisher to start development, but “once it’s set, only a Teamwork.com Administrator can update these details.”
- **Portal role model:** App Owners can update details/scopes/redirect URIs/origins, delete the app, update the secret, manage app users/owner; App Users can only edit name/description; all owner-company users can view apps and create their own.
- **Verification failure modes are mundane:** the checklist implies rejections commonly stem from missing paging, ignoring 429s, data changes on sign-up, or branding/privacy issues — test these before submitting.

## 6. Sources (official docs actually read)
- https://apidocs.teamwork.com/guides/teamwork/developer-portal
- https://apidocs.teamwork.com/guides/teamwork/create-an-app
- https://apidocs.teamwork.com/guides/teamwork/set-up-your-publisher-details
- https://apidocs.teamwork.com/guides/teamwork/developer-portal-permissions
- https://apidocs.teamwork.com/guides/teamwork/app-login-flow
- https://apidocs.teamwork.com/guides/teamwork/app-verification
- https://apidocs.teamwork.com/guides/spaces/authentication (auth methods; same App Login Flow guidance repeated across the Desk/Spaces API guides)

## 7. Unverified (not stated in the docs read)
- Any verification fee/cost; total approval duration beyond the “up to 5 working days” first response; whether re-review is needed after post-publication scope/config changes.
- Access-token expiry/revocation behavior (docs call it “permanent” and document no refresh token or revocation path).
- Any numeric rate limit (only the 429 + ~60-second lockout behavior is described; no requests-per-period figure).
- Whether unpublished apps can be installed by a limited set of external test accounts (docs say own account only).
- Exact list of selectable Product Scopes in the portal (docs reference products/APIs but do not enumerate the checkbox list).

---

# Sentry

## 1. What the OAuth app/object is called / flavors
Sentry’s docs distinguish three relevant things, all managed under **Settings > Developer Settings** (sentry.io) for the integration platform:

- **Public integrations** (the “Sentry apps” of the Integration Platform): built for “the ‘general public’ of Sentry users,” installable by any Sentry organization **once published**. They **require an OAuth flow** for tokens: after a user installs and approves permissions, Sentry issues a grant code + installation ID, which you exchange for an organization-scoped installation token. **This is the recommended flavor for a third-party AI agent/SaaS whose users connect their own Sentry organizations.**
- **Internal integrations**: “custom integrations unique to your organization… for use within a single Sentry organization.” They **do not require an OAuth flow** — “You receive an org-wide auth token immediately after creation,” and creating one “will automatically install it on your organization.” Wrong flavor for a multi-customer product; right flavor for generating an org auth token for your own org.
- **Generic OAuth2 (API Authentication page)**: separately, the API docs state “For third-party applications that need to access Sentry on behalf of users, Sentry supports **OAuth2 with the authorization code grant type**,” hitting `https://sentry.io/oauth/authorize/` and `https://sentry.io/oauth/token/`, with PKCE (S256) recommended and a device authorization grant (RFC 8628) for headless/CLI use. These are **user** tokens (the response includes the authorizing `user`), distinct from public-integration **installation** tokens.

Permissions are org-scoped scopes applied to tokens issued to the application: `org:read/write/admin`, `org:ci`, `project:read/write/admin`, `project:releases`, `team:read/write/admin`, `member:read/write/admin`, `event:read/write/admin` (per the Permissions & Scopes table). Key constraint: “You can't create an integration or change an existing integration to have permissions greater than your own user settings,” and “Permissions are scoped to the organization” — a scope grants that access across the whole installing organization.

## 2. Setup steps in order
For a **public integration**:
1. **In sentry.io: Settings > Developer Settings → create a new integration and choose “public.”** Configure name/description and the integration’s settings. *(Human: requires a Sentry account/org and choices; the config form-filling would need the human’s logged-in session.)*
2. **Configure permissions (scopes)** the integration needs; for public integrations “Sentry will prompt users to approve these permissions upon installation.” *(Human product decision; capped by your own user permissions.)*
3. **Configure webhooks**: specify the **webhook URL** when creating the integration to receive resource events; optionally toggle **“Alert Action”** (requires the alert-action setup so alerts can use `Send a notification via <your integration>`). Optionally add **UI components** (declarative components rendered inside sentry.io). *(Human configures URLs/toggles; agent builds the receiving service.)*
4. **Configure the redirect URL** (where users land after install) and, if your service does post-install finalization work, enable **“Verify Install.”** *(Human config; agent implements endpoint.)*
5. **Upload the required logo** (see §5 specs); upload the icon too if you use UI components. *(Human supplies assets.)*
6. **Build the install/OAuth backend** *(agent-automatable code)*:
   - Users install from the **Integrations** page in sentry.io, or via the fixed external URL `https://sentry.io/sentry-apps/<your-integration-slug>/external-install/` (multi-org users pick the organization from a dropdown).
   - After the user approves all permissions, Sentry sends a **grant code + installation ID** either via the `installation.created` webhook to your webhook URL, or as query params (`code`, `installationId`) on your redirect URL.
   - Implement a `/setup` endpoint that POSTs to `https://sentry.io/api/0/sentry-app-installations/{installationId}/authorizations/` with `grant_type: "authorization_code"`, `code`, `client_id`, `client_secret`. Response: `token`, `refreshToken`, `expiresAt`, etc. Securely store install ID + both tokens per installation.
   - If “Verify Install” is on, PUT `https://sentry.io/api/0/sentry-app-installations/{}/` with `{"status": "installed"}` to mark installation complete.
   - Handle `installation.deleted` webhooks on uninstall.
7. **Refresh tokens on an 8-hour cycle** (see §5): either the standard `grant_type: "refresh_token"` exchange at the same authorizations endpoint, or Sentry’s recommended **manual refresh**: send a JWT signed with your client secret (claims `iss`/`sub` = client_id, `iat`, `exp` = now+60s, `jti` = UUID) as a Bearer header with payload `grant_type: "urn:sentry:params:oauth:grant-type:jwt-bearer"`. *(Agent.)*
8. **Publish** when ready (see §3). *(Human clicks Publish.)*

For completeness — **internal integration** setup is: Developer Settings → create internal integration → set permissions → it auto-installs on your org and generates a token immediately; manage/add tokens from the integration’s page (Settings > Developer Settings > [integration]).

The separate **generic OAuth2** flow (user tokens): redirect user to `https://sentry.io/oauth/authorize/?client_id=…&response_type=code&scope=…` (optional `redirect_uri` — “must match registered URI,” `state`, PKCE `code_challenge`/`code_challenge_method=S256`), then POST the code to `https://sentry.io/oauth/token/` with `client_id`, `client_secret`, `grant_type=authorization_code`, `code` (plus `code_verifier` if PKCE). Device grant: POST `https://sentry.io/oauth/device/code/`, user approves at `https://sentry.io/oauth/device/`, poll the token endpoint.

## 3. Approval / review
- **Public integrations start unpublished:** “These start in an unpublished state for development purposes and can be submitted later for approval to publish.” While unpublished, “the only Sentry organization that will be able to see or install it will be the organization that created the integration.” So the pre-approval cap is effectively **your own organization only** — real external users cannot install until published.
- **Publication is a review submission, done by email:** “When you're ready for the publication process, click ‘Publish’ next to the integration you want to submit. This sends an email to **integrations-platform@sentry.io**, letting us know your integration is ready for review.”
- **Once published**, the integration is listed for all Sentry users; clicking it shows its description and “the permissions that will be granted if the user chooses to install.”
- **Stated timeline: none.** The public-integration page gives no review duration, SLA, or criteria beyond the content/logo requirements. **Cost: none stated** (unverified).
- **Internal integrations:** no review/publication concept — they are private to the creating org and auto-installed.
- **Generic OAuth2 user flow:** the API auth page describes no review/approval step for registering/using an OAuth client — but see §7 (client registration itself is not documented on the pages read).

## 4. Time / effort (provider-stated figures only)
- Example-project onboarding: the Integration Platform page says of its example app, “Typically, the app should be safely consuming live Sentry data **within minutes**.” No other setup duration is stated.
- Publication review duration: **not stated**.
- Token lifetimes that drive ongoing engineering effort: public-integration tokens “automatically expire **every eight hours**”; generic OAuth2 access tokens “expire after **30 days**” (example response: `expires_in: 2591999`); device codes expire in **600 seconds (10 minutes)** with a default **5-second** polling interval.
- Internal-integration tokens “do not expire automatically.”

## 5. Blockers and nuances
- **Publication gate is opaque and manual:** unpublished = your org only; going public is an email-triggered review with no documented timeline or checklist on the page read. This is the schedule risk for launch.
- **8-hour token expiry for public integrations** is the signature trap: “Public integrations generate authentication tokens after a successful OAuth installation. These tokens automatically expire every eight hours, meaning they must be refreshed manually.” Your backend must run refresh reliably per installation.
- **Lost-refresh anomaly has a dedicated workaround:** Sentry notes refreshes can commit on Sentry’s side but the new token can be “lost in transmission,” which is why they recommend the **manual JWT-bearer refresh** method over plain refresh-token exchange.
- **Tokens are installation/org-scoped, not per-user** (public integrations): permissions apply across the installing organization, and users approve the full permission set at install; there is no documented per-project or partial-consent subsetting at install time.
- **Scope ceiling = your own permissions:** you cannot create or edit an integration to request permissions greater than your own user’s.
- **Generic OAuth2 tokens are single-org:** “A Sentry user can belong to multiple organizations. The access token only provides access to the specific organization the user selected during the OAuth flow.”
- **Internal-integration token cap:** “You can have up to **20 tokens** at a time for each internal integration,” revocable manually; creation auto-installs the integration on the org.
- **Logo is mandatory for public integrations** (optional for internal): PNG, **256×256 to 1024×1024 px**, transparent background unless it fills the space. With UI components, an icon is also required: PNG RGBA, same size range, “All pixels must be black or transparent.”
- **Browser use / JS origins:** tokens can be used from the browser only for origins listed in “Authorized JavaScript Origins” (one per line, no protocol) — and “you can't use any wildcard characters (for example, `*.sentry.io`),” so subdomains must be listed individually.
- **Alert Action dependency:** to receive alert webhooks you must have the webhook URL set and the integration selected as the `Send a notification via <your integration>` alert action; just enabling webhooks isn’t the whole story.
- **Uninstall signal:** expect an `installation.deleted` request to your webhook URL when a user uninstalls.
- DSNs/API keys exist for other purposes (event ingestion, personal tokens under User settings) but are not the third-party OAuth path; the docs steer third-party apps to OAuth2/integration platform.

## 6. Sources (official docs actually read)
- https://docs.sentry.io/integrations/integration-platform/
- https://docs.sentry.io/organization/integrations/integration-platform/public-integration
- https://docs.sentry.io/integrations/integration-platform/internal-integration/ (docs.sentry.io)
- https://docs.sentry.io/api/auth/
- https://docs.sentry.io/api/permissions/

## 7. Unverified (not stated in the docs read)
- Publication review timeline, criteria/checklist, and any cost — the docs only document clicking “Publish,” which emails integrations-platform@sentry.io.
- Where/how a standalone OAuth2 `client_id`/`client_secret` is **registered** for the generic API-auth OAuth2 flow: the auth page assumes a “registered client ID” and a `redirect_uri` that “must match registered URI,” but the pages read do not document the registration step.
- Whether unpublished public integrations can be installed by any org via the external-install URL or are hard-limited to the creating org in all cases (page says only the creating org can “see or install” it while unpublished).
- Any user/install caps for published integrations; any rate limits for the authorizations endpoint.
- Review/re-review requirements when changing scopes on an already-published integration; scope behavior when a user declines some permissions (page conditions token issuance on the user having “approved all permissions”).
- Whether the 8-hour expiry/refresh behavior differs for self-hosted Sentry (internal integrations are noted as available for self-hosted; public-integration details are written against sentry.io).