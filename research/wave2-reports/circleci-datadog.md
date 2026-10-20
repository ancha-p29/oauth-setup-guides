Research complete. Here are the two structured reports, based only on each provider's official documentation.

---

# CircleCI

**Headline finding:** CircleCI *is* an OAuth provider, but not in the classic "register an app in a developer portal" way. It offers **OAuth 2.0 with Dynamic Client Registration (DCR) and mandatory PKCE**, designed for tools that run on the user's own machine. There is no portal, no client secret, and no review. The documented flow is loopback-only, which makes it a poor fit for a hosted SaaS redirect flow and a good fit for CLI/desktop/local agents.

## 1. The OAuth object and recommended flavor

- The object is a **dynamically registered OAuth client**. Your tool registers itself once via `POST https://app.circleci.com/oauth/register`; the response contains only a `client_id` (`token_endpoint_auth_method: "none"`). Per the docs: "No pre-shared client secret or manual token creation in the CircleCI web app is needed," and "Because no client secret is issued, this value does not need to be treated as sensitive. PKCE protects the flow instead."
- The flow's output is not a generic access token: it is **a standard CircleCI personal API token** (prefix `ccipat_`) minted on behalf of the user, "scoped to the access level (Read, Write, or Admin) chosen by the user during authorization" (Managing API Tokens).
- Recommended flavor for a third-party tool: **this DCR + authorization-code + PKCE (S256) flow is the only OAuth flavor CircleCI documents for third parties.** Optional RFC 9126 Pushed Authorization Requests (`POST /oauth/par`) are also supported. The fallback non-OAuth flavors are manually created **personal API tokens** (always full user permissions) and **project API tokens** (Status / Read Only / Admin) — both require the user to create and paste a token by hand.

## 2. Setup steps in order

1. **Register the client (once per tool).** `POST https://app.circleci.com/oauth/register` with JSON: `client_name`, `redirect_uris`, `grant_types: ["authorization_code"]`, `response_types: ["code"]`. "No authentication is required, and no client secret is issued." Returns `201 Created` with the `client_id`. *Fully automatable by an agent — no human, no account, no portal.*
2. **Generate PKCE values and run a local redirect listener.** `code_verifier` (43–128 chars), `code_challenge = base64url(sha256(verifier))`, plus a random `state`. S256 is mandatory: "CircleCI requires the S256 challenge method — plain code challenges are not accepted." *Automatable.*
3. **Send the user to the authorization URL** (`https://app.circleci.com/oauth/authorize?response_type=code&client_id=…&redirect_uri=…&code_challenge=…&code_challenge_method=S256&state=…`). `state` is required; omitting it returns a 400 error. *Human step:* the user logs in (if needed), **chooses the access level (Read / Write / Admin)**, separately authorizes any SSO-enabled organizations, and clicks **Allow**.
4. **Exchange the code** at `POST https://app.circleci.com/oauth/token` (form-encoded: `grant_type=authorization_code`, `code`, `client_id`, `redirect_uri`, `code_verifier`). The code is "single-use and short-lived." Response: `access_token`, `token_type: Bearer`, `expires_in: 7776000`. *Automatable.*
5. **Use the token** as a Bearer token on API v3 or v2 calls (v2 also accepts the `Circle-Token` header). *Automatable.*

Prerequisite stated by the guide: "A CircleCI account. You must be a member of an organization."

## 3. Approval / review

- **None.** The DCR guide describes no CircleCI review, approval, listing, or marketplace step at any point. Client registration is unauthenticated and immediate; the only consent gate is the end user clicking Allow. There are **no user caps before approval** because there is no approval.
- The only per-organization gate is user-driven: for SSO-enabled orgs, the user must explicitly authorize each org on the consent screen (see §5).

## 4. Time / effort (provider-stated figures only)

- Active work: one registration POST plus implementing a standard auth-code + PKCE flow and a loopback listener. The docs give **no setup-duration estimates**.
- Waiting: **none** — no review queue exists.
- Recurring effort is dictated by token lifetime: tokens last **90 days** (`expires_in` 7,776,000 seconds), and "This flow does not issue refresh tokens," so every user must re-run authorization roughly every 90 days. CircleCI emails personal-token expiry warnings **30 days, 15 days, seven days, and one day** before expiry (Managing API Tokens).

