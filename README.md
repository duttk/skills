# skills

Claude Code skills, packaged as plugins in a marketplace.

```
/plugin marketplace add duttk/skills
```

| Plugin | What it does | Install |
| --- | --- | --- |
| [create-pwa](plugins/create-pwa) | Interview-driven workflow: requirements → stack → `FEATURES.md` → UI design → push-to-deploy on your phone → build, test, verify. Default stack: Vite + Svelte + Dexie PWA, GitHub Actions, Cloudflare Workers/D1/R2/Access. | `/plugin install create-pwa@duttk-skills` |

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` and `plugins/<name>/skills/<name>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. Check: `claude plugin validate plugins/<name>` and `claude plugin validate .`
4. On changes, bump `version` in both files and tag: `claude plugin tag plugins/<name>`.
