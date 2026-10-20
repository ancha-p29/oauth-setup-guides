# Facebook + Instagram: BLOCKED via research-agent route (Oct 20, 2026)

developers.facebook.com returned HTTP 403 to the research agent's reader,
and search indexes returned zero developers.facebook.com pages (only
third-party blogs, excluded per rules). Every substantive claim about
Facebook Login, App Review/Advanced Access, Business Verification, Data
Use Checkup, Instagram API with Instagram Login, and all token lifetimes
is UNVERIFIED. Do NOT write these guides from this pass.

Full agent report: ~/agents/agent-0920324b-d874-4a32-94d7-39841ba681a0/sessions/0920324b-d874-4a32-94d7-39841ba681a0.jsonl

Fallback in flight: live-browser task (browser-task:61a18eec) attempting
to read the doc trees directly. Count without the whole Meta family
(FB/IG/WhatsApp): 105, still above 100.

RESOLVED Oct 20, 2026: a live-browser session read developers.facebook.com directly (no 403, no login wall, signed-out), covering Facebook Login, App Review, and the Instagram Platform pages. Facebook and Instagram guides written from that report.
