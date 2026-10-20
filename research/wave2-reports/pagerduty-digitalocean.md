Research complete. Both providers have real OAuth programs for third-party apps. Below are the two structured reports, sourced only from official documentation I read directly.

---

# PagerDuty

## 1. The OAuth object and the recommended flavor

The object is a **PagerDuty App**, created through **App Registration** (the "My Apps" page). An app is a container; you add **OAuth 2.0 functionality** to it, and at that point you must choose one of two flavors:

- **Classic User OAuth** (legacy): authorization code grant acting as a user, with a single blanket scope — `read` or `write` (case-sensitive). Works on **any** account the moment it is registered; no publishing required.
- **Scoped OAuth**: per-resource-type scopes (e.g. `incidents.read`, `services.read` — every REST API endpoint documents its required scope). Supports two token types: a **user token** (authorization code grant) and an **app token** (client credentials grant, acting as the app itself with no user). Scoped OAuth apps are **always confidential clients**: the `client_secret` must be sent on the token request **and PKCE is mandatory**. Non-confidential clients (browser/mobile apps that cannot hold a secret) are only possible with Classic User OAuth.

**Recommended for a third-party AI agent/SaaS whose users connect their own accounts: Scoped OAuth with user tokens.** It is the only least-privilege option, and the docs position OAuth apps as "the preferred choice for both third-party integrations and long-lived applications." But note the constraint in §3/§5: cross-account use of Scoped OAuth requires publishing (PagerDuty review) plus an admin install on every customer account, and on other accounts you get **user tokens only**. If you need to ship before review, Classic User OAuth is the only flavor that works cross-account immediately, at the price of blanket read/write access.

## 2. Setup steps in order

1. **Get access to App Registration.** Either a PagerDuty developer account, or a customer account where your base role is **Manager, Global Admin, or Account Owner** — App Registration is not available below Manager. *(Human: account/role provisioning.)*
2. **Register the app.** Log in → **Integrations → Developer Tools → App Registration → My Apps → New App**; enter name and brief description in the wizard. *(Human, web UI; no app-registration API is documented.)*
3. **Add OAuth functionality.** On the app's Edit App page, in the **Functionality** section click **Add** on the OAuth card → on the **Configure OAuth 2.0** screen choose **Classic User OAuth** or **Scoped OAuth**. Configure the redirect URL (registered with the app; the runtime `redirect_uri` must match it) and the permission scopes. Saving generates the `client_id` and `client_secret`. *(Human in UI; an agent can prepare all values. Choosing the flavor here matters — see §5.)*
4. **Implement the flow in your code** against `https://identity.pagerduty.com/oauth/authorize` and `https://identity.pagerduty.com/oauth/token`. Send the user to the authorize endpoint (`response_type=code`, `scope`, plus PKCE `code_challenge`/`code_challenge_method=S256` — always for Scoped, recommended for Classic); exchange the returned code at the token endpoint with `client_secret` (confidential clients) and `code_verifier` (when PKCE is used); store the access + refresh tokens; implement refresh. *(Fully automatable — this is ordinary code. Scoped app tokens are even simpler: a single POST with `grant_type=client_credentials` and a scope string that must include `as_account-{region}.{subdomain}`, regions `us` or `eu`.)*
5. **Only if distributing**: change **App Type** to Public and complete the publishing flow in §3. *(Human.)*

Effective access is always the **intersection of the app's granted scopes and the authorizing user's own permissions** (for app tokens, scopes alone).

## 3. Approval / review

