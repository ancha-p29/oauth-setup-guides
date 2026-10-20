Research complete. Below are the two structured reports, sourced only from official documentation I read directly (Atlassian: developer.atlassian.com; Klaviyo: developers.klaviyo.com). Read-only research; no accounts created, nothing submitted.

---

# CONFLUENCE (Atlassian Confluence Cloud)

**Headline:** Confluence uses Atlassian's shared OAuth 2.0 (3LO) program. There is no Atlassian review before external users can connect once you flip the "sharing" toggle, but unapproved integrations carry a "not yet reviewed by Atlassian" warning, and the grant-type choice at app creation (account-level vs resource-level) is a real design decision.

## 1. What the OAuth app is called / recommended flavor

- The object is an **OAuth 2.0 integration** (an "OAuth 2.0 (3LO) app"), created and managed in the **Atlassian Developer console** (Create → **OAuth 2.0 integration**). It yields a **Client ID** and **Secret** (Settings page).
- **Recommended flavor for a third-party SaaS/AI agent:** one single, distributable 3LO app with the **Confluence API** added under Permissions, the minimum Confluence scopes configured, `offline_access` included so you get refresh tokens, and **sharing enabled** under Distribution. Atlassian explicitly steers vendors this way: "Build a single, distributable 3LO app for your integration."
- At app creation you choose one of two grant types:
  - **Account-level grant:** "The user's consent applies at the Atlassian account level. A single grant can cover access to multiple sites where your app is installed, subject to the scopes you request."
  - **Resource-level grant:** "The user's consent is limited to the specific site(s) they select on the consent screen. The resulting tokens can only be used with those selected site(s)."
