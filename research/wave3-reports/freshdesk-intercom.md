Research complete. Both providers have real OAuth programs for third-party apps; neither is a dead end. Key structural difference: Freshworks' OAuth is org-domain-based and its review only gates Marketplace listing, while Intercom's review gates the public app itself before any external customer can install it.

---

# FRESHDESK (Freshworks)

**Headline:** Freshworks offers a genuine OAuth 2.0 authorization-code program. The OAuth object is a set of **OAuth credentials** (one `client_id` / `client_secret` pair) created in the Freshworks Developer Portal and bound 1:1 to one app. Note on sources: Freshworks' App SDK docs are templated across products (Freshdesk = `support_*` modules; the pages read were the `service_*` variants carrying the identical platform text, with Freshdesk examples such as `subdomain.freshdesk.com`). Product-specific Freshdesk scope strings were not enumerated in the pages read — see Unverified.

## 1. What the OAuth app is called, and the recommended flavor

- **The object:** "OAuth credentials" for an "OAuth app." You create them in the Developer Portal under **Settings → OAuth Credentials → Create OAuth Credentials**, specifying a name (not exceeding 3 words, shown in the authorization window), a description (also shown in the authorization window), redirect URLs, and scopes selected per product/resource.
- **Recommended flavor for a third-party SaaS / AI agent whose users connect their own Freshdesk accounts: an External OAuth app using those credentials, running the standard authorization-code flow against each customer's Freshworks organization domain.** The docs define External apps as "non-Freshworks apps, built using third-party SDKs (or other provisions)" and describe exactly this journey: your app routes the user to the Freshworks authorization server at `https://<org_url>/oauth/v2/authorize` with `client_id`, `redirect_uri`, `state`, `scope`, `response_type=code`; the user picks an account (if they have several), reviews the consent form, clicks **Agree**, and your app exchanges the returned code at the org token endpoint (`https://<org domain>/org/oauth/v2/token` in the documented `oauth_config.json` format, with an `Authorization: Basic base64(client_id:client_secret)` header).
- **Alternatives:**
  - **FDK Freshworks app (Marketplace app):** built with the Freshworks CLI/FDK and installed from the Freshworks Marketplace. Here "OAuth handshake is completely taken care by the Freshworks Developer Platform" — the platform constructs the authorization-code and access-token requests from your `config/oauth_config.json` and stores the tokens. Choose `token_type: "account"` (handshake only when an admin installs; the app then serves all agents) or `token_type: "agent"` (handshake when an agent installs; **cannot** be `agent` for serverless/background apps, and only one OAuth configuration per app may be agent-type).
  - **Custom app:** an FDK app private to a single customer account (see §3 — no review).
  - **Non-OAuth alternative (documented, not recommended for multi-tenant SaaS):** Freshdesk API key passed as a secure installation parameter (iparams) and used as Basic auth, as in Freshworks' own "Build your first Freshworks app" tutorial. That is per-account credential collection, not delegated OAuth.

## 2. Setup steps in order

