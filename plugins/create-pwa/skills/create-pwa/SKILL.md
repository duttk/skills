---
name: create-pwa
description: Interview-driven workflow to plan, design, build, test and ship a progressive web app (PWA) that the user can install and test on their own phone, with push-to-deploy. Use when the user wants to create a new PWA or web app from scratch ("build me an app for…", "I want a PWA that…"), or to restart planning one.
---

# Create a PWA

Run this as a conversation, phase by phase. **Ask, propose, confirm, then build.** Never skip
ahead to code before the user has agreed to the features and the stack.

Let the user choose how involved they are. In the first message, offer:

- **Hands-on:** you ask about each decision and show options before choosing.
- **Hands-off:** you make sensible choices, state them in one line each, and only stop for
  decisions that are truly theirs (what the app does, who uses it, accounts and money).

Keep honouring that choice. If they answer "you decide", decide, say what you picked, and move on.

Use the AskUserQuestion tool for every multiple-choice decision (up to 4 questions per call,
recommended option first, labelled "(Recommended)"). Free-text questions go in plain chat.

References (read when you reach the phase):

- `reference/stack.md`: the default stack, project layout, CI and Cloudflare deploy setup.
- `reference/pitfalls.md`: problems already hit building PWAs this way, and the test that
  catches each one. **Read before implementing.**

The `duttk/recipes` repository is a working example of everything here.

---

## Phase 1: Interview for requirements

Goal: understand the app well enough to write the feature list. Ask in **two rounds** at most,
not twenty questions.

**Round 1 (chat, open-ended):** what the app is for, who uses it (just them? a household?
the public?), and the 3–5 things it must do. Ask for an example of a typical session.

**Round 2 (AskUserQuestion), only what Round 1 left open:**

| Question                                                                     | Why it matters                       |
| ---------------------------------------------------------------------------- | ------------------------------------ |
| Offline: must it work with no signal?                                        | Local-first storage vs. server-first |
| Devices: one phone, several devices, or shared between people?               | Sync and accounts                    |
| Data: text only, photos, audio, files? Roughly how much?                     | Storage choice and limits            |
| Sign-in: nobody, a fixed small group, or anyone can sign up?                 | Cloudflare Access vs. real auth      |
| Phone features: camera, notifications, location, screen kept awake, sharing? | iOS support varies                   |
| Platforms: iPhone, Android, desktop?                                         | iOS has the most PWA limits          |

Then **estimate** what matters for the stack (storage size, number of users, requests per day)
and show the arithmetic briefly. Example from the recipes app: 100 recipes with photos ≈ 0.2 GB.

## Phase 2: Propose the stack

Default (use unless a requirement rules it out):

- **App:** Vite + Svelte 5 + TypeScript, `vite-plugin-pwa` (Workbox), Dexie (IndexedDB).
- **Sync/back end:** one Cloudflare Worker (serves the app and `/api`), D1 database,
  R2 for files, Cloudflare Access for sign-in of a fixed group.
- **Quality:** Vitest, Playwright (Chromium desktop + iPhone viewport, WebKit in CI),
  Prettier, svelte-check.
- **Delivery:** GitHub Actions: checks and tests on every push, then deploy to Cloudflare.

Show the stack as a short table with one line of "why" per piece. Only deviate when a
requirement forces it, and say which one. For example:

| Requirement                       | Change                                                               |
| --------------------------------- | -------------------------------------------------------------------- |
| Anyone can sign up / public users | Real auth (e.g. a hosted auth provider) instead of Cloudflare Access |
| Real-time collaboration           | Durable Objects or a sync engine instead of simple pull/push         |
| No sync, single device            | Drop Worker/D1/R2; deploy static assets only (still on Cloudflare)   |
| Very large files or video         | R2 with direct uploads; check Worker request size limits             |

If free tiers matter, check current limits before quoting them; if you can't check, say the
numbers are from memory. Confirm the stack with the user before continuing.

## Phase 3: Write FEATURES.md

Create `docs/FEATURES.md` in the repo from the interview, and commit it:

