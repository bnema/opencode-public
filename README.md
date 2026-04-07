# Public OpenCode Config

This repo is a public-safe copy of my OpenCode setup.

Contents:
- `opencode.json`
- `AGENTS.md` and `agents/`
- local `plugins/`
- reusable `skills/`

Notes:
- `opencode.json` still expects `CONTEXT7_API_KEY` to come from the environment.
- This setup uses my own Superpowers fork with additional features. The official Superpowers repository is `https://github.com/obra/superpowers`.
- This setup also includes the RTK plugin. The RTK repository is `https://github.com/rtk-ai/rtk`.
- The config currently keeps `"permission": { "*": "allow" }`. Tighten that before using it if you want a safer default.
