Research complete. Both reports below, based only on the providers' official documentation (vercel.com/docs and docs.netlify.com), read-only.

---

# VERCEL

**Headline:** Vercel has *two* different OAuth-flavored objects that developers routinely confuse: the Marketplace **Integration** (the one that grants API access to users' projects/teams via an install flow) and the **Sign in with Vercel "App"** (a standard OAuth 2.0 / OIDC client for identity). They have different consoles, different credentials, and different token behavior.

## 1. What the OAuth object is called, and the recommended flavor

- **Integration** — created in the dashboard via **Integrations → Integrations Console → Create** (the "Create Integration" form). This is the object for a third-party product whose users connect their Vercel accounts/projects so the product can call the Vercel REST API on their behalf. Creating one assigns a **client (integration) ID and secret**. Two types:
  - **Connectable account integration** — user connects Vercel to their *existing* account on your platform; install uses a **Redirect URL + OAuth flow**, and you declare **API Scopes** (per-resource: `deployment`, `project`, `project-env-vars`, `domain`, `team`, `user`, `integration-configuration`, `drains`, `billing`, etc., each at None / Read / Read-Write). This is the self-serve flavor and the recommended one for an AI agent/SaaS.
  - **Native integration** — a two-way, partner-built integration where products are provisioned and **billed through Vercel** (Marketplace API + your own "integration server"; no separate account on your site). **Not self-serve**: you must first sign up as a native integration provider (vercel.com/marketplace/program) and coordinate with Vercel in a shared Slack channel.
- **App (Sign in with Vercel)** — created under **Team Settings → Apps → Create**. A standard OAuth 2.0/OIDC client against Vercel's Identity Provider. Scopes are identity-only: `openid` (required), `profile`, `email`, `offline_access` (issues a refresh token). The access token can also carry team/project **Permissions** (e.g. Deployments Read/Write), but the docs state those API permissions are **"currently in private beta."** Recommended only when the product's need is "log in with Vercel"; for dependable programmatic API access today, the docs point you to the Integration model.

## 2. Setup steps in order

**Connectable account integration (agent vs human):**
1. *(Human, dashboard)* Dashboard → select team → **Integrations** → **Integrations Console** → **Create**.
2. *(Human)* Complete the Create Integration form in full: Name, URL Slug, Developer (legal name), contact email (private), support email (public), short description, logo (1:1, min 256px), category, Website, Documentation URL, **EULA URL**, **Privacy Policy URL**, overview, feature media (1–8 images, 16:9, 1920×1080), **Redirect URL**, API Scopes, optional Webhook URL and Configuration URL; accept Vercel's terms to publish.
3. *(Human/agent)* Build your Redirect URL endpoint: it must exchange the install `code` for a token, render the install UI in a popup, let users connect projects, and redirect back to Vercel on completion.
4. *(Agent, fully automatable)* Exchange the code: `POST https://api.vercel.com/v2/oauth/access_token` with form-encoded `client_id`, `client_secret`, `code`, `redirect_uri`. Response: `access_token`, `installation_id`, `user_id`, `team_id` (non-null `team_id` = installed on a team; append `teamId` to API calls).
5. *(Human)* Optionally pursue public Marketplace listing (see §3).

**Sign in with Vercel App (for completeness):** Team Settings → Apps → Create (name; slug auto-generated; optional logo) → generate a client secret (max **two active secrets** at a time, for rotation) → choose a client authentication method (`client_secret_basic`, `client_secret_post`, `client_secret_jwt`, `private_key_jwt`, `oidc_token`, or `none` for public clients) → register **Authorization Callback URLs** → enable scopes/permissions on the Permissions page → implement authorize (`https://vercel.com/oauth/authorize`, PKCE S256 in the official guide) → token exchange at `https://api.vercel.com/login/oauth/token`. Dashboard steps are human; the flow code is agent-automatable.

## 3. Approval / review

- **Connectable integrations:** No pre-approval gate to distribute. Per the docs: once created, the integration gets a **Community badge** and "will be available for external users to download" — installable via your own site or a Deploy Button, but it does **not** appear in the public Marketplace.
- **Public Marketplace listing (connectable):** gated. Requirements stated in the docs: **at least 500 active installations** (500 accounts), compliance with the review guidelines/approval checklist, then **email integrations@vercel.com** to request review.
- **Native integrations:** partner-gated from the start (provider signup + shared Slack channel), plus a product form, and the published approval checklist ends with: email integrations@vercel.com → "Vercel reviews your integration and provides feedback or requests additional testing" → "Schedule a final walkthrough call."
- **Sign in with Vercel Apps:** the docs describe **no review or approval step at all** — create the app and users can authorize (with a consent page on first sign-in). Whether any unpublished review exists is not stated.
- **Cost:** no listing fee or review cost is stated anywhere in the docs read. **Timeline:** no review duration is stated anywhere (see §7).

## 4. Time / effort (provider-stated figures only)

- Active work: one long dashboard form + building the redirect/install endpoint and token exchange (integrations), or a short app-creation form + standard OIDC code (Apps). The docs quantify **no** setup durations.
- Stated time figures: the integration install `code` is **valid for 30 minutes** and **single-use**. Sign in with Vercel **access token lasts 1 hour**; **refresh token lasts 30 days and rotates each time it's used**.
- Waiting: no stated review timelines for any path; the 500-install threshold is the real elapsed-time gate for Marketplace listing.

## 5. Blockers and nuances

- **The 500-install wall:** you cannot even *request* a public Marketplace listing for a connectable integration until you have 500 active installations — distribution before that is your own site/Deploy Button only, under a "Community" badge that signals "not in the marketplace."
- **Native ≠ self-serve:** native integrations require becoming a Vercel marketplace provider first; an agent cannot just create one.
- **Two credential systems:** an Integration's client ID/secret and an App's client ID/secret are different objects in different consoles; using one where the other is expected fails (the authorize endpoint for Apps is `vercel.com/oauth/authorize`, while integrations receive their `code` on the install Redirect URL).
- **Integration tokens:** the docs call the exchanged integration access token **"long-lived"** and describe **no refresh token and no expiry** for it (see §7). Sign-in App tokens are the opposite: 1-hour access tokens, rotating 30-day refresh tokens, and refresh requires the `offline_access` scope.
- **Sign-in API permissions are private beta:** the granular team/project permissions an access token can carry are "currently in private beta," so the App route is not yet a general-purpose API-access path.
- **Ownership trap (integrations):** if the integration's owner(s) leave the owning team, the integration is flagged **disabled** (API calls fail with `403` / `integration_configuration_disabled`, most webhooks pause); the team has **30 days** to transfer ownership or the integration is deleted.
- **Scopes are set at the integration level** in the console (None/Read/Read-Write per resource), not requested per-authorize-call like classic OAuth scopes; scope changes surface to installs via the `integration-configuration.scope-change-confirmed` webhook.
- **Install-UX gotchas (from the listing requirements):** if your app sets `Cross-Origin-Opener-Policy`, it must be `unsafe-none` or the Vercel dashboard can't monitor the install popup; localhost redirect URLs are allowed for testing.
- **Sign-in access switch (Apps):** an app can be restricted from "anyone with a Vercel account" to "members of the owning team"; Vercel re-checks this at authorize **and at token refresh**, and denies refreshes (`access_denied`) after a user leaves the team — existing app sessions are *not* ended.

## 6. Sources (official docs actually read)

- https://vercel.com/docs/integrations
- https://vercel.com/docs/integrations/create-integration
- https://vercel.com/docs/integrations/create-integration/submit-integration
- https://vercel.com/docs/integrations/create-integration/approval-checklist
- https://vercel.com/docs/integrations/create-integration/vercel-api-integrations
- https://vercel.com/docs/sign-in-with-vercel
- https://vercel.com/docs/sign-in-with-vercel/getting-started
- https://vercel.com/docs/sign-in-with-vercel/scopes-and-permissions
- https://vercel.com/docs/sign-in-with-vercel/manage-from-dashboard

## 7. Unverified

- Any review **timeline** (Marketplace listing or native approval) — not stated.
- Any **cost/fee** — not stated.
- Whether integration access tokens ever **expire** or can be refreshed — docs say "long-lived" and give no expiry/refresh mechanism.
- Whether Sign in with Vercel Apps face any review before arbitrary users can authorize — no review is mentioned; absence is not confirmed as policy.
- How to get into the **private beta** for App API permissions, and its scope — not stated.
- User/install **caps** before approval, other than the 500-install listing threshold — none stated.

---

# NETLIFY

**Headline:** Netlify *does* have a real, self-serve OAuth program for third-party apps — but it is minimal. You register an "OAuth application" in your user settings, get a client ID and secret, and send users to Netlify's authorize endpoint. The official docs describe **no scopes system, no review, no marketplace, and no token expiry rules**.

## 1. What the OAuth object is called, and the recommended flavor

- The object is an **OAuth application**, registered under **User settings → Applications → OAuth applications**. Per the docs: "When you register an application, Netlify will provide a client ID and secret you can use in your application settings."
- The docs are explicit about when to use it: "If you're making a public integration with Netlify for others to enjoy, **you must use OAuth2**. This allows users to authorize your application to use Netlify on their behalf without having to copy/paste API tokens or touch sensitive login info."
- There is effectively **one flavor**: the OAuth application, used with the OAuth2 end-user authorization endpoint **`https://app.netlify.com/authorize`**. The alternative documented mechanism, **Personal Access Tokens (PATs)**, is for your *own* scripts/CLI use ("Use your PAT for manual authentication in shell scripts"), not for a multi-user product. So for a third-party AI agent/SaaS whose users connect their own Netlify accounts, the OAuth application is the recommended (and only documented) flavor.

## 2. Setup steps in order

1. *(Human, portal)* In the Netlify UI: avatar → **User settings** → **Applications** → **OAuth applications** → register a new application. Netlify issues a **client ID and client secret**. (The docs' user-settings page confirms where registration lives and what you receive; it does not enumerate the registration form's fields.)
2. *(Agent, automatable)* Build the connect flow: direct users to `https://app.netlify.com/authorize` so they can authorize your application; then call the Netlify API (`https://api.netlify.com/api/v1/…`) with the resulting token as `Authorization: Bearer <token>`.
3. *(User-side, documented)* Users who authorized your app see it under **User settings → Applications → Authorized applications** and can revoke it there ("Options → Revoke access").

That is the full documented setup. The docs do **not** document, on docs.netlify.com: the token endpoint, grant types, a redirect-URI registration field, or a consent-screen configuration.

## 3. Approval / review

- **None documented.** The docs describe no review, approval, verification, or listing step before real users can connect an OAuth application, and no marketplace/listing program for OAuth apps at all.
- **No user caps** before approval are stated, because there is no approval concept in the docs.
- **Cost:** none stated.

## 4. Time / effort (provider-stated figures only)

- Active work: a single registration in user settings plus implementing the authorize/API calls. The docs state **no setup durations and no waiting periods** — there is nothing to wait for in the documented flow.
- The only numeric limits stated on the API side: **500 requests per minute** for most API requests; API deploys limited to **3 per minute and 100 per day**; list responses paginate at **100 items per page**.

## 5. Blockers and nuances

- **No documented scopes:** docs.netlify.com describes no scope parameter or permission granularity for OAuth applications — nothing in the docs lets a developer request read-only or site-limited access; authorization appears to be all-or-nothing at the account level (the docs simply don't specify, see §7). This is the biggest surprise for developers coming from GitHub/Google-style scoped OAuth.
- **No documented token lifecycle:** the API get-started page never states access-token expiry, refresh tokens, or grant types for OAuth apps. (By contrast, PATs *do* get a documented, user-chosen **expiration date** at creation — a control the OAuth-app docs don't mention.)
- **Docs are thin and point elsewhere:** the get-started page defers flow details to "our blog post on integrating with Netlify" — i.e., the canonical docs site itself does not contain a complete OAuth implementation guide.
- **Rate limits bite deploy-heavy agents:** 3 deploys/minute and 100 deploys/day through the API is a hard ceiling for an agent that deploys on behalf of many users (higher limits only "please contact us").
- **Revocation is user-controlled and silent:** users can revoke your app any time from Authorized applications; the docs describe no webhook or notification to the developer when that happens.
- **SAML wrinkle (PATs only):** PAT creation has an "Allow access to my SAML-based Netlify team" toggle; the OAuth-app docs mention no equivalent consideration.

## 6. Sources (official docs actually read)

- https://docs.netlify.com/api-and-cli-guides/api-guides/get-started-with-api/ (Authentication section: OAuth2, registration location, authorize endpoint, rate limits)
- https://docs.netlify.com/manage/accounts-and-billing/user-settings/ (Applications section: OAuth applications, PATs, Authorized applications/revocation)

## 7. Unverified (material items docs.netlify.com does not state)

- Grant types supported (authorization code, implicit/ticket, etc.), the token endpoint URL, and PKCE support.
- Whether OAuth access tokens expire, and whether refresh tokens exist.
- Whether any scopes/permissions model exists for OAuth apps.
- The registration form's exact fields (e.g., whether/where a redirect URI is registered).
- Any review, verification, listing, or fees — none are mentioned anywhere in the docs read; treat "no review" as "no review documented," not a stated policy.
- What exactly a user sees on the authorize/consent screen.

---

**Cross-cutting note for the library:** Vercel and Netlify sit at opposite ends of the spectrum covered so far. Vercel is a gated, two-system program (self-serve Community distribution, a 500-install wall before Marketplace listing review, and a separate partner-only native track). Netlify is the opposite extreme: instant self-serve registration with no documented review, scopes, or token rules at all, the surprises are what's *missing* from the docs.