- Alternatives (not the third-party-connect path): **Forge apps** (run on Atlassian's platform and use OAuth 2.0 with managed APIs, for apps embedded in Confluence); **Connect apps** (HTTPS/JWT — the Confluence docs note "OAuth 2.0 (3LO) isn't supported for Connect apps"); **API tokens / basic auth** (supported but not recommended, and see the compliance trap in §5).

## 2. Setup steps in order

1. **Create the app** in the Developer console: Create → OAuth 2.0 integration, enter a name, agree to Atlassian's developer terms, and choose the **grant type** (account-level vs resource-level). *(Human only — console + terms.)*
2. **Add the Confluence API and scopes:** Permissions → Add next to Confluence API → Configure → select scopes on the **Classic scopes** and **Granular scopes** tabs; Save. *(Human in console; an agent can prepare the scope list from each API operation's "OAuth scopes required" field.)*
3. **Enable 3LO and set the callback:** Authorization → Add/Configure next to OAuth 2.0 (3LO) → enter the **Callback URL**; Save changes. The runtime `redirect_uri` must match this URL. *(Human in console.)*
4. **Copy Client ID and Secret** from Settings. *(Human — credential custody.)*
5. **Implement the authorize redirect** *(agent-automatable code)*: send the user to `https://auth.atlassian.com/authorize?audience=api.atlassian.com&client_id=…&scope=…&redirect_uri=…&state=…&response_type=code&prompt=consent`. Scopes in the URL "Only choose from the scopes that you have already added to the APIs for your app in the developer console." Include `offline_access` in the scope parameter to receive a refresh token.
6. **User consents** *(human end user)*: Atlassian login if needed, then the consent screen; for site-restricted grants the user must select the site.
7. **Exchange the code** *(agent)*: `POST https://auth.atlassian.com/oauth/token` with `grant_type=authorization_code`, `client_id`, `client_secret`, `code`, `redirect_uri`. Response: `access_token`, `expires_in`, `scope`.
8. **Discover sites** *(agent)*: `GET https://api.atlassian.com/oauth/token/accessible-resources` with the access token → returns each site's `id` (**cloudid**), URL, and granted scopes.
9. **Call the Confluence API** *(agent)*: requests go via `https://api.atlassian.com/ex/confluence/{cloudid}/{api}` (e.g. `/rest/api/space`) with `Authorization: Bearer <access_token>` — **not** via the site's `*.atlassian.net` domain.
10. **Refresh** *(agent)*: `POST` the token URL with `grant_type=refresh_token`, `client_id`, `client_secret`, `refresh_token`. Refresh tokens are **rotating**: "You receive a new refresh token as well and the refresh token you used for the request is disabled" — you must replace the stored token every time.
11. **Enable distribution** *(human)*: Distribution → turn on the **Enable sharing** toggle, then copy the Authorization URL(s) from Authorization → Configure and send the link to your users.

## 3. Approval / review

- **No review is required before real external users can connect.** "When you create an OAuth 2.0 (3LO) app, it's private by default. This means that only you can install and use it. If you want to distribute your app to other users, you must enable sharing." Once sharing is on, any user can authorize via your Authorization URL.
- **Pre-approval cap:** the only cap documented is the private default (creator only). The docs state **no numeric user/install cap** for a shared app (see §7).
- **The warning:** "Users trying to install an unapproved OAuth 2.0 integration are warned that the app has not yet been reviewed by Atlassian."
- **Optional approval:** "To get your integration reviewed and approved, follow the steps on Listing a third party integration on the Atlassian Marketplace. Note, you don't need an informative Atlassian Marketplace listing to submit your integration for approval." A third-party integration is one where "no artifact is required to be installed on the customer's Atlassian instance." On the Publish-a-new-app page you select **"My app isn't directly installable"**; the App Key "can't be altered once you set it." The Marketplace team reviews it: "In order to test the integration we will need access to a test instance of your software so we can run through the set up and configuration."
- **Listing shape:** "Although OAuth 2.0 (3LO) apps can be listed on the Atlassian Marketplace, they will appear as informational listings only, with limited Marketplace features," and "Enabling sharing doesn't make your app available on the Atlassian Marketplace."
- **Cost:** no fee for 3LO integration approval/listing is stated anywhere in the docs read (see §7).

## 4. Time / effort (provider-stated figures only)

- Atlassian states **no setup duration** for creating/configuring a 3LO app, and **no approval duration** on the third-party-integration page itself.
- Marketplace listing guide (the process the 3LO doc points to): "**It usually takes 5-10 business days for our team to start a review**" and "Each app is reviewed by a Marketplace support team member before being publicly listed… **You should hear from a support member within 5 - 10 business days**."
- Refresh-token configuration table (defaults): "**Inactivity expiry time | 90 days** — A rotating refresh token expires if the user is inactive for this period. Each new rotating refresh token resets the inactivity expiry time and allocates another 90 days"; "**Reuse interval or leeway | 10 minutes**" (breach detection doesn't apply to repeat exchanges inside this window). FAQ restates: each new rotating refresh token "is valid for 90 days."
- Access-token lifetime: the token response includes `expires_in` ("expiry time of access_token in second"), but the pages read **do not state the numeric value** — see §7. Authorization-code lifetime is likewise not stated.

## 5. Blockers and nuances

- **Grant type is chosen at creation** (account-level vs resource-level). Known-issue friction for resource-level grants: "The consent screen currently requires the user to select the site where they want to install the app… To do this, the user would need to repeat the installation process multiple times."
- **One grant per app per Atlassian account:** "Only one grant exists per app for a given Atlassian account"; a new consent overrides the old grant's scopes, and "users who previously consented to the scopes will need to re-consent to the new scopes" when you change scopes in the console.
- **Scope ceiling is double:** you may only request scopes pre-added to the app in the console, and "the permissions held by the user an app is acting for always constrain the app, regardless of the app's scopes." Some REST resources are marked "**Apps can't access this REST resource**" and cannot be used with 3LO at all.
- **Two scope families for Confluence:** classic scopes (e.g. `read:confluence-content.all`, `write:confluence-content`, `read:confluence-content.summary`, `read:confluence-space.summary`, `read:confluence-user`, `read:confluence-groups`, `search:confluence`, `manage:confluence-configuration`) and granular per-resource scopes (e.g. `read:page:confluence`, `read:space:confluence`), configured on separate tabs in the console; each API operation documents its required scopes.
- **Rotating-refresh failure modes:** failing to store the replacement refresh token, or an expired one, returns `403 Forbidden` with `invalid_grant` ("Unknown or invalid refresh token"); a refresh token also dies if "The user's Atlassian account password has been changed" — the user must then "initiate the entire authorization flow from the beginning again."
- **Compliance trap:** "Apps that collect API tokens or instruct customers to create individual 3LO apps don't comply with our Security requirements for cloud apps and Acceptable use policy." Also: "Clearly identify your app in the Marketplace (recommended)."
- **Code flow only:** known issue — "OAuth 2.0 (3LO) currently supports the code grant flow only. It does not support the implicit grant flow," which Atlassian acknowledges blocks standalone mobile and web/JavaScript apps (CORS whitelisting *is* supported for `api.atlassian.com`).
- **Admin revocation gap:** "An admin can't revoke a user's access to an app through the Connected Apps user screen… An admin can only choose to uninstall the app to restrict a user's access."
- **API-routing quirk:** calls must use `api.atlassian.com/ex/confluence/{cloudid}/…`; the `accessible-resources` `id` "is not unique across containers," so you may need to infer the container type from its scopes.

## 6. Sources (official, actually read)

- https://developer.atlassian.com/cloud/jira/platform/oauth-2-authorization-code-grants-3lo-for-apps/ — Atlassian's canonical "OAuth 2.0 (3LO) apps" guide (the shared platform doc the Confluence doc tree routes to; it documents Confluence endpoints, a Confluence accessible-resources example, distribution/sharing, and the refresh-token table)
- https://developer.atlassian.com/cloud/confluence/index/ — About Confluence Cloud (security options; 3LO not supported for Connect apps)
- https://developer.Atlassian.com/platform/marketplace/knowledge-base/listing-a-third-party-integration-on-the-atlassian-marketplace/ — integration definition, listing fields, review process
- https://developer.atlassian.com/platform/marketplace/creating-a-marketplace-listing/ — "5-10 business days" review statements, submission steps
- https://developer.Atlassian.com/platform/adopting-forge-from-connect/connect-to-oauth2-scope-equivalences/ — Confluence classic scope list

## 7. Unverified (docs read do not state)

- Numeric access-token lifetime and authorization-code lifetime (only `expires_in` as a field is documented).
- Any numeric user/install cap for a shared (unapproved) 3LO app.
- Any fee for third-party-integration approval or an informational Marketplace listing.
- Whether the account-level vs resource-level grant-type choice can be changed after app creation (docs describe choosing it at creation only).
- A review duration specific to third-party-integration approvals (the integration KB page gives none; the 5–10 business days figures come from the general Marketplace listing guide).

---

# KLAVIYO

**Headline:** Klaviyo has a real, self-serve OAuth program for third-party "public apps": authorization code flow with mandatory PKCE, account-scoped tokens, and **no review needed to get real customers** (you distribute your own installation URL). The formal App Marketplace review is optional but gated behind **5 active customer installs** — and over-scoping is an explicit rejection reason.

## 1. What the OAuth app is called / recommended flavor

- The object is an **app** (a public OAuth app) created on the **Manage apps** page (`klaviyo.com/manage-apps`) via **Create app**. Creation issues a **client ID** and **client secret**. "Public apps that can be installed from Klaviyo's App Marketplace require authentication via OAuth."
- **Recommended flavor for a third-party SaaS/AI agent:** one public OAuth app using the **authorization code flow with PKCE (S256)**, token exchange done server-to-server. "Klaviyo requires PKCE for safely storing credentials for both public and confidential client types."
- Tokens are **account-scoped, not user-scoped**: "Access tokens and refresh tokens grant access to Klaviyo accounts, not users." Agencies/platforms serving many merchants connect each Klaviyo account separately.
- Alternative (not recommended for third parties): **private API keys** — per-account credentials for your own/single-account integrations. Klaviyo pushes OAuth for anything distributed: the docs state "Your app must use OAuth in order to be considered for Klaviyo's App Marketplace," and apps still on private-key auth had to migrate: "If your app uses private key authentication, you'll need to update your app to OAuth by June 30th, 2025… Failure to update may cause service disruptions and potential de-listing of your app."

## 2. Setup steps in order

1. **Create the app** *(human only — "only owner, admin, and manager roles can create OAuth apps in Klaviyo")*: Manage apps → **Create app** → name it → "securely save your client secret and client ID. You won't be able to view your client secret again, although you can generate a new one later" → **Create**.
2. **Set scopes** *(human in UI; agent can prepare the list from API reference)*: space-separated list of every scope your authorization requests will need. "`accounts:read` is required by default and must remain in your scopes list," and "you must use the least permissive scope set possible."
3. **Set Redirect URLs** *(human in UI)*: the allowlist Klaviyo may redirect to after authorization; multiple URLs separated by space/tab/enter. **Save.**
4. **Build the installation URL** *(agent-automatable)*: `https://www.klaviyo.com/oauth/authorize?response_type=code&client_id={client_id}&redirect_uri={redirect_uri}&scope={scopes}&state={state}&code_challenge_method=S256&code_challenge={code_challenge}`. "You are required to generate a new `code_challenge` and `code_verifier` pair for your installation URL for every unique authorization request"; store the `code_verifier` server-side keyed by a customer identifier passed as `state`. The `redirect_uri` "must exactly match a URL that has been entered in the allowlist," and "your OAuth app must be able to support installation from both klaviyo.com and any subdomain of klaviyo.com."
5. **User authorizes** *(human end user)*: Klaviyo shows the requested permissions; on approval it redirects to your `redirect_uri` with `code` and `state`; on denial with `error=access_denied`.
6. **Exchange the code** *(agent, server-to-server)*: `POST https://a.klaviyo.com/oauth/token` with `Authorization: Basic <base64 client_id:client_secret>`, `Content-Type: application/x-www-form-urlencoded` ("isn't in some other format like JSON, which can result in difficult to debug errors"), body `grant_type=authorization_code`, `code`, `code_verifier`, `redirect_uri`. Store the returned `access_token`, `refresh_token`, `expires_in`, `scope`.
7. **Call the API** *(agent)*: `Authorization: Bearer {access_token}` against `https://a.klaviyo.com/api/{endpoint}/`; reuse the access token for its whole lifetime rather than minting per call.
8. **Refresh** *(agent)*: `POST` the same token endpoint with `grant_type=refresh_token` and the stored refresh token; persist the newly returned token set.
9. **Revoke on uninstall** *(agent)*: `POST https://a.klaviyo.com/oauth/revoke` (refresh tokens only) when a user uninstalls your app from your platform.
10. **Distribute** *(human)*: share the installation URL with customers — no listing required (see §3). After 5+ active installs, optionally submit for Marketplace review.

## 3. Approval / review

- **No review is required before real external users can connect.** "Your app does not have to be listed to share and use your installation URL with customers. Once you've set up OAuth for your app, share the installation URL you created to invite customers to install and start using your app."
- **Pre-approval consequence is a warning, not a cap:** "Until your app has been reviewed and approved by our team, users will receive an OAuth installation warning that your app has not been reviewed by Klaviyo. While customers and testers can still install and use your app, having your app approved by Klaviyo will remove this warning and allow for easy discoverability within the App Marketplace." No install/user cap for unlisted apps is stated in the docs.
- **Marketplace listing is a formal review with an eligibility gate:** "**Your app must have at least 5 active customer installs for submission. These must be real customer accounts with live API activity—test accounts do not qualify.**" The listing requirements restate: "Your app must have at least 5 installs with demonstrated Production level API activity. (Note: Developer accounts or any accounts associated with your company do not count towards this requirement.)"
- **Submission process** (Manage apps → 3 dots → Edit): complete **Authentication** (scopes, redirect URLs, and testing instructions — a URL to a Google Doc with "Commenter" access that "must be an adapted copy of our Klaviyo App & Testing Requirements template. Links to public-facing documentation will not be accepted"; never share credentials — add `integration.directory@klaviyo.com` to your test account instead), **Company details**, **Listing details** (developer name ≤ 30 chars, short description ≤ 75 chars, full description ≤ 300 chars, 2–6 features, at least one user-guide article, SVG logo 48×48 "without any scripting," media gallery of 4–8 images or a cover image, installation/settings/pricing/support/privacy URLs), and the **Registration & security** forms — "If you submit your application without submitting these forms, your app will be rejected" (integration registration form once per company; security form per app). Then **Review submission** → **Submit**.
- **Review checks content:** listing requirements cover customer value, ease of install ("We expect your app to be live and functional within minutes"), OAuth with least-permissive scopes, stable APIs, full install/uninstall/error handling, listing content rules (ToS/Privacy links, no implied Klaviyo endorsement), testing information (workflow, endpoint/scope list, architectural diagram, recorded demo), security (TLS 1.2+, AES-256 at rest, MFA, logs retained "a minimum of 365 days," credential-store and rotation requirements) and legal/AUP restrictions.
- **Discretion and re-review:** "not all submitted apps will be approved. Klaviyo reserves the right to not list your app for any reason." Post-submission, "our team will review your submission and follow up via email if the app has been approved, rejected, or requires additional changes before approval." App states: *Draft*, *Submitted for review*, *Published*, *Unpublished*. Changes to a published app "will not be published until your app has been re-reviewed and approved."
- **Cost:** no listing/submission fee is stated anywhere in the docs read (see §7).

## 4. Time / effort (provider-stated figures only)

- "**Access tokens are valid for 1 hour** (this is subject to change). Make sure to rely on the `expires_in` returned by `/token`."
- "**The authorization code expires in 5 minutes.**"
- Refresh token: "will remain valid until the app is uninstalled by the user. Caveat: **If a refresh token is not used after 90 days, it is revoked.**" Also: "**Refresh token can only be called 10 times per minute maximum.**"
- Rate limits: "OAuth apps receive their own rate limit quota per installed app instance (i.e., per account per app)," unlike private keys which share a per-account quota; the token endpoint's rate limit is described only as "much lower" than API endpoint limits (no number stated).
- Token sizes for storage planning: "Access tokens may be up to 4,096 characters long and refresh tokens may be 512 characters long."
- No setup duration and **no review duration** are stated in the docs read (see §7).

## 5. Blockers and nuances

- **Over-scoping is a stated rejection reason:** "You must use the least permissive scope set possible in compliance with our app listing requirements. If your app requests more scope permissions than necessary, it will be rejected." Scopes are fine-grained `resource:action` pairs (e.g. `lists:write`, `campaigns:write`, `metrics:read`), each endpoint's API reference lists its required scope, and `accounts:read` can never be removed.
- **Host trap:** "On March 31, 2025, Klaviyo started blocking all OAuth *token* traffic (`oauth/token` and `oauth/revoke`) routed through `www.klaviyo.com`. Ensure that your integration routes token traffic to `https://a.klaviyo.com/oauth/token` or `https://a.klaviyo.com/oauth/revoke` to avoid breakages." (Authorize still starts on `www.klaviyo.com`.)
- **Refresh semantics are unusual:** refreshing issues a new access token but "the previous access token is not revoked. The access token remains valid until it expires or the user uninstalls the app"; per the token response and Klaviyo's integration guidance, refresh tokens **rotate on every use**, so persist the latest refresh token after every refresh or the next refresh fails.
- **`invalid_grant` means treat as uninstalled:** a 400 `{"error":"invalid_grant"}` on refresh means the token is invalid because "The customer uninstalled your app in Klaviyo," "The token expired after 90 days of no-use," "The token has been revoked by Klaviyo's internal systems for security reasons," or it is incorrect. "You can treat all `invalid_grant` responses, regardless of the `error_description`, like the application has been uninstalled by the user."
- **Access tokens cannot be revoked:** "The revoke endpoint only accepts refresh tokens. Access tokens are JWTs that are intended to be short-lived, so they can't be revoked. Any access token issued before you revoke the refresh token stays valid until it expires."
- **Secret custody:** the client secret is shown once at creation; losing it means generating a new one via the UI. Token exchange must be server-to-server: "The client secret is private and should never be in the browser because you would expose all of your customer's data."
- **Role gate on creation:** only owner/admin/manager roles can create OAuth apps.
- **Listing traps:** the 5-install gate must be met with *real customer* accounts first (chicken-and-egg: you must distribute the unlisted, warning-carrying app to earn eligibility); testing instructions must follow Klaviyo's template in a shared Google Doc, and scope or listing changes after publication require resubmission and re-review before they go live.

## 6. Sources (official, actually read)

- https://developers.klaviyo.com/en/docs/oauth_resources — "Create a public OAuth app"
- https://developers.klaviyo.com/en/docs/set_up_oauth — "Make API calls using OAuth" (authorize URL, PKCE, token exchange, refresh, revoke, rate-limit quota, endpoint table)
- https://developers.klaviyo.com/en/docs/submit_your_app_for_review — submission steps, 5-install gate, warning, forms, post-submission flow
- https://developers.klaviyo.com/en/docs/klaviyo_app_listing_requirements — listing requirements (install gate, security, testing, content rules)

## 7. Unverified (docs read do not state)

- Any App Marketplace review timeline/duration (the docs only say the team "will follow up via email").
- Any fee for creating an OAuth app, submitting for review, or being listed.
- Any cap on installs/users for an unlisted (unreviewed) app distributed via its installation URL.
- The numeric rate limit of the `oauth/token` endpoint (stated only as "much lower" than API endpoint limits).
- Whether a generated replacement client secret immediately invalidates the previous secret (docs say you "can generate a new one later" without specifying overlap behavior).

---

**Cross-cutting note for the library:** both are real OAuth programs with the same shape as Jira's (Atlassian shared platform): no review blocks real users, distribution is via your own authorization/installation link, and "approval" only removes an "unreviewed app" warning and unlocks marketplace discoverability. The gates differ: Atlassian has no install threshold for approval submission, while Klaviyo requires 5 active production installs before you may even submit.