## 5. Blockers and nuances

- **Loopback-only redirect URIs — the big one for SaaS.** "The `redirect_uri` must be a loopback address (`127.0.0.1`, `::1`, or `localhost`) with a port in the range 1025–65535 and no query parameters." The documented flow is therefore built for local tools; the docs describe **no hosted (https) redirect option** for this flow.
- **No refresh tokens.** The 90-day token simply expires; re-authorization is the renewal mechanism.
- **Coarse scopes.** There are no granular API scopes — only the three access levels Read / Write / Admin, chosen by the *user* at consent time, not requested by the app. The level is "a ceiling on the token, not a grant of new permissions": a Viewer who selects Admin still gets a read-only token; org management/billing require the user to actually hold the Organization Admin role.
- **SSO org trap.** Leaving an SSO org unauthorized "does not fail the flow. The token is still issued, but it cannot access that org's resources." Access can later be granted/revoked from the **OAuth Clients** page in user settings without redoing the flow.
- **Re-registration multiplies clients.** "Register only once per tool… Re-registering creates a new, separate client."
- Tokens are visible to and revocable by the user from their Personal API Tokens page; API tokens (any kind) "cannot be modified after they have been created" — change means delete and recreate (rotation).
- Contrast with manual tokens: manually created personal tokens "grant full read and write permissions" and take a user-set expiry "no more than one year in the future"; project tokens' Read Only and Admin scopes work with API v1 only, and project tokens are not supported on API v2 (API Developer's Guide).

## 6. Sources (circleci.com/docs, read directly)

- OAuth 2.0 API access with Dynamic Client Registration — https://circleci.com/docs/guides/toolkit/oauth-dynamic-client-registration/ (read through the token-use and expiry/rotation sections)
- Managing API tokens — https://circleci.com/docs/guides/toolkit/managing-api-tokens/
- CircleCI APIs overview — https://circleci.com/docs/guides/toolkit/api-intro/ (via docs search excerpt)
- API v2 developer's guide (project-token support note) — https://circleci.com/docs/guides/toolkit/api-developers-guide/

## 7. Unverified

- Whether non-loopback (hosted SaaS) redirect URIs are accepted anywhere in CircleCI's OAuth implementation — the docs state the loopback rule and describe no alternative.
- The exact authorization-code lifetime ("short-lived" is all the docs say).
- Any rate limits or quotas on the registration endpoint or on DCR-issued tokens.
- Whether CircleCI runs any abuse review/blocking of DCR clients behind the scenes (no review is documented).
- Any partner/marketplace program for OAuth tools (the docs cover an orb registry elsewhere, but nothing ties it to this OAuth flow).

---

# Datadog

**Headline finding:** Datadog *does* have a real OAuth 2.0 program — but it is **gated to approved Technology Partners building official, published Datadog integrations**. The docs state it outright: "OAuth is available only to approved Technology Partners developing official Datadog integrations. Standalone OAuth clients are not supported." Everyone else (end users, ad-hoc tools, non-partners) uses **API keys + application keys per organization**, which remain the default auth model.

## 1. The OAuth object and recommended flavor

- The object is an **OAuth client attached to an integration listing** in the Datadog **Developer Platform** (Integrations > Developer Platform > Publishing). The client is configured per listing with an OAuth Client Name, Onboarding URL, Redirect URIs, and a generated client secret.
- Recommended (and only) flavor for a third-party SaaS: **an API-based integration with OAuth** ("API with OAuth" configuration method), built after joining the **Datadog Partner Network** and getting access to a **partner developer organization**. API-based integrations are "best suited for SaaS-based vendors that can host a service to interact with the Datadog API," and OAuth is mandatory for them: "Your integration must implement OAuth for authorization"; "If your platform is incompatible with OAuth, reach out to the Datadog Ecosystems team for an exception."
- Non-partner alternative: **API key + application key** per org. Per the Authorization Scopes doc: "OAuth clients → Can only be assigned authorization scopes (limited set). Scoped application keys → Can be assigned any Datadog permission."

