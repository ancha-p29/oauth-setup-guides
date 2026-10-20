Research complete. Below are the two structured reports. One methodology note up front: Smartsheet's narrative "OAuth Walkthrough" page itself could not be loaded in this session (the fetch service rate-limited it), so everything asserted for Smartsheet comes from other developers.smartsheet.com pages and the official OpenAPI spec that I did read; details that live only in the walkthrough are flagged as unverified rather than asserted.

---

# SMARTSHEET

## 1. What the OAuth app is called / recommended flavor
- The object is a **third-party application registered in "Developer Tools"** inside a Smartsheet account. The official registration page states: "With Developer Tools, you can register applications that request user authorization to access Smartsheet data." Registering yields a **client ID** and **app secret** (the token endpoint docs call it "The client ID you obtained when you registered your app").
- Smartsheet's implementation is standard **OAuth 2.0 authorization-code (3-legged) flow**. The Authentication guide: "If your scenario involves user consent and interaction, it's typically best to use OAuth with Smartsheet. Note that the Smartsheet implementation of OAuth 2.0 is a 3-legged process, which requires human intervention."
- **Recommended flavor for a third-party AI agent/SaaS:** one Developer Tools–registered app using the authorization-code flow, requesting only the access scopes it needs (see §5). The alternative documented auth method, a personal **API access token (PAT)** generated in the Smartsheet UI, is for your own/machine-to-machine use, not for customers connecting their accounts.
- Endpoints (from the official OpenAPI spec's OAuth2 security scheme): authorization URL `https://app.smartsheet.com/b/authorize`; token URL `https://api.smartsheet.com/2.0/token`.

## 2. Setup steps in order
1. **Register for Developer Tools** (human): at developers.smartsheet.com/register. Gate: "The email address used to register for Developer Tools must belong to an existing licensed Smartsheet user on a Business plan or higher." Submitting the form means agreeing to the **Smartsheet Developer Agreement**.
2. **Register the app in Developer Tools** (human, in the Smartsheet UI): Smartsheet assigns the client ID and app secret. A **redirect URI is registered on the app** (the token operation docs reference "the redirect URI you registered for your app"). Exact form fields: see Unverified.
3. **Implement the flow** (agent-automatable, standard code):
   - Redirect the user to the authorize URL; the user logs in and clicks **"Allow"** (the spec describes the code as "Authorization code acquired after user selects 'Allow' in the Web login UI"), and Smartsheet returns an authorization code.
   - Exchange the code: `POST https://api.smartsheet.com/2.0/token`, `Content-Type: application/x-www-form-urlencoded`, with `grant_type=authorization_code`, `code`, `client_id`, and **either** `client_secret` (plain text) **or** `hash` = `SHA_256(app_secret|code)`.
   - Response is the spec's `Token` object: `access_token`, `token_type` (bearer), `expires_in`, `refresh_token`.
   - Refresh: same endpoint with `grant_type=refresh_token`, `refresh_token`, `client_id`, and `client_secret` or `hash` = `SHA_256(app_secret|refresh_token)`.
   - Revoke: `DELETE /2.0/token` with the bearer token (optionally `deleteAllForApiClient=true` to delete all tokens for that client ID + user ID).
4. **Per-user consent** (human end user): each connecting user authorizes in the web UI; there is no way around the human step by design ("requires human intervention").

## 3. Approval / review
- **No app review, approval queue, marketplace, or listing process is described anywhere in the developer documentation I read** (developer homepage, API introduction, Basics/Authentication guides, Advanced Topics index, Developer Tools registration page, OpenAPI reference). Nothing in those docs conditions real users connecting on any Smartsheet-side approval, and no user cap before approval is mentioned. Registration + the Developer Agreement appear to be the only gates.
- Caveat: the OAuth Walkthrough (which I could not load) is the canonical narrative doc; if it contained a review step, it is not referenced by any of the surrounding docs I read. Treat "no review" as "no review documented in the docs reviewed," flagged in §7.

## 4. Time / effort
- **Provider-stated durations: essentially none for setup.** The docs give no registration timeline, no review timeline, and no "time to first user" figure.
- The one provider-stated lifetime: the spec's `Token` schema defines `expires_in` ("Number of seconds token is valid once issued") with a **default of 604799 seconds** (just under 7 days) for the access token.
- Active work is: one human registration flow (Developer Tools sign-up + app registration) plus standard OAuth client implementation. Waiting: none documented.

## 5. Blockers and nuances
- **Plan gate, two layers.** (a) Developer Tools registration requires an existing licensed user on **Business plan or higher**. (b) "The Smartsheet API is restricted to users on Business and Enterprise plans" (OpenAPI reference); the introduction lists required plan as "Business, Enterprise, or Advanced Work Management." Your customers' users must be on qualifying plans.
- **Coarse, all-or-nothing-style scopes.** 17 scopes defined in the spec, requested per authorization: `ADMIN_SHEETS` (modify sheet structure incl. columns/publish state), `ADMIN_SIGHTS`, `ADMIN_USERS` (add/remove org users, manage groups), `ADMIN_WEBHOOKS` (incl. "reset shared secret"), `ADMIN_WORKSPACES`, `CREATE_SHEETS`, `CREATE_SIGHTS`, `DELETE_SHEETS`, `DELETE_SIGHTS`, `READ_CONTACTS`, `READ_EVENTS`, `READ_SHEETS` ("Read **all** sheet data, including attachments, discussions, and cell data"), `READ_SIGHTS`, `READ_USERS`, `SHARE_SHEETS`, `SHARE_SIGHTS`, `WRITE_SHEETS`. There is no per-sheet or read-only-except-X granularity: `READ_SHEETS` is org-account-wide for that user. Scopes are also attached to operations individually (each endpoint lists the scope it requires).
- **Non-standard client authentication option:** besides sending `client_secret`, Smartsheet accepts `hash` = SHA-256 of `app_secret|code` (or `app_secret|refresh_token`). OAuth libraries won't do this for you by default; the plain-secret option exists, so standard libraries still work.
- **Token endpoint takes parameters as query/form fields with `application/x-www-form-urlencoded`**, per the spec's operation definition and code samples.
- **Regions are separate worlds.** Base URLs differ (api.smartsheet.com / .eu / .au, plus Smartsheet Gov api.smartsheetgov.com), and the Authentication guide warns: "Smartsheet Gov and Smartsheet Regions Europe use separate tokens from Smartsheet.com." Plan for per-region handling; the spec's OAuth scheme lists only the US authorize/token URLs.
- **Refresh tokens exist and are first-class** ("A credential tied to the access token that can be used to obtain a fresh access token with the same permissions, without further involvement from a user"), and `DELETE /token` supports revoking one token or all tokens for a client+user.
- **Webhooks can be app-scoped:** the spec's webhook subscription schema carries an `appClientId` and a `scopes` field (example given: `READ_SHEETS,WRITE_SHEETS`), i.e., webhooks created on behalf of an app are tied to the registering app, distinct from user-token webhooks.
- An optional `smartsheet-integration-source` header (`TYPE,OrgName,SourceName`, e.g. `AI,SampleOrg,My-AI-Connector-v2`) is documented for distinguishing third-party/AI-connector calls.

## 6. Sources (official, actually read)
- https://developers.smartsheet.com/ (developer homepage)
- https://developers.smartsheet.com/register (Developer Tools registration)
- https://developers.smartsheet.com/api/smartsheet/introduction
- https://developers.smartsheet.com/api/smartsheet/guides/basics/authentication
- https://developers.smartsheet.com/api/smartsheet/guides/advanced-topics (index; describes the OAuth guide as covering "the registration of third-party applications, the management of access and refresh tokens, and the scopes required to perform actions on behalf of a user")
- https://developers.smartsheet.com/api/smartsheet/openapi/tokens (Tokens tag page)
- https://developers.smartsheet.com/_bundle/api/smartsheet/openapi.yaml (official OpenAPI spec: OAuth2 scheme, scope definitions, /token operations, Token schema)

## 7. Unverified (docs read do not state)
- The OAuth Walkthrough page (developers.smartsheet.com/api/smartsheet/guides/advanced-topics/oauth) could not be loaded this session; therefore unverified: exact app-registration form fields and any developer-profile required fields; the authorize URL's exact query-parameter list; **authorization-code lifetime**; **refresh-token lifetime** and whether refresh tokens rotate; whether changing app settings (e.g., redirect URI) is restricted after creation.
- Any app review/approval step, marketplace/listing program, listing cost, or pre-approval user cap: **not mentioned in any developer doc read** — absence noted, not proven non-existent outside the docs.
- EU/Gov authorize URLs for OAuth (only US URLs appear in the spec's scheme).
- Any stated setup/approval timelines: none exist in the docs read.

---

# BASECAMP

## 1. What the OAuth app is called / recommended flavor
- The object is an **OAuth 2 application registered at 37signals Launchpad**: the official authentication guide says "Register your app at **launchpad.37signals.com/integrations**. You'll be assigned a `client_id` and `client_secret`. You'll need to provide a `redirect_uri`: a URL where we can send a verification code."
- **Recommended flavor for a third-party AI agent/SaaS:** exactly this — one Launchpad-registered app, standard **OAuth 2.0 authorization-code flow** ("web server" flow), with each of your users authorizing their own Basecamp account. Endpoints: authorize `https://launchpad.37signals.com/authorization/new`, token `https://launchpad.37signals.com/authorization/token`.
- The docs also mention two other credential types that are **not** the third-party SaaS path: **personal access tokens**, and in-product **AI agents** ("Agent — an AI agent added to the account. Agents create content and use the API as themselves… An agent authenticates with its own client-credentials token"), which act inside one account rather than connecting many customers' accounts.

## 2. Setup steps in order
1. **Register the app at launchpad.37signals.com/integrations** (human, signed in with their 37signals/Basecamp ID): provide the app details and a `redirect_uri`; receive `client_id` and `client_secret`. The guide notes you may "enter a dummy URL like `http://myapp.com/oauth` if you're not ready for this yet," i.e., the redirect URI is editable/not a launch gate.
2. **Configure an OAuth 2 library** with client_id, client_secret, redirect_uri, the authorize URL and token URL above. (Agent-automatable: all of the flow code.)
3. **Authorization** (end user, human): redirect the user to `https://launchpad.37signals.com/authorization/new?response_type=code&client_id=…&redirect_uri=…`. "We authenticate their Basecamp ID and ask whether it's ok to give access to your app." Basecamp redirects back with a **time-limited verification code**.
4. **Token exchange** (agent/backend): backchannel `POST https://launchpad.37signals.com/authorization/token` with `grant_type=authorization_code`, `client_id`, `client_secret`, `redirect_uri`, `code`. Response: `access_token`, `token_type: Bearer`, `expires_in: 1209600`, `refresh_token`.
5. **Discover the account** (agent): `GET https://launchpad.37signals.com/authorization.json` with the bearer token — "This should be the first request after obtaining a user's authorization token. Pick the account with `"product": "bc3"` and use its `href` as the base URL" (API calls go to `https://3.basecampapi.com/{account_id}/…`). It returns the token's `expires_at`, the user's identity, and all accounts they can access.
6. **Refresh** (agent): `POST` the token endpoint with `grant_type=refresh_token`, `refresh_token`, `client_id`, `client_secret`.
7. Legacy parameters `type=web_server` / `type=refresh` "are still accepted but not recommended"; use the standard `response_type`/`grant_type` parameters.

## 3. Approval / review
- **No review or approval of any kind is described before real users can connect.** The authentication guide's whole path is: register → get credentials → users authorize. No approval queue, no verification step, no user cap, no cost is mentioned anywhere in the API docs.
- **Listing is separate and optional:** the README's "Listing your application" section says only: "To add your application to our public list of Basecamp integrations, go to https://github.com/basecamp/bc3-integrations and open a pull request." No review criteria, timeline, or fee is stated for that listing, and nothing ties it to the ability to connect users.

## 4. Time / effort
- **Provider-stated setup/approval durations: none.** No timelines are given for registration (it reads as self-serve and immediate) and there is no review to wait for.
- Provider-stated token lifetime: access tokens live **2 weeks** (`expires_in: 1209600`; the Get authorization section repeats "access tokens have a 2-week lifetime").
- Active work: one human registration form at Launchpad, plus standard OAuth client code. Waiting: none documented.

## 5. Blockers and nuances
- **There are effectively no scopes in the Launchpad flow.** The documented authorize URL carries only `response_type`, `client_id`, and `redirect_uri` — no `scope` parameter, no scope list, no least-privilege selection. Authorization is all-or-nothing per user: the token acts with the authorizing user's access. (For Basecamp-issued tokens, the authorization document shows a `scope` field — the example value is `"full"` — and notes "Legacy Launchpad-issued tokens predate scopes, so a missing `scope` is not an error." The docs also mention per-account token requests via an RFC 8707 `resource` parameter, `urn:bc:account:<id>`, on the Basecamp-served authorization document.) An agent product therefore cannot down-scope; permission control is whatever the connected user can see/do.
- **Two issuers now exist:** Launchpad-issued tokens (the classic flow above) and Basecamp-issued tokens; Basecamp serves its own authorization document at `GET https://3.basecampapi.com/authorization.json` accepting "a Basecamp-issued OAuth token or personal access token, or a legacy Launchpad-issued token."
- **DPoP note:** the docs state a DPoP-bound token authenticates with `Authorization: DPoP <token>` plus a DPoP proof header, and "the `Bearer` scheme is rejected for bound tokens (RFC 9449 §7.1)" — relevant if you encounter sender-constrained tokens.
- **Mandatory User-Agent:** "You must include a `User-Agent` header with both the name of your application and a link to your application or your email address… If you don't include a `User-Agent` header, you'll get a `400 Bad Request`."
- **Do not build "Login with Basecamp":** "We don't recommend using the OAuth API to authenticate users in third-party services (e.g. a 'Login with Basecamp' button). We don't verify email addresses, so an attacker could gain access using email addresses they don't own." Also, the `identity` in the authorization document is "**not** used for identifying users within Basecamp — use the Get person endpoints for that."
- **Multi-account users:** one authorization returns every account the user can access; your app must select the `bc3` account's `href` per user, and API URLs are account-ID-prefixed on a different domain (`3.basecampapi.com`).
- **Verification code is "time-limited"** (exact lifetime not stated — see §7), and refresh is required every 2 weeks per user token.
- **Rate limits:** "Multiple rate limits are in effect… they're adjusted dynamically"; the first commonly hit is "currently 50 requests per 10 second period per IP address," with `Retry-After` on 429s.

## 6. Sources (official, actually read)
- https://github.com/basecamp/bc3-api (README: authentication summary, User-Agent rule, rate limiting, "Listing your application," agent person type)
- https://github.com/basecamp/bc-api/blob/master/sections/authentication.md (OAuth 2 overview, from-scratch flow, Get authorization, Refreshing access tokens)

## 7. Unverified (docs read do not state)
- Exact Launchpad registration form fields beyond name/redirect URI (the guide only names the redirect URI and the issuance of client_id/client_secret); whether registration requires a paid Basecamp account.
- Verification-code exact lifetime ("time-limited" only); refresh-token lifetime/rotation behavior; any token revocation endpoint (not covered in the authentication guide).
- Any approval/review for connecting users: **none is described** — reported as absence from the docs, per §3.
- The bc3-integrations listing PR: review criteria, timeline, and cost are not stated.
- Whether the newer Basecamp-issued OAuth (scopes, `resource` parameter, DPoP) has a separate app-registration route distinct from Launchpad: the authentication guide still directs registration to Launchpad; the scope/resource/DPoP details appear only in the authorization-document section, with no separate registration flow documented there.