1. **Create/sign in to a Freshworks Developer account and open your organization.** The docs note: "To submit a Freshworks or External app, you only need a Freshworks Developer account. An active product subscription is not required." (Custom apps are the exception — they require an active product subscription.) *[Human-only]*
2. **Create the OAuth credentials:** Developer login → enter organization/account URL → Settings (gear) → **OAuth Credentials → Create OAuth Credentials** → enter name and description, add redirect URL(s) (**maximum of five**; docs direct local testing to `http://localhost:10001/auth/callback` and production to `https://oauth.freshdev.io/auth/callback` in the FDK context), and select the required scopes in the **Add scopes** section → **Create Credentials**. *[Human-only — portal action plus custody of the secret]*
3. **Record the generated Client ID and Client Secret** from the credentials page and store them securely; they are what your app uses for the handshake. *[Human-only]*
4. **Implement the connect flow:** for each customer, build the authorize URL on *that customer's* Freshworks org domain (never hardcode a domain — the docs warn a hardcoded org domain breaks if the domain changes, e.g. on account transfer, and recommend capturing it dynamically via `oauth_iparams`). Parameters: `response_type=code`, `client_id`, `state` (mandatory, "randomly generated alphanumeric string"), `scope` (space-separated; "You may include all or a subset of the scopes as selected in the OAuth credentials. Ensure that you **do not** include scopes that are not selected in your OAuth credentials"), `redirect_uri` (must be one registered on the credentials). *[Agent-automatable — code; the per-customer domain handling is the part to get right]*
5. **User authorization:** the customer admin/agent signs in, selects the account whose data to expose, reviews the consent form (which separates scopes already approved from scopes awaiting approval), and clicks **Agree**; cancelling leaves "the OAuth app remains unauthorized and non-functional." *[Human-only — the end user]*
6. **Exchange the code** at the org token endpoint and **store the access token and refresh token per customer org**; call the Freshdesk REST APIs with the access token as a bearer token. (In the FDK flavor this and token storage are done by the platform/Marketplace; in the External flavor your app logic does it.) *[Agent-automatable]*
7. **Handle refresh and reauthorization:** implement refresh before expiry, and guide users through reauthorization when refresh fails (account-level: App Gallery → app → **Settings → Reauthorize**; agent-level FDK apps: display a **REAUTHORIZE** button via `client.interface.trigger('reAuthorize')`). *[Agent-automatable, except the reauthorization click itself is human]*
8. **(FDK flavor only) Test, validate, pack, submit:** test locally with `fdk run` (configurations at `http://localhost:10001/custom_configs`, product UI with `?dev=true`, click **Authorize** in the app), then "Validate, pack, and submit your OAuth app." *[Either — CLI work is automatable; portal submission is human]*
9. **(Optional, for Marketplace distribution) Submit the listing:** Developer Portal → **Public App** tab → **+ Create New App** → for an External app, enter the URL to your app page, name, and product; complete App Information, Media and Installation, Support Details, and Privacy and Security tabs; click **Save and Publish** to submit for approval. *[Human-only]*

## 3. Approval / review