- **Classic User OAuth: no review needed for real users to connect.** "A user on any account can authorize it as soon as it is registered, without publishing." Publishing is optional and *recommended* — its benefit is being listed in PagerDuty's integrations library for discovery.
- **Scoped OAuth: review is required before it works on any account other than the one that created it.** Until published, a Scoped OAuth app "behaves as a private app: it only works on the account that created it." The same applies to Events Integration functionality.
- **The publishing review**: on the app's configuration page set App Type to **Public**, open the **Distribution Form**, and enter listing information (documentation, support contact) plus **instructions for testing your app**; Save, then Save on the Edit App page and confirm submission. You get a confirmation email. **Stated timeline (quoted): "We aim to review your submission and approve or reply with feedback within 5 business days."** During review PagerDuty will test the app's functionality, review documentation for completeness and accuracy, and verify listing information. Reviewers may request changes/info by email (apps@pagerduty.com).
- **One-way-door in the process**: "Once you submit, you will no longer be able to make updates to your app." Getting it back into an editable draft requires emailing apps@pagerduty.com.
- **Second gate after approval**: before any user on a customer account can authorize a published Scoped OAuth app, **an admin on that account must install it** (via `https://app.pagerduty.com/oauth_apps/[app_id]`, which shows the app name, requested scopes, and token kinds). Installing grants no access by itself; each user still completes the OAuth flow individually. Until installed, authorization attempts from that account fail.
- **Cost:** no fee is mentioned anywhere in the publishing docs (see Unverified). **User caps before approval:** none stated for either flavor — the Scoped restriction is account-scope (your own account only), not a user count.

## 4. Time / effort (provider-stated figures only)

- **Review wait (only if publishing):** "within 5 business days" to approve or reply with feedback. No other timelines are stated; the docs give no figures for hands-on setup time.
- **Token lifetimes (stated, these drive your refresh engineering):**
  - Classic User OAuth: access token **30 days**, refresh token **210 days**, rolling refresh window **3 years** (continuous use as long as there is activity at least once every 210 days).
  - Scoped OAuth: access token **1 day**, refresh token **30 days**, rolling refresh window **1 year** (use at least once every 30 days). Refresh tokens apply to **user tokens only**; when an app token expires "the application can simply request a new token."

## 5. Blockers and nuances

- **Authorization codes live 30 seconds.** "The authorization code is valid for 30 seconds" and the token POST "must be received within that time." Slow callbacks will fail.
- **Invalid scopes fail silently.** If you request an invalid scope, PagerDuty returns **no error**; it grants partial scopes — only the `openid` scope that is automatically attached to all tokens. You must inspect the `scope` field of the token response to detect this.
- **Scoped OAuth cannot act as the app on customer accounts.** The client credentials (app token) flow "only works on the account that created the app." On other accounts, user tokens only.
- **Apps are immovable.** "Apps are associated with a PagerDuty subdomain and a user and cannot be moved between subdomains or users. Continued access to this subdomain and user is needed to make any configuration changes to the app." Losing that user/subdomain orphans the app configuration.
- **PKCE matrix traps:** Scoped OAuth = secret + PKCE both mandatory, non-confidential clients unsupported. If your client is a browser/mobile app, the docs explicitly direct you to Classic User OAuth as a non-confidential client — i.e. you must give up granular scopes.
- **Deleting OAuth functionality does not revoke tokens.** You must use "Revoke All Tokens" (on the OAuth Client screen) *before* deleting the functionality, or outstanding tokens survive.
- **Role gate:** only Manager+ base roles can even open App Registration, so the person registering the app is an ops decision, not a dev convenience.
- **App Type does not apply to Classic User OAuth** — Classic works cross-account regardless; "Private" app type only constrains Scoped OAuth (and Events) apps.
- **403 semantics:** a 403 can mean the token lacks the endpoint's scope *or* the endpoint does not yet support API scopes; when a token expires you get 401 and must obtain a new one.

## 6. Sources (official PagerDuty developer docs actually read)

- App Overview — https://developer.pagerduty.com/docs/ZG9jOjExMDI5NTY3-app-overview
- Register an App — https://docs.pagerduty.com/developer/register-an-app
- OAuth Functionality — https://docs.pagerduty.com/developer/oauth-functionality
- Private Apps (Scoped OAuth, app tokens, token lifetimes) — https://docs.pagerduty.com/developer/private-apps
- Publish an App — https://docs.pagerduty.com/developer/publish-an-app

## 7. Unverified (not stated in the docs read)

