# Verification log (2026-10-19)

Independent spot-checks by the site author against official provider docs,
beyond the research agents' own source tracing. Every guide's `evidence`
block already lists its sources and its unverified items; this log records
what was additionally checked, corrected, or left uncertain.

## Checked and confirmed (no changes needed)

1. **Spotify dev-mode 5-user cap, Premium requirement, allowlist flow** --
   confirmed verbatim at
   https://developer.spotify.com/documentation/web-api/concepts/quota-modes
   ("Up to 5 authenticated Spotify users can use an app that is in development
   mode"; "The app owner must have a Spotify Premium account").
2. **Spotify org-only quota extensions, 250k MAU, 6-week review** -- confirmed
   on the same page ("as of May 15th 2025, Spotify only accepts applications
   from organizations"; "at least 250k MAUs"; "This review process can take up
   to six weeks").
3. **Slack 10-active-workspace submission gate** -- confirmed at
   https://docs.slack.dev/slack-marketplace/slack-marketplace-review-guide/
   ("Your app has been installed on 10 or more active workspaces... apps that
   do not meet this requirement will be blocked from submitting").
4. **Slack review timelines** -- confirmed on the same page: preliminary "up to
   10 business days", functional "up to 10 weeks for new submissions and 6
   weeks for published apps resubmitting changes"; preliminary queue place
   resets on resubmission; published app config is locked.
5. **Google 100-new-user cap** -- confirmed at
   https://support.google.com/cloud/answer/7454865 ("100 new users in total,
   after the app presents the unverified app screen"; appeal = request
   verification).

## Corrected during research

- An early hypothesis that Google brand verification takes "3-5 business days"
  was wrong per the docs: brand verification is minutes (automated) to 2-3
  business days (manual); 3-5 business days applies to *sensitive-scope*
  verification. The Google guide reflects the corrected figures.
- The research brief's implied framing of Notion's review as a single step was
  corrected: Notion has TWO reviews (a mandatory but opaque "submit for
  review" gating the Authorization URL, plus an optional Marketplace listing
  review). The Notion guide keeps both and labels both timelines unknown.

## Could not verify (carried as `unverified` in the guides)

- Salesforce: whether new orgs must contact Support before creating connected
  apps (one developer guide says so; current Help docs show a self-serve
  toggle). No official end-to-end AppExchange review timeline is published.
- Spotify: exact dev-mode request quotas ("subject to change"); whether every
  agent-relevant endpoint remains available to dev-mode apps after the
  2024-2026 API restrictions (needs a live test); any exception path to the
  org-only extension policy.
- Slack: whether a verified email is required to create an app; the exact four
  Manage Distribution checklist items; whether any listing fee exists (docs
  silent).
- Notion: what the mandatory "submit for review" step checks, its timeline,
  and resubmission handling; Marketplace listing review timeline (docs defer
  to an unretrievable FAQ); exact required distribution fields and
  redirect-URI format rules (only third-party-quoted).
- GitHub: whether a verified email is required before app registration; any
  Marketplace onboarding-review SLA.
- Google: per-scope sensitivity markers for individual Calendar scopes
  (console's Data Access page is authoritative); CASA assessment cost/duration.

## Method note

All research was read-only: official docs were opened and read; no accounts
were created, no apps registered, nothing submitted. "Tested" in each guide
means exactly this, stated plainly, so a developer never mistakes
documentation review for a live walkthrough.