- **OAuth credentials themselves:** no review is described for creating credentials — it is a self-serve Settings action.
- **Custom apps:** explicitly **not reviewed**: "A custom app is not subject to review; it is available for installation as soon as it is submitted and published," and the Apps overview table repeats: Custom apps — "Review: Not required," distribution by "Manual upload of the packaged FDK artifact," visibility "Private to one product account."
- **Freshworks Marketplace apps and External apps:** **review is required before publication/listing.** Clicking **Save and Publish** submits "the app for the approval process"; the app shows as **Draft Version (Under Review)**, and "After the app is reviewed and approved, it is published to the Freshworks Marketplace" (for External apps: "it is listed in the Freshworks Marketplace," while installation happens from the third-party's URL). Review is tracked across five categories — **Code** ("Code quality, security, and platform compliance"), **QA** ("Functional testing"), **Content** ("Listing content, links, category, screenshots, video, and descriptions"), **Security** ("Security and privacy review"), **Business** ("Business and marketplace policy review") — with overall statuses Draft/In Development → Under Review → Changes requested / Ready to publish / Rejected → Published.
- **Pre-approval caps:** no numeric user/install cap before approval is stated in the docs read. The structural cap is that an unapproved Freshworks/External app is not Published/listed, so Marketplace users cannot install it; "selective access" via a hidden listing plus an account-specific shareable URL is documented only **after** publication.
- **Cost:** no listing fee, review fee, or revenue-share figure is stated in the docs read (see Unverified). The docs do note global apps "can be published as Freshworks paid apps on the Freshworks Marketplace for all products except Freshsales Classic."

## 4. Time / effort (provider-stated figures only)

- **Access token lifetime:** "The access token has a lifetime of 30 minutes. At a regular interval, the developer platform places a refresh-access-token call to the authorization server and regenerates the access token" (the automatic refresh sentence describes the platform-managed FDK flow).
- **Refresh token lifetime:** "The refresh token has a lifetime of 365 days. After the refresh token expires, the access token is not updated and all REST API calls fail with a 401 - Unauthorized error." The app-user-journey doc repeats: "The refresh token has a lifetime of 365 days. After the refresh token expires, a new access token cannot be retrieved from the authorization server."
- **Review duration:** no duration is stated in the official docs read (see Unverified).
- No setup-duration estimates are stated.

## 5. Blockers and nuances

- **Scopes are additive-only — a one-way door:** "While you can add new scopes, removing scopes that are already selected is not supported" when updating OAuth credentials. Choose the initial scope set carefully.
- **Credentials are bound 1:1 to an app:** "The mapping between OAuth apps and OAuth credentials is 1-1. A configured OAuth credential can only be used with one app," and uploading another Marketplace/Custom app with the same credentials produces an error in the App Management Portal.
- **Per-customer org domains:** authorize/token URLs embed the Freshworks organization domain, which differs per customer; hardcoding yours breaks other customers and breaks on domain change/transfer. Capture the domain at install (oauth_iparams).
- **User/account deletion revokes everything:** "When a user is deleted from an organization or removed from an account, the authorization server revokes any consents given to OAuth apps in that account and delete the refresh tokens issued to those apps. So, any subsequent API requests will fail."
- **365-day reauthorization cliff:** after refresh-token expiry there is no silent renewal — an admin (account-level) or agent (agent-level) must reauthorize; your app "should guide the app users to do this" with clear error messaging.
- **Multi-OAuth and multi-account ceilings (FDK platform):** platform v3.0 allows a maximum of **three** OAuth-secured resources per app; v3.1+ allows **five**, with only one configuration facilitating agent installation; the multi-account feature "allows your app to access up to **20** accounts per OAuth-secured resource" (account-level only). Agent-level OAuth "cannot be used from the **server.js** file."
- **Redirect URL ceiling:** up to **five** redirect URLs per credential set — plan local + production URLs within that budget.
- **Marketplace code-coverage gate (FDK apps):** "For Marketplace submission, fdk pack requires a minimum of **50%** coverage for statements, branches, functions, and lines, evaluated individually. If unit tests are configured in your app, fdk pack further requires at least **80%** coverage for each of these metrics based on results from fdk unit-test."
- **Module lock-in after publishing (FDK global apps):** "After you build a global app that works across modules and publish it to our marketplace, in the subsequent versions of the app, you cannot remove modules (that is, downgrade the number of modules)."
- **Consent naming matters:** the credential name (≤3 words) and description are exactly what users see in the authorization window; the docs recommend matching the app-submission name for a consistent experience.

## 6. Sources (official, actually read)

- https://developers.freshworks.com/docs/app-sdk/v3.0/service_user/advanced-interfaces/request-method/oauth-access-freshworks-resources/ (creating/updating OAuth credentials, `oauth_config.json`, token lifetimes, testing, publishing)
- https://developers.freshworks.com/docs/app-sdk/v3.0/service_ticket/oauth-implementation-in-apps/oauth-app-installation/ (app user journey: Freshworks, External, and reauthorization flows; 365-day refresh token)
- https://developers.freshworks.com/docs/app-sdk/v3.0/lead/app-submission-process/ (app types, Custom-app no-review rule, review statuses and categories)
- https://developers.freshworks.com/docs/app-sdk/v3.0/service_asset/app-submission-process/external-apps/ (External app submission steps, listing/media/support/privacy requirements)
- https://developers.freshworks.com/docs/app-sdk/v3.0/service_ticket/app-development-process/ (FDK build/test/pack process, code-coverage requirements)
- Search-result excerpts read from developers.freshworks.com for: "OAuth implementation in apps" (Freshworks/Custom/External app definitions), "What is OAuth?" (platform-managed handshake), "Apps overview" (Marketplace vs Custom distribution/review table), and "Build your first Freshworks app" (API-key iparam alternative)

## 7. Unverified (not stated in the official docs read)

- **Review timeline** for Freshworks/External Marketplace apps — the docs read describe statuses and categories but no duration.
- **Any listing/review cost or revenue share** for Marketplace or paid apps.
- **Exact Freshdesk-specific scope strings** (e.g. a `freshdesk.tickets.*` list): the credentials UI lists per-product resource permissions, and the docs' worked examples use Freshservice/Freshchat scope names; a Freshdesk scope catalog was not in the pages read.
- **Whether the External (non-FDK) token endpoint accepts the same Basic-header client authentication in all cases** — documented explicitly in the `oauth_config.json` format; the External-app user-journey page states only that "your app logic… place[s] the access-token request."
- **Standalone REST API rate limits** for OAuth tokens (not covered in the pages read).

---

# INTERCOM

**Headline:** Intercom has a standard, well-documented OAuth 2.0 authorization-code program for **public apps** in the Developer Hub. The defining trait: **every public app — listed or unlisted — must pass Intercom's review before any external customer can install it**; until approval the app is private to you and your team.

## 1. What the OAuth app is called, and the recommended flavor

- **The object:** an **app** created in the Intercom **Developer Hub** (in a development workspace), with OAuth enabled via the **Use OAuth** checkbox on its Authentication page. Client ID and Client Secret are found on the app's **Basic Information** page.
- **Recommended flavor for a third-party SaaS / AI agent: a public app with OAuth.** Intercom is explicit: "If you are building a public integration that will have access to **other peoples'** Intercom data you are required to set up OAuth," and the Authentication guide says you should use OAuth if "You are requesting access to other people's Intercom accounts/data," "You currently ask people for their API Keys," or "You want to make it easier for your customers to share their customers' data with you." Asking users for their Access Tokens instead "is against our terms of service and may result in your API access being revoked."
- **Alternative (own workspace only):** a **private app** using the app's **Access Token** (available "as soon as you create an app on your workspace," found under **Configure > Authentication**). This is only for accessing "your own" workspace data — scripts, automation, a private app — and private apps "do not require a review." It is not a customer-connection mechanism.
- **Distribution flavors (both are "public apps," both require review):** **Listed** (discoverable/installable in the Intercom App Store) vs **Unlisted** ("available for anyone to install but is **not** listed in the App Store: It must be installed via your own product or website").

## 2. Setup steps in order

1. **Create a development workspace and create the app in the Developer Hub** (New app → name → workspace → Create app). The OAuth guide notes "Each development workspace automatically gives you the option to use OAuth." *[Human-only]*
2. **Enable OAuth:** on the app's Authentication page, tick **Use OAuth**. *[Human-only]*
3. **Add Redirect URL(s):** "Redirect URLs must use HTTPS" (the redirect communicates over TLS/SSL). Multiple URLs are allowed via **Add redirect URL**; the first is the default, and "After approval, you can choose which URL to use when you're initiating the OAuth flow via the `redirect_uri` parameter." *[Human-only — configuration; the URLs themselves must exist in your product]*
4. **Select permissions (scopes)** by checkbox on the Authorization/Authentication page — "You should only specify the scopes you definitely need for your use case," because they "will go up for review when you submit your app for review." Standard ("People & conversation data") scopes include Read and list users and companies, Read and write users, Write users and companies, Read tags / Write tags, Read conversations, Write conversations ("Reply to, mark as read and close conversations"), Read events / Write events, Read counts, Write data attributes, Export message data, Export content data, Read tickets, Write tickets ("Create tickets"), Read and write custom object instances, Read status of jobs; Extended ("Workspace data") scopes include Read admins, Update admins, Read admin activity logs, Read and List / Read and Write articles and news items, Read and write AI content, Read / Read and write data connectors, Create phone call redirects, and others. Webhook topics are tied to matching scopes (e.g. a user-created webhook needs "Read and write users"). *[Human decision — product/engineering choice]*
5. **Retrieve `client_id` and `client_secret`** from the app's Basic Information page in the Developer Hub and store them server-side. *[Human-only]*
6. **Send the user to the regional authorize URL:** `https://app.intercom.com/oauth?client_id=___&state=___` (US), `https://app.eu.intercom.com/oauth?client_id=___&state=___` (EU), `https://app.au.intercom.com/oauth?client_id=___&state=___` (Australia). `state` is "A random session specific string which we will return in the callback… to mitigate CSRF attacks." The user sees the requested permissions and approves. *[Agent-automatable — code; approval click is the end user's]*
7. **Handle the callback:** Intercom redirects to your Redirect URL with `code` and `state` parameters. *[Agent-automatable]*
8. **Exchange the code:** `POST https://api.intercom.io/auth/eagle/token` with body parameters `code`, `client_id`, `client_secret`. The response contains `token_type` ("Always 'Bearer'"), `token` (the Access Token), and `access_token` ("A duplicate of the Access Token above"). *[Agent-automatable]*
9. **Call the API** with the token as an `Authorization: Bearer <ACCESS_TOKEN>` header (e.g. `GET https://api.intercom.io/me`). *[Agent-automatable]*
10. **Submit for review** (required before external installs — see §3): in the Developer Hub's **Test and publish** section, complete the App Partner Program information; if listing, the App Store listing and Start guide; then on the **Submit for a review** page add the app description (including test-account credentials or setup instructions if your service needs an account) and the mandatory end-to-end video, and click **Submit for Review**. *[Human-only]*

## 3. Approval / review

- **Review is mandatory for all public apps:** "In order to make your app public and available for customers to install, you will need to submit it for a review from the Intercom team." "If you are building an app that can be installed and used by any customer on Intercom, and will have access to their workspace data, you are building a **public app**. This means that the Intercom team must review and approve your app before it is available to install." And: "**Both unlisted and listed apps must be submitted for review.**"
- **Private apps are exempt:** "Private apps that only access your own paid workspace data **do not require a review.**"
- **Pre-approval state/cap:** no numeric user or install cap is stated; the cap is categorical — "During this process, your app will be **private** and can only be used by you and your team." On approval "it will become a **public unlisted** app" (you then opt in to App Store listing separately via **List in App store**).
- **What reviewers check:** the required end-to-end video must show "Installation of the app with OAuth successfully working," "The end-to-end experience and functionality of your app," and "Uninstallation of the app with OAuth," and "Please show the full `client_id` and state parameter in the OAuth URL." Scope discipline is enforced: "If you have requested scopes outside of the current functionality, you may be asked to adjust your scopes and resubmit the app," and an incomplete video means "you will have to resubmit your video." Feedback arrives via an in-app Messenger conversation; rejected apps are fixed and resubmitted via **Submit for Review**.
- **Listed-app requirements (additional):** App Listing + App Partner Program pages must be complete; the listing guide requires, among other fields: icon "512px by 512px and PNG format"; app name "40 characters or less," unique in the store, and it "Must not contain Intercom"; short description "70 characters or less"; Terms of service URL and Privacy policy URL; a pricing-model selection (Free / Trial for free / Some features free / Paid app — a paid app must give a pricing-page URL or a sales contact route); Documentation URL; Support email; and "Your listing must have **2-5 images**" of at least 1600px by 1000px PNG, under 21MB. For listed apps the Installation status section is also mandatory — "Providing broken or unclear links will result in an application rejection."
- **Updates re-enter review:** "if you need to make any changes in the future the app will need to be re-approved by the Intercom team" (for an already-published app, the current version stays public while updates are reviewed). Intercom recommends keeping a separate staging app (name the production app's twin with `[Staging]`/`[Dev]` appended).
- **Cost:** no review or listing fee is stated anywhere in the docs read (see Unverified). Publishing a listed app does auto-enroll you in the App Partner Program: "Once you've published your app to the store and you will automatically be enrolled in our App Partner Program."

## 4. Time / effort (provider-stated figures only)

- **Review wait:** "You will see a yellow banner at the top of the screen stating it can take up to seven business days for review." For resubmissions: "Your application can take up to seven business days to review."
- **Token lifetimes:** the OAuth and Authentication docs read state **no** access-token lifetime, expiry value, or refresh-token behavior (see Unverified — do not assume a refresh flow exists based on these docs).
- **Setup durations:** none stated.
- Listing-video guidance (not a review requirement): the listing guide recommends a marketing video of "30-60 seconds."

## 5. Blockers and nuances

- **Review gates connectivity, not just listing:** unlike marketplaces where review only gates the storefront, an unapproved Intercom public app cannot be installed by customers at all — plan launch timing around the up-to-seven-business-day review, plus possible resubmission cycles.
- **Regional authorization hosts are a real trap:** workspaces live in US/EU/AU regions, each with its own `/oauth` host. Using the US host for an EU/AU customer adds a region-selection step on sign-in — and "if the customer signs in with Google, the wrong regional host fails rather than prompting, so matching the region is required in that case."
- **The trailing `=` code trap:** "The code value in the redirect URL may contain a `=` at the end which needs to be included." Dropping it yields a `401` with error message `Unauthorized Code` at token exchange — a classic silent-integration bug.
- **No refresh token appears in the documented exchange:** the token response documented is only `token_type`, `token`, and `access_token`. The docs read describe no refresh grant, so token renewal/revocation handling is unverified territory (see §7) — design your connect UX so re-authorization is painless.
- **Canvas Kit force-selects scopes:** if your app uses Canvas Kit, "Read and list users and companies," "Read conversations," "Read admins," and "Gather App data" "will be automatically selected and cannot be deselected" — which reviewers will then see in your scope set.
- **Scope changes are review events:** because scopes "go up for review" and post-approval changes need re-approval, adding a permission later is a release-process decision, not a config tweak.
- **Access Token hygiene is enforced by policy:** never ask customers for their Access Token — Intercom states this "is against our terms of service and may result in your API access being revoked," and tells users to report providers who ask.
- **Listing name/format pitfalls cause rejections:** names containing "Intercom" "will not be approved"; non-marketing-format images are called out as "one common pitfall"; broken installation links "will result in an application rejection, and you will have to resubmit for review."
- **Review needs a testable product:** if your app "requires a subscription or account with your service, you'll need to provide us with test account credentials or set up instructions within the description" — have a reviewer account ready before submitting.

## 6. Sources (official, actually read)

- https://developers.intercom.com/docs/build-an-integration/learn-more/authentication/setting-up-oauth (OAuth setup, redirect URLs, full scope tables, regional hosts, code/token exchange, `=` code warning)
- https://developers.intercom.com/docs/publish-to-the-app-store/review-publish-your-app (which apps require review, listed vs unlisted, submission requirements, seven-business-day review, approval/update behavior)
- https://developers.intercom.com/docs/publish-to-the-app-store/listing-your-app (listing requirements, App Partner Program auto-enrollment)
- https://developers.intercom.com/page-data/docs/build-an-integration/learn-more/authentication/data.json (official page data for the Authentication guide: Access Token vs OAuth, token usage)

## 7. Unverified (not stated in the official docs read)

- **OAuth access-token lifetime/expiry and whether refresh tokens are issued** — the Setting up OAuth and Authentication pages document the token response fields only, with no `expires_in` and no refresh flow.
- **Authorization-code lifetime** — not stated.
- **Any review, listing, or App Partner Program fee** — no cost figure appears in the docs read.
- **Any numeric pre-approval user/install cap** — the docs state only that the app is private to "you and your team" during review.
- **Exact criteria/timeline for App Partner Program enrollment beyond auto-enrollment on publishing a listed app** — not detailed in the pages read.
- **Whether unlisted apps can be installed by customers before approval via a direct OAuth link** — the review doc says public apps must be approved "before it is available to install" and that during review the app "can only be used by you and your team," which reads as no; no separate pre-approval external-install allowance is documented.