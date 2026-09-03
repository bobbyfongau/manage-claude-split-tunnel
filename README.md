# manage-claude-split-tunnel

A [Claude Code](https://claude.com/claude-code) skill for when **Claude Code can't
connect from a region Anthropic blocks** — you get `API Error: 403 Request not
allowed` and a misleading **"Please run /login"**.

**That is not an auth failure.** Your login is fine; your traffic just isn't leaving
an allowed region. Running `/login` cannot help and will not fix it.

## Is this actually your problem?

Run this first — it takes one second:

```bash
curl -o /dev/null -w '%{http_code}\n' https://api.anthropic.com/v1/messages
```

- **`403`** → you're region-blocked. This repo is for you.
- **`405`** → your network is fine (405 is the correct answer for a GET here).
  Your problem is something else; this won't help.

## What it does

Runs a local [`sing-box`](https://sing-box.sagernet.org/) proxy to an allowed region —
over OpenVPN via a [gluetun](https://github.com/qdm12/gluetun) container (static VPN
username/password, never expires, no `sudo`) or over WireGuard if your provider's
configs don't expire — and points **only Claude Code** at it via `settings.json` → `env`.
Everything else on your machine keeps its normal route — which matters if your local
network reaches sites the VPN can't.

**Fail-safe:** the proxy uses the tunnel while it is healthy and drops to a direct
connection by itself when the tunnel is dead (expired config, VPN outage). So on a
network that is not region-blocked, Claude keeps working even with a dead tunnel — you
never have to remove the proxy setting when you're not travelling.

## Why `settings.json`, not your shell

A `claude()` wrapper in `.zshrc` only works in Terminal. Warp and other launchers
start Claude without it and you still get the 403. Claude Code reads `settings.json`
itself, so that layer works from **any** launcher.

## Before you start

1. **A VPN account** — any provider gluetun supports (PureVPN, NordVPN, Mullvad, Proton, …)
   with OpenVPN username/password, or any non-expiring WireGuard config file
2. **macOS + Homebrew** — the commands assume it (sing-box runs on Linux too; adjust paths).
   The OpenVPN path adds `colima` + `docker` (installed by the skill, no admin password)
3. **Install it while Claude still works.** ⚠️ Chicken-and-egg: if Claude is already
   blocked it can't run, so the skill can't trigger. Either install it *before* you
   travel, or follow the steps in [SKILL.md](SKILL.md) by hand the first time — it's
   written to be readable without Claude. After that, the skill handles everything.

## Install

```bash
git clone https://github.com/bobbyfongau/manage-claude-split-tunnel
cp -r manage-claude-split-tunnel ~/.claude/skills/
```

Then ask Claude: *"claude code says 403 request not allowed"* — it will find the
skill and walk you through diagnose → install → configure → verify, and afterwards
handle credential rotation, rebuilds, and "works in Terminal but not Warp".

## Gotchas it records

- `sing-box` runs **userspace** WireGuard — it changes no system routing, so it can't break your connectivity
- `NO_PROXY` must exempt localhost or you break local dev servers
- `env` is read **only at startup** — restart Claude Code, or the fix looks broken when it isn't
- Probe `/v1/messages` (expect `405`); the API root returns 403/404 normally and is a useless test
- sing-box ≥1.11 uses `endpoints`, not `outbounds`, for WireGuard
- VPN provider WireGuard configs often **expire** — that's the most common later failure

## Note to other agents (Codex CLI, ChatGPT desktop, etc.)

If you're a different AI tool reading this repo because *you're* the one
region-blocked — don't port these commands literally. `settings.json → env`,
`/opt/homebrew` vs `/usr/local`, and `sing-box`'s config schema are Claude Code
and macOS/Homebrew specifics. What should transfer is the **pattern**:

1. **A region block is not an auth failure.** Confirm it against your own
   provider's real API endpoint, expecting the specific "reachable but wrong
   method" code for that endpoint (Anthropic's is `405` on `POST /v1/messages`
   hit with `GET` — look up your own provider's equivalent, don't assume 405).
   If you get that code, the network is fine and this whole pattern doesn't apply.
2. **Fix your own process, not the whole machine.** Find where *you* read your
   own startup environment or config — a CLI tool (Codex CLI included) most
   likely reads `HTTPS_PROXY`/`HTTP_PROXY` from its process env at launch, or
   has its own settings file. A `.zshrc`/`.bashrc` export only reaches terminal
   launches — it won't help a Dock icon, Spotlight, or Warp-style launcher that
   starts you without sourcing the shell profile. Find your equivalent of
   `settings.json`, don't rely on the shell.
3. **A GUI app is a different mechanism, not a bigger version of the same one.**
   ChatGPT desktop won't read a CLI env var or config file at all. Electron/
   native macOS apps generally pick up the **OS-level network proxy** (System
   Settings → Network → Proxies) automatically instead. That's a real tradeoff,
   not a free upgrade: it also routes Safari, Mail, and anything else that
   honors the system proxy. Say so explicitly and get it confirmed rather than
   flipping it silently — don't assume the blast radius is welcome.
4. **Reuse the tunnel, don't stand up a second one.** If `sing-box` is already
   running on `127.0.0.1:7890` for Claude Code, point your own proxy setting at
   that same port instead of provisioning another WireGuard instance.
5. **Never let the raw WireGuard file pass through chat.** Ask the user for the
   *file path*, read the private key from disk yourself, mask it in anything
   you print. This one isn't tool-specific — it holds for any agent doing this
   kind of setup.
6. **Env is almost always read once, at process startup.** After wiring in a
   proxy, fully quit and relaunch the tool being fixed — a session that was
   already running when you made the change won't pick it up.
7. **The VPN config will expire.** Whatever you build, give it (or the human)
   an explicit "rotate the config" step — don't treat first-run success as done
   forever.

- `ECONNREFUSED 127.0.0.1:7890` means sing-box itself is stopped, nothing else —
  `brew services start sing-box`. Deleting the proxy setting to "fix" it only hides
  the problem until the next blocked network (this happened on day one).
- PureVPN WireGuard configs expire after **15 minutes** (24 h with a paid add-on) — use
  OpenVPN there; the skill runs it in gluetun so nothing on the Mac needs root

## License

MIT