- Any **cost or fee** for publishing/listing review — never mentioned.
- Any **user/install cap** before approval, and any limit on number of apps per account.
- Whether redirect URIs are limited in count, and whether multiple redirect URIs per app are supported (docs speak of "the Redirect URL" registered with the app).
- Whether app registration/configuration can be done via API — docs describe UI only.
- Exact review rejection criteria beyond the three stated review activities (functionality testing, documentation review, listing-info verification).
- `client_secret` rotation/regeneration procedure (docs cover revoking tokens and deleting/recreating OAuth functionality if a secret is compromised, not rotation).
- Per-app rate limits for the OAuth endpoints themselves (the docs note a 300 req/min operation-specific limit for the `/incidents` REST endpoint, which is API usage, not OAuth).

---

# DigitalOcean

## 1. The OAuth object and the recommended flavor

The object is an **OAuth application** registered with DigitalOcean's authorization server (`https://cloud.digitalocean.com/v1/oauth`). The OAuth API "lets you obtain limited access to DigitalOcean **teams**" — tokens are granted against a team, and the token response carries the user's and team's UUIDs/names.

Two client types:

- **Confidential clients** (can hold a secret, e.g. a server-side web app): registered **manually in the control panel** (the **OAuth Applications** page), which assigns a `client_id` and `client_secret`. Use the **authorization code flow**.
- **Public clients** (native, desktop, CLI, and explicitly **Model Context Protocol (MCP) clients**): no secret; the code exchange is secured with **PKCE (S256 only)**. "You typically create them using **Dynamic Client Registration**" — an unauthenticated POST to `/v1/oauth/register`; the server also publishes a metadata document at `https://cloud.digitalocean.com/.well-known/oauth-authorization-server` so such clients can discover endpoints and self-register programmatically.

**Recommended for a third-party AI agent/SaaS whose users connect their own accounts: a confidential client with the authorization code flow**, registered once in the control panel. (If the agent itself is a local MCP/CLI client, the public-client + Dynamic Client Registration + PKCE path is the documented one.)

## 2. Setup steps in order