```markdown
# Features

Status: ⬜ todo · 🟨 in progress · ✅ done

| #   | Feature | Status |
| --- | ------- | ------ |
| 1   | …       | ⬜     |

## Later

- …

## Decisions

- Stack: … (why)
- Sign-in: …
- Data: … (estimate)
```

Rules: must-haves in the table, numbered; "should"/"later" items in **Later**; each feature
testable as written (avoid "nice UX"). Show the list to the user, adjust, then commit.
Keep it updated as work progresses: 🟨 when started, ✅ only when tested end to end.

## Phase 4: UI design

Ask how involved they want to be in design (AskUserQuestion): _show me options_ /
_one proposal, I'll tweak_ / _you decide_.

- **Options:** make a design canvas (Artifact tool, `quickstart` with intent `design`)
  with **3 distinct directions** (different type, colour and layout ideas, not three shades
  of one idea). Each shows the 3–4 key screens of the must-have features at phone size
  (390×844). Then offer palette variations of the chosen direction.
- **One proposal:** one direction on a canvas; iterate on their comments.
- **You decide:** pick a direction suited to the app, describe it in two lines, build it.

In every case: phone-first, tap targets ≥ 44 px, text contrast ≥ 4.5:1, light and dark,
no emoji as icons, real buttons and labels. Record the decision (direction, palette, fonts,
link to the canvas) in `docs/DESIGN.md`. Self-host fonts (Fontsource) so they work offline.

## Phase 5: Scaffold and push-to-deploy, before features

Get an installable shell onto the user's phone **first**, so every later change can be
tried on the real device. Follow `reference/stack.md`:

1. Scaffold the app, PWA manifest and icons, a minimal screen, unit + e2e test setup, CI.
2. Commit and push to the working branch.
3. Walk the user through Cloudflare setup (account, `wrangler login`, D1, R2, first
   `npm run deploy`, Access with their email addresses).
4. Walk them through GitHub secrets so CI deploys after tests pass.
5. Have them open the URL on their phone, sign in, **Add to Home Screen**, and confirm.

Until step 5 works, don't build features. Debug setup with them using their exact error text.

### Walking the user through dashboards: verify every breadcrumb

Cloudflare, GitHub and phone settings menus change often. Before telling the user a path like
_Zero Trust → Access → Applications_:

1. **Check it against current documentation** (fetch the vendor's docs page). If the docs host
   is blocked by the network, ask to allow it (environment tools) rather than guessing.
2. If you can't check, **say so** ("from memory, labels may differ"), give the most likely
   path _and_ an alternative, and suggest the dashboard's search box with the exact term.
3. When the user reports a missing menu, don't repeat the same path: ask what they see
   (top-level sidebar names) and adapt.
4. Prefer CLI steps (`wrangler …`, `gh`-free GitHub UI paths) over dashboard clicks where both work.

## Phase 6: Implement features

Work through `docs/FEATURES.md` in order of dependency, one feature per commit:

1. Read `reference/pitfalls.md` for anything the feature touches (storage, photos, sync,
   sign-in, iOS).
2. Implement. Keep data access behind one repository module so sync can be added later.
3. Unit-test the logic; e2e-test the user flow (including offline and two-device sync when
   relevant). Run every test under the **Safari-like fixture**.
4. **Prove the tests can fail:** break the feature on purpose once and see the test catch it.
5. Look at the screens yourself (phone-size screenshots, light and dark) and fix what looks wrong.
6. Commit and push; CI deploys. Mark the feature ✅ only after this.

Report honestly: what passed, what you couldn't test (real iPhone Safari, hardware features),
and anything that only works in Chromium.

## Phase 7: Verify end to end, then offer next steps

- Run the whole suite locally and check the CI run (including the WebKit job and the deploy).
- Ask the user to try the deployed app on their phone(s) with a short checklist specific to
  the features (e.g. airplane mode, install, two-device sync).
- Offer next steps: the **Later** items from FEATURES.md, polish found while testing, and
  hardening (backup/export, error reporting). Ask which to do next.