## 2. Setup steps in order

Prerequisites (human/organizational): **join the Datadog Partner Network** (Technology Partner track — per the Marketplace docs you apply and must be approved before building a Marketplace offering), obtain a **partner developer organization** / sandbox, and ensure "Your product must be generally available" and "must submit telemetry to Datadog."

1. **Create a listing** in Integrations > Developer Platform > Publishing > **New Listing**: offering type, name, vendor details, SVG wordmark logo, unique integration ID. *Human, in the Datadog UI.*
2. **Implement OAuth 2.0 in your product** (authorization code flow; PKCE supported). Choose the minimum scopes from Datadog's published scope list (e.g., `apm_read`, `synthetics_read`…). *Agent can write essentially all of this code.*
3. **Add OAuth details to the listing** — Configuration Method tab > **API with OAuth**: OAuth Client Name (should match the integration name), **Onboarding URL**, one or more **Redirect URIs**, click **Generate OAuth Client Secret** for testing credentials, select minimum scopes, Save. *Human in UI; the secret "is not displayed again," so it must be recorded at generation.*
4. **Test with "Test Authorization"** — this simulates a user clicking **Connect Accounts** on the integration tile, redirecting to your Onboarding URL with `domain` and `site` parameters; complete the flow against your partner developer organization. "Until your integration is published by Datadog, you can only test OAuth within your Datadog partner developer organization." *Mixed: agent can drive the flow; a human partner-org account is required.*
5. **Run the flow correctly at runtime:** the flow **must be initiated from Datadog** (tile → Onboarding URL) so Datadog supplies `domain` (region, e.g. `datadoghq.com`, `ap1.datadoghq.com`) and `site` (custom subdomain; handshake-only). Exchange the code at `https://api.<domain>/oauth2/v1/token` (reference: `POST /api/v2/oauth2/token`) to get access + refresh tokens; call APIs at `https://api.<domain>/…`.
6. **If you submit data** (metrics/logs/events), during the handshake request the `api_keys_write` scope and call **`POST /api/v2/api_keys/marketplace`** to create an API key on behalf of the user — "An API key, created through this endpoint, is the only way to send data into Datadog through OAuth."
7. **Submit the listing for review** with production URLs replacing test URLs; "Upon submission in the Developer Platform, production client credentials are provided" (again, secret shown once).

## 3. Approval / review

- **Two gates.** (a) **Partner gate:** OAuth exists only for approved Technology Partners; there is no self-serve OAuth client creation for the public. (b) **Listing review:** "Submit your integration for review. After approval, Datadog publishes it," and "The Datadog team will review and respond with feedback" (Build an Integration). Your integration appears as a tile only after publication.
- **Pre-approval cap (functional, not numeric):** until published, the OAuth client can only be authorized from your own partner developer organization — "Until an OAuth client is published, you can only authorize the client from the partner developer organization it was created in" (attempting otherwise yields `invalid_request - Invalid client_id`). No numeric user cap is stated in the docs.
- **Timeline:** the docs pages read state **no review duration**. (See Unverified.)
- **Cost:** no fee for building/publishing an integration is stated in the docs. The docs distinguish free out-of-the-box Integrations from the **Marketplace**, "a commercial platform where Technology Partners can sell offerings" with "additional fees" billed to customers — partner-side program fees are not stated in the docs.

## 4. Time / effort (provider-stated figures only)

- Sandbox provisioning: "Creating a developer sandbox may take **up to one or two business days**" (Marketplace docs).
- Token lifetimes: **access tokens have a TTL of 1 hour**; **refresh tokens are long-lived with no expiration (TTL of infinity)** for Marketplace OAuth clients, ending only on user/partner revocation.
- Everything else — partner application turnaround, review duration, total build time — is **not quantified in the docs**. Active work is dominated by building the listing content (overview, setup/uninstall instructions, data definitions, at least one required dashboard, support details) plus the OAuth implementation; dashboards are "required for all integrations."

