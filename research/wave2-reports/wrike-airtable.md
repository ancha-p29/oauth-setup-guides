Research complete. Both providers have real, self-serve OAuth programs; neither docs set describes any pre-launch review. Details below, official docs only.

---

# WRIKE

## 1. The OAuth object and recommended flavor
The object is a **Wrike API application** ("API App"), created in the **Apps & Integrations portal** inside Wrike (the OAuth doc also calls the management surface the **App Console**). Registration assigns a unique **Client ID and Client Secret**. Wrike supports **only the authorization code flow** of OAuth 2.0 ("At the moment, we support only authorization code flow"), so the recommended (and only) flavor for a third-party agent/SaaS is: one API App registered by the developer, standard authorization-code flow, acting **on behalf of each end user** who consents ("Wrike API requests are executed on behalf of the end-user, who gives consent for the third-party app through OAuth2 protocol"). There is no client-credentials/daemon flavor.
A separate alternative exists but is explicitly not for this use case: the **Permanent Access Token** (see §5).

## 2. Setup steps in order
1. **Be a Wrike user** (any plan; FAQ: API use "is completely free and allowed for accounts with any subscription plan (including Free)"). *Human: account sign-up/login.*
2. **Create the API App** in the Apps & Integrations portal / App Console; note the Client ID and Client Secret; register redirect URI(s). *Human in the portal UI; an agent can prepare the values but cannot create the app without account access.*
3. **Send the user to the authorize URL** `https://login.wrike.com/oauth2/authorize/v4?client_id=…&response_type=code` with optional `redirect_uri`, `state`, and `scope` (comma-delimited, case-sensitive; each API method's doc lists its required scopes, e.g. Create Task requires `Default, wsReadWrite`). *Agent-automatable (code).*
4. **User logs in and consents.** If the user belongs to multiple accounts, an extra screen asks them to select one account. *Human only.*
5. **Exchange the code**: POST to `https://login.wrike.com/oauth2/token` with `client_id`, `client_secret`, `grant_type=authorization_code`, `code`. The response includes access token, refresh token, `expires_in: 3600`, and a **`host`** value. *Agent-automatable.*
6. **Call the API at `https://<host>/api/v4`** with the token in the Authorization header — the host comes from the token response because data sits in US or EU data centers. *Agent-automatable.*
7. **Refresh** via the token endpoint with `grant_type=refresh_token` (the docs note refreshes should use the host-specific base URL). *Agent-automatable.*

The FAQ is explicit that the consent leg cannot be automated: the initial OAuth run "cannot be fully scripted or run with cURL."

## 3. Approval / review
**None described.** The official developer docs contain no app review, approval queue, or marketplace listing process for API Apps: registration is self-serve in the portal and users can authorize immediately. No user caps before approval are stated (there is no approval). Cost: none — "Usage of our API is completely free."
Note: the FAQ mentions **Wrike Integrate**, a paid add-on (Business/Enterprise) for no-code integrations with 150+ apps; that is a separate product surface, not a review gate for API apps.

## 4. Time / effort (provider-stated figures only)
Wrike publishes **no setup durations, review timelines, or SLAs**. Active work is the portal registration plus implementing the flow; waiting is zero per the docs. Hard numbers the docs do state:
- Authorization code valid for **10 minutes**.
- Access token TTL: **1 hour** (`expires_in: 3600`).
- Rate limit: "approximately **400 requests per minute**" (per-second based), with 429 + exponential backoff guidance.

## 5. Blockers and nuances
- **Refresh-token rotation**: "The refresh operation returns a new access token and a new refresh token, making the old refresh token invalid." Clients must persist the replacement every time.
- **The `host` trap**: API base URL differs per user's data center; using the wrong host returns 401 "Access token is unknown or invalid." Always build URLs from the token response's `host`.
- **Tokens die with the user**: OAuth and permanent tokens are revoked when the user manually revokes, resets their password, is deactivated, an admin uses "Log in as this user," or an admin changes the password-strength policy (the last invalidates tokens even when the user's password already complies).
- **Redirect URIs must be HTTPS**; `http://localhost` is allowed for development.
- **One account at a time**: API v4 "can access data from only one account at a time"; multi-account users pick an account at consent. Data visible to the app is only what the authorizing user can see under Wrike's sharing settings.
- **Permanent Access Token trap**: available from the App Console ("Get token," password prompt, shown **only once**). It never expires and grants access to all data in the account on that user's behalf; FAQ warns permanent tokens "grant full access to user's account, never expire and are **not intended to be used in multi-user applications**." Fine for testing/single-user tools; a dead end for a SaaS.
- **Background apps**: docs suggest a dedicated "technical" user for UI-less apps, with a manual browser consent run first.
- **Scope minimality** is docs-recommended best practice ("only request the scopes that are necessary"); scopes are requested per authorization call, not fixed at registration.

## 6. Sources (official, actually read)
- https://developers.wrike.com/docs/oauth-20-authorization
- https://developers.wrike.com/docs/faq

## 7. Unverified (docs do not state)
- Whether creating an API App requires an account-admin role (not stated in the developer docs read).
- The full scope catalog (docs give per-method requirements and examples; no single enumerated scope list was found in the pages read).
- Refresh-token lifetime/expiry (no expiry figure stated).
- Any timelines for anything (none published).
- Whether an app registered in one Wrike account can be authorized by users of unrelated accounts, in those exact terms (docs describe the consent flow generically and per-account data access, but do not spell out cross-account authorization eligibility).

---

# AIRTABLE

## 1. The OAuth object and recommended flavor
The object is an **OAuth integration**, registered and managed at **airtable.com/create/oauth**. Registration yields a `client_id`; a **client secret is optional** ("You **should** generate a client secret if you will be issuing token requests from a server"). Recommended flavor for a third-party agent/SaaS: one OAuth integration using the **authorization code flow with PKCE (S256)** — the authorize request takes `code_challenge` / `code_challenge_method: "S256"` — with scopes pre-selected on the integration and requested per authorization. Authorization is two-dimensional: **scopes** (what actions) plus **resources** (which bases/workspaces the user picks on the consent screen).
Not this: personal access tokens (single-user, per the scopes doc's framing) — the guide states plainly: "If you are building an Airtable integration for your product or for many different users… you will need to create an OAuth integration."

## 2. Setup steps in order
1. **Register the integration** at airtable.com/create/oauth → "Register new OAuth integration": give it a **unique name** and a valid **redirect URL**, click "Register integration" (both changeable later). *Human in the UI.*
2. **Configure properties** on the integration management page: permission scopes (the menu of scopes the integration may request), tagline, homepage URL, logo, support email. All fields are shown to users on the authorization dialog. *Human; an agent can prepare values/assets.*
3. **(Server-side apps) Generate a client secret** from the management page ("Generate client secret"). *Human — and see the trap in §5.*
4. **Build the authorization request**: GET `https://airtable.com/oauth2/v1/authorize` with `client_id`, `redirect_uri` (must **exactly match** a registered URI; HTTPS required except localhost), `response_type=code`, `scope` (space-delimited, **at least one**, every scope must be Airtable-defined **and pre-selected on the integration**), `state` (must be a cryptographically generated opaque string), and the PKCE pair. *Agent-automatable (code).*
5. **User authorizes and picks resources**: on the consent screen the user chooses bases/workspaces — specific bases, one or more workspaces (all current **and future** bases in them), all workspaces, or (enterprise admins) an entire enterprise — then clicks Authorize. *Human only.*
6. **Exchange the code**: POST `https://airtable.com/oauth2/v1/token`, form-encoded, **within 10 minutes** of the authorization request. If the integration has a client secret, an `Authorization: Basic base64(client_id:client_secret)` header is **required**; if it has none, that header is **forbidden**. *Agent-automatable.*
7. **Call the API** with `Authorization: Bearer <access_token>`. Token reach = granted scopes ∩ granted bases, and "these may change at any time"; use the List Bases endpoint and the userId-and-scopes endpoint to discover current reach. *Agent-automatable.*
8. **Complete the distribution fields** (§3) so users other than yourself can authorize. *Human.*

## 3. Approval / review
**No review.** The developer docs describe no Airtable review, approval, or marketplace-listing step for OAuth integrations. The only distribution gate is paperwork: "the Privacy policy URL and Terms of service URL must be provided before sharing your OAuth integration with other users — if you have not provided them, other users will not be able to authorize your integration," and the guide's distribution step adds the **support email** to that list. The reference confirms missing ToS/Privacy/Support Email surfaces as an `access_denied` authorization error. No user cap before "approval" exists (there is no approval), and no cost is stated; the help center lists OAuth integrations as available on "All plan types."
One enterprise caveat: a user's enterprise can restrict them from using your integration (listed as an `access_denied` cause; the reference has a dedicated Enterprise Restrictions section, and admins manage "API access for third-party integrations" in the Enterprise Admin Panel).

## 4. Time / effort (provider-stated figures only)
Airtable publishes **no setup durations or review timelines**. Active work: minutes of portal configuration plus your flow code. Waiting: none — self-serve, no queue. Stated hard numbers:
- Token exchange must happen **within 10 minutes** of the authorization request.
- Refresh tokens are "automatically invalidated after **60 days of inactivity**" (help center).

## 5. Blockers and nuances
- **Per-user grant cap: 20 authorizations per user per integration.** "You have reached the maximum access authorizations for this integration" — the user must revoke one before authorizing again. (No limit on how many different integrations a user can authorize.)
- **Refresh-token rotation with teeth** (help center): no API endpoint exists to revoke a refresh token programmatically; tokens are revoked if **reused after rotation**, retried after the duplicate-refresh grace period, or sent with incorrect client credentials — after which the user must re-authorize.
- **Grant codes are single-use, even on failure**: "Individual `code` values can only be used once, even if the request fails."
- **The client-secret one-way door**: the secret is **not recoverable** ("Airtable does not store a copy of your client secret"); regenerating is the only remedy. Worse, "generating a client secret is potentially a breaking change: if you are already sending existing authorization requests, they will fail until you start sending this client secret." And the token endpoint flips its rule based on whether a secret exists (Basic header required vs forbidden).
- **Scope pre-registration**: you can only request scopes already selected on the integration; anything else (empty, duplicated, unregistered) returns `invalid_scope`. Scopes are granular (`data.records:read/write`, `data.recordComments:read/write`, `schema.bases:read/write`, `webhook:manage`, `user.email:read`, `workspacesAndBases:read`, …) and tiered: basic scopes for all users, **enterprise-member scopes** (e.g. `workspacesAndBases:write`, `enterprise.groups:read`), and **enterprise-admin scopes** (SCIM, audit logs, `enterprise.user:write`, etc.) that most users simply cannot grant.
- **Consent is resource-picking, and grants move**: a token's base access can change at any time (user edits the grant; if the user loses collaborator access to a base, the token loses it too, and access "can automatically resume" if they're re-added). Revoking an authorization kills everything granted through that token at once.
- **Tokens are opaque**: "do not rely on tokens having a particular length or format. Changes to the token format… are not considered to be breaking changes."
- **Redirect mismatch fails silently**: if `redirect_uri` doesn't match one registered for the client_id, "you will not receive an error response."

## 6. Sources (official, actually read)
- https://airtable.com/developers/web/guides/oauth-integrations
- https://airtable.com/developers/web/api/oauth-reference (authorization routes, parameters, error codes, token creation — first half read in full; see gap below)
- https://airtable.com/developers/web/api/scopes
- https://support.airtable.com/articles/6811769763-third-party-integrations-via-oauth-overview (official help center; used for the 20-authorization cap, base-grant mechanics, and refresh-token invalidation rules)

## 7. Unverified / gaps
- **Access-token TTL**: the OAuth reference's "Token expiry & refresh tokens," "Resources," and "Enterprise Restrictions" sections could not be finished in this session (page continuation hit a browser rate limit and was not retried), so the exact access-token lifetime figure is unverified here; refresh behavior above comes from the official help center instead.
- Whether Airtable offers any optional public listing/marketplace for OAuth integrations: not described anywhere in the developer docs read; the documented distribution path is direct authorization links only.
- Any cost beyond plan availability (help center says all plan types; no developer fee is mentioned).
- No stated effort estimate for building the flow (Airtable provides an example repo, github.com/Airtable/oauth-example, but no time figures).

---

**Cross-cutting note for the library:** Wrike and Airtable are both "no gate" providers — register, configure, and users can connect — which contrasts sharply with the review/cap-heavy providers already in the library. Their difficulty lives in token mechanics instead: Wrike's rotating refresh tokens + per-user data-center `host`, and Airtable's 20-grant cap, single-use codes, rotation reuse-detection, and the unrecoverable-secret trap.