# Agent generation brief (the "live generation" pipeline)

When a developer submits an app name or website that is not in the library, an
agent investigates it and produces a guide in this schema. This file is the
exact brief given to the research agent.

---

Research the OAuth app setup and approval process for **{APP}** ({URL_OR_NULL}),
from its OFFICIAL developer documentation only (search the web to find docs,
open and read them). Return a structured report covering:

1. **What "the OAuth app" is called** on this platform and which flavor a
   third-party AI-agent developer should pick, with why.
2. **Setup steps in order**: exact console/portal URLs; what is needed (account
   type, verified email, domain, app homepage URL, callback/redirect URIs,
   logo, privacy policy, etc.); which steps an automated agent could do vs
   which need the human developer.
3. **Approval/review process**: is there one? What triggers it? What gets
   reviewed, typical wait times, costs, resubmission reality.
4. **Time/effort estimate**: realistic range from "I have an account" to "my
   users can connect", split into active work vs waiting.
5. **Blockers and nuances**: scope sensitivity, user caps, verification tiers,
   branding/domain verification, common rejection reasons, surprises.
6. **Sources**: exact official doc URLs verifying each claim; flag anything NOT
   verifiable or ambiguous.

Rules: quote provider-stated wait times only; otherwise say "not published".
Do not invent steps. Do not create accounts or submit anything. Attempt as
much of the setup flow as is practical without credentials (e.g. walk public
registration pages to confirm required fields) and report what you tested.

---

The report is then converted to the guide schema (see
`schema/guide-schema.md`), marked `status: "draft"`, and added to the library.
A human reviews drafts before they are marked `published`.