## 5. Blockers and nuances

- **The partner gate is absolute.** No partner status → no OAuth, period. Non-partners fall back to asking users for API/app keys.
- **Flow must start inside Datadog.** "Your product must direct users to initiate the OAuth flow from Datadog, so Datadog can provide the `domain` and `site` parameters to your onboarding URL. These parameters are not included if the flow starts from your product." A "Connect Datadog" button hosted purely in your own product breaks the documented model.
- **Multi-tenancy restriction:** "Datadog does not support multi-tenanted applications where customers authorize using individual subdomains; instead, authorization is supported only through a single domain."
- **Data submission detours through API keys.** OAuth tokens alone cannot ingest data; you must mint a per-org API key during the handshake. "Only one API key can exist per Datadog organization" via the marketplace endpoint, its value "is shown once after creation," and if lost you must revoke it and have the user reinstall/re-authorize to generate a new one. The required `api_keys_write` scope is described as a *private* scope; if you can't set it, Datadog tells you to email marketplace@datadoghq.com.
- **Secrets are fragile.** Test secret ≠ production secret (production credentials are issued at submission). "Only one secret can be active at a time," and rotation is **manual, by email**: "If your secret was leaked and needs to be rotated, contact ecosystems@datadog.com." Lost published secret → also ecosystems@datadog.com.
- **Region handling is on you.** API endpoints differ per site (e.g. `api.datadoghq.eu` vs `api.datadoghq.com`); you must build URLs from the `domain` parameter and must not include the customer's subdomain in API calls.
- **Refresh tokens effectively never expire** (until revocation), so secure storage and revocation handling matter more than renewal logic; if the partner revokes, "the user must reauthorize the integration to generate new refresh and access tokens."
- **Scope ceiling:** OAuth clients can only receive the published authorization-scope set (least-privilege guidance is explicit), unlike scoped application keys, which can carry any Datadog permission.

## 6. Sources (docs.datadoghq.com)

Read in full:
- Create an API-based integration — https://docs.datadoghq.com/extend/integrations/api_integration/
- Build an Integration with Datadog — https://docs.datadoghq.com/extend/integrations/build_integration/

Read via official docs excerpts:
- OAuth2 Authorization Endpoints Reference (token TTLs, marketplace API-key endpoint) — https://docs.datadoghq.com/extend/authorization/oauth2_endpoints/
- OAuth2 in Datadog (flow steps, domain/site handling, API-key creation step) — https://docs.datadoghq.com/extend/authorization/oauth2_in_datadog/
- Authorization Scopes (OAuth clients vs scoped application keys; scope list) — https://docs.datadoghq.com/api/latest/scopes/
- Datadog Integrations overview (submit-for-review step; Integrations vs Marketplace) — https://docs.datadoghq.com/extend/integrations/
- Build a Marketplace Offering (partner approval prerequisite; sandbox "up to one or two business days") — https://docs.datadoghq.com/developers/marketplace/

## 7. Unverified

- **Review/publish timeline:** no duration is stated anywhere in the docs pages read. (Datadog's separate partner-program website, outside docs.datadoghq.com, quotes "Integrations typically take 4-6 weeks to be published" — not a docs statement, not relied on here.)
- Partner Network application criteria and approval turnaround.
- Any partner program or Marketplace revenue-share fees (docs mention customer-side "additional fees" only).
- Whether an unpublished client has any numeric install/user cap beyond the partner-org-only authorization restriction (none stated).
- Whether OAuth clients not tied to a listing (e.g., for Datadog Apps / internal org tools) can be self-created by ordinary org admins — the scopes doc references OAuth clients for "your Datadog Apps," but no self-serve creation flow is documented in the pages read.

---

**One gap to note:** a mid-research rate limit prevented re-reading the final paragraphs of CircleCI's DCR guide; its expiry/rotation content was corroborated against CircleCI's Managing API Tokens page (90-day tokens, delete-and-recreate rotation, expiry email schedule), which I read in full.