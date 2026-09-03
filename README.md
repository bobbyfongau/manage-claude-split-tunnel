# manage-claude-split-tunnel

A [Claude Code](https://claude.com/claude-code) skill for when **Claude Code can't
connect from a region Anthropic blocks** — you get `API Error: 403 Request not
allowed` and a misleading "Please run /login".

**That is not an auth failure.** Your login is fine; the traffic just isn't leaving
an allowed region. Running `/login` cannot help.

The fix: run a local `sing-box` proxy over WireGuard to an allowed region, and point
**only Claude Code** at it via `settings.json` → `env` — so everything else on your
machine keeps its normal route (which matters if your local network is the one that
reaches sites the VPN can't).

## Why `settings.json`, not your shell

A `claude()` wrapper in `.zshrc` only works in Terminal. Warp and other launchers
start Claude without it and you get the 403 anyway. Claude Code reads `settings.json`
itself, so that layer works from any launcher.

## Install

```bash
git clone https://github.com/bobbyfongau/manage-claude-split-tunnel
cp -r manage-claude-split-tunnel ~/.claude/skills/
```

Then ask Claude: *"claude code says 403 request not allowed"*.

See [SKILL.md](SKILL.md) for the diagnose / rotate-config / rebuild procedures.

## Gotchas it records

- `sing-box` runs **userspace** WireGuard — it changes no system routing, so it can't break your connectivity
- `NO_PROXY` must exempt localhost or you break local dev servers
- `env` is read **only at startup** — restart Claude Code or the fix looks broken
- Probe `/v1/messages` (expect `405`); the API root returns 403/404 normally and is a useless test
- sing-box ≥1.11 uses `endpoints`, not `outbounds`, for WireGuard

## License

MIT
