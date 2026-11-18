# create-pwa

A Claude Code plugin with one skill, **create-pwa**: an interview-driven workflow that takes you
from "I want an app that…" to an installable progressive web app on your phone, deployed on
every push.

1. **Interview** about what the app does, who uses it, offline needs, data and sign-in.
2. **Stack proposal.** Default: Vite + Svelte 5 + TypeScript, `vite-plugin-pwa`, Dexie;
   a Cloudflare Worker with D1, R2 and Cloudflare Access; Vitest + Playwright; GitHub Actions
   that test, then deploy.
3. **`docs/FEATURES.md`** written from the interview and kept up to date.
4. **UI design** at your level of involvement: options on a design canvas, one proposal, or
   "you decide".
5. **Push-to-deploy to your phone first**, before any features.
6. **Build, test, verify** each feature end to end, including a Safari-like storage test
   fixture, then offer next steps.

Dashboard walkthroughs are checked against current docs before they're given.

## Install

```
/plugin marketplace add duttk/skills
/plugin install create-pwa@duttk-skills
```

Then ask Claude Code for a new app, e.g. _"I want a PWA to track my plant watering"_, or
invoke it directly with `/create-pwa`.

The skill was developed alongside a worked example, `duttk/recipes` (private). Everything the
skill needs is in its reference files.
