# WhatsApp Business: DEFERRED (docs inaccessible, second confirmation)

Wave 4 retry (Oct 20, 2026) reproduced the wave 1 failure exactly:
developers.facebook.com returns HTTP 403 to automated readers, domain-wide
(6 URLs attempted: /docs/whatsapp, /docs/whatsapp/cloud-api,
/docs/whatsapp/cloud-api/get-started, /docs/whatsapp/cloud-api/overview,
/docs/whatsapp/embedded-signup, /docs/development/release/access-tokens).
Third-party sources (Plivo, Infobip, Twilio, etc.) were excluded per the
official-docs-only rule. NOTHING about the WhatsApp Business Platform program
(Embedded Signup, tech provider model, App Review for
whatsapp_business_management/messaging, token types/lifetimes, caps) is
verified. Do NOT write this guide from unverified material.

Options: (1) live-browser session with a Meta login to read the pages;
(2) leave WhatsApp out. Count math without it: 61 (waves 1-3) + 16 (wave 4
rest) + 16 (wave 5) + 15 (wave 6) = 108, still above 100.

RESOLVED Oct 20, 2026: a live-browser session read developers.facebook.com directly (no 403, no login wall, signed-out). WhatsApp guide written from that report (Embedded Signup, advanced-access gate, 10/200 onboarding caps). Facebook and Instagram guides written from the same browser report; see facebook-instagram-BLOCKED.md resolution note.