1. **Register the application (confidential clients).** In the DigitalOcean control panel, on the **OAuth Applications** page, register the application; this assigns the client ID and client secret. The `redirect_uri` you later send must "match a callback URL you provided during application registration" — so the callback URL is set here. *(Human, control panel UI. An example authorization link for a registered app is viewable in the control panel: OAuth Applications page → "…" next to the app → View → OAuth Application Details.)* — or, for public clients, **register programmatically**: POST JSON (`redirect_uris` 1–5, `token_endpoint_auth_method: "none"`, `grant_types` including `authorization_code`, optional `client_name`/`client_uri`) to `/v1/oauth/register`; store the returned `registration_access_token` and `registration_client_uri` — the token is **shown only in that response and cannot be retrieved again**; it is required to read/update/delete the client later. *(Fully automatable.)*
2. **Send users to authorize.** Direct them to `https://cloud.digitalocean.com/v1/oauth/authorize` with `client_id`, `redirect_uri`, `response_type=code`, optional `scope`, `state` (recommended), `prompt`, and for public clients `code_challenge` + `code_challenge_method=S256`. *(Automatable code. The signed-in user authorizes the app for a team; this consent step is the human's.)*
3. **Exchange the code.** POST to `/v1/oauth/token` with `grant_type=authorization_code`, `code`, `client_id`, `redirect_uri`, and `client_secret` (confidential) or `code_verifier` (public). The response returns `access_token` (`doo_v1_` prefix), `refresh_token` (`dor_v1_` prefix), `expires_in`, granted `scope`, and user/team `info`. *(Automatable.)*
4. **Call the API and refresh.** Use the access token as a Bearer token against `https://api.digitalocean.com/v2/...`. Refresh via POST `/token` with `grant_type=refresh_token`. Revoke via POST `/revoke`. *(Automatable.)*

## 3. Approval / review

- **No review of OAuth applications is described anywhere in the OAuth documentation.** Registration (manual or dynamic) is self-serve, and users can authorize the app once it exists. There is no submission step, no approval queue, no stated waiting period, **no user cap before approval, and no cost** in the docs read. (This is an absence-of-review finding: the docs simply contain no review step for OAuth apps — see Unverified.)
- **Marketplace distinction:** DigitalOcean Marketplace "Add-Ons" are third-party applications distributed through the Marketplace, but that is a separate program from OAuth application registration; nothing in the OAuth docs ties OAuth apps to a Marketplace listing or review.

## 4. Time / effort (provider-stated figures only)

- **Waiting:** none documented — no review or approval timeline exists in the OAuth docs, and no setup duration is stated.
- **Token lifetimes (stated):** access tokens expire **30 days** after issue for confidential clients (`expires_in` 2592000) and **one hour** for public clients (`expires_in` 3600). Refreshing keeps access alive; the refresh token's own lifetime is not stated (see Unverified).

## 5. Blockers and nuances

- **Refresh tokens are single-use, and refreshing kills the old access token.** "You can use a refresh token exactly once to create a new access token (and refresh token). Doing so invalidates the access token that the refresh token was issued with." You must persist the *new* refresh token on every refresh; losing it (or racing two refreshes) breaks the connection.
- **Scope model has two layers.** The token response example uses the legacy broad scopes (`read`; `read write` for full access). Separately, DigitalOcean documents **granular custom scopes** of the form `<resource>:<action>` (`droplet:create`, `database:read`, `kubernetes:access_cluster`, …), plus two **alias scopes**: `api:read` (expands to all read scopes) and `api:write` (equivalent to all scopes) — and the aliases **automatically expand to include new API endpoints as they are added**, so an `api:write` grant silently grows over time.
- **Omitting `scope` hands the choice to the user.** "If you omit the `scope` parameter in the authorization request, the consent screen displays a **Manage permissions** option, allowing the user to select specific granular scopes to grant" — your app may receive less than it assumed.
- **Public-client redirect flexibility:** public clients may register more than one redirect URI (dynamic registration allows 1–5), and **loopback redirect URIs (`127.0.0.1`, `[::1]`, `localhost`) match on any port**, so native apps can use ephemeral ports.
- **`prompt=none` silent flow:** already-authorized signed-in users are redirected back with a new token; others get `error=consent_required` or `error=login_required`. An account hint can be supplied via the first 6 characters of an expected account UUID.
- **Deleting a dynamically registered client deletes everything:** DELETE on `/register/{client_id}` "remove[s] the client along with its tokens and grants."
- **Implicit flow discrepancy worth knowing:** the `/authorize` documentation still describes `response_type=token` (implicit flow, token returned in the callback fragment) for client-side apps, while the published authorization-server metadata lists `response_types_supported: ["code"]` only, and `grant_types_supported: ["authorization_code", "refresh_token"]`. The docs present both; plan on the code flow.
- Tokens are team-scoped in effect: authorization grants access to a **team**, and the grant response identifies `team_uuid`/`team_name`, so multi-team users pick which team they are connecting.

## 6. Sources (official DigitalOcean docs actually read)

- DigitalOcean OAuth API reference — https://docs.digitalocean.com/reference/api/oauth/
- Scopes for API Tokens — https://docs.digitalocean.com/reference/api/scopes/

## 7. Unverified (not stated in the docs read)

- The **exact control-panel registration form fields** for a confidential application (the current reference confirms registration happens in the control panel on the OAuth Applications page and that a callback URL is provided at registration, but does not enumerate the form).
- **Refresh token lifetime/expiry** — only its single-use behavior is stated.
- Whether confidential clients may register **multiple callback URLs**, and any cap on number of OAuth applications per account/team.
- Whether confidential clients can also use PKCE in the token exchange for web apps is stated as allowed "for additional protection," but secret rotation/regeneration procedures are not documented in the pages read.
- Any **review, fee, or listing requirement** for OAuth applications — absent from the docs (reported as "none described," not as a stated guarantee), and the Marketplace Add-Ons vendor review process was not covered by the OAuth documentation read.
- Rate limits specific to the OAuth endpoints, and whether a registered app's name/description shown on the consent screen can be edited after registration.

---

**Bottom line for the library:** PagerDuty = two-flavor system where the *good* flavor (Scoped OAuth) is gated by a stated 5-business-day publishing review plus per-account admin install, while the ungated flavor (Classic) forces blanket read/write. DigitalOcean = no review at all, self-serve registration (manual for confidential clients, fully programmatic Dynamic Client Registration for public/MCP clients), with the sharp edges in token behavior instead: single-use refresh tokens that invalidate the prior access token, and alias scopes that expand automatically.