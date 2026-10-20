# Guide schema (v1)

Every guide is one JSON object. The site renders directly from this schema, so a
research agent can produce a new guide and it appears in the library with no
code changes.

```jsonc
{
  "id": "github",                    // kebab-case slug, unique
  "name": "GitHub",
  "tagline": "One short line on what the integration unlocks.",
  "category": "Developer tools",     // free text, used for filtering
  "providerUrl": "https://github.com",
  "flavor": {                        // which OAuth app type to pick, and why
    "recommended": "GitHub App",
    "alternatives": ["OAuth App (legacy)"],
    "why": "GitHub Apps give fine-grained permissions..."
  },
  "difficulty": 1,                   // 1-5 integer
  "difficultyLabel": "Trivial",      // Trivial | Easy | Moderate | Hard | Arduous
  "time": {
    "active": "30-60 min",           // hands-on work
    "waiting": "None",               // provider review / queues
    "total": "Under an hour"         // wall-clock, best honest range
  },
  "approval": {
    "required": false,
    "trigger": "None for private use...",   // what forces a review
    "wait": "n/a",
    "cost": "Free"
  },
  "prerequisites": ["A GitHub account", "..."],
  "steps": [
    {
      "title": "Register the app",
      "detail": "Go to Settings > Developer settings > GitHub Apps > New GitHub App...",
      "owner": "human",              // human | agent | either
      "time": "~10 min",
      "links": [{"label": "Register a GitHub App", "url": "https://docs.github.com/..."}]
    }
  ],
  "blockers": [
    {"title": "...", "detail": "...", "howToClear": "..."}
  ],
  "nuances": ["Things that surprise people, scope gotchas, quota notes..."],
  "agentCan": ["Generate the OAuth authorize/token URL pair", "..."],
  "humanMust": ["Click through the provider consent screen", "..."],
  "evidence": {
    "tested": ["Walked the registration form to the callback-URL field (no submit)", "..."],
    "sources": [{"label": "...", "url": "..."}],
    "unverified": ["Exact review wait time is not published by the provider"]
  },
  "lastVerified": "2026-10-19",
  "status": "published"              // published | draft (agent-generated, unreviewed)
}
```

## Difficulty rubric

| Level | Label | Meaning |
|---|---|---|
| 1 | Trivial | Register in a dashboard, no review, works immediately. |
| 2 | Easy | Register + a few config values; no approval, or approval is a formality. |
| 3 | Moderate | Multi-step config (domains, URLs, branding assets) and/or a light review with a short wait. |
| 4 | Hard | Formal review with real rejection risk, sensitive scopes, or weeks of waiting. |
| 5 | Arduous | Paid review, legal/security audit, or gating that blocks most small developers. |

## Effort honesty rules

- Split **active work** from **waiting**; never present a 20-minute form plus a
  3-week review as "3 weeks of work".
- Quote provider-stated times only. Otherwise say "not published" and give the
  range community reports, labeled as such, or "unknown".
- Every claim in a guide must trace to `evidence.sources` or `evidence.tested`.
  Anything else goes in `evidence.unverified`.
