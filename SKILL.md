---
name: manage-claude-split-tunnel
description: Diagnose and repair Claude Code's connection when it fails from a Hong Kong / China network — "API Error 403 Request not allowed", "Please run /login", "ECONNREFUSED 127.0.0.1:7890", works in Terminal but not Warp, or Claude works but gitee/mainland sites don't. Also sets up or rebuilds the split tunnel that sends ONLY Claude through PureVPN Singapore (OpenVPN in a gluetun container behind a local sing-box proxy that falls back to direct by itself) while everything else goes direct, and rotates the PureVPN credentials.
---

# Manage Claude split tunnel

Your network reaches mainland China but **Anthropic region-blocks your IP**.
The reverse is also true of a full VPN. This skill owns that conflict.

**The single most important fact: `403 Request not allowed` + "Please run /login"
is NOT an auth problem.** The login is fine. Traffic just isn't leaving Singapore.
Never run `/login` for it — it cannot help and wastes a re-auth.

## Parameters

- `MODE`: `diagnose` (default) | `rotate-creds` | `rebuild` | `teardown`
- `PROXY_PORT`: `7890` (Claude's local proxy) — `7891` is the tunnel-only test port
- `CREDS_FILE`: `~/.config/gluetun/gluetun.env` (chmod 600, holds the PureVPN
  **manual-config** VPN username/password from
  https://my.purevpn.com/v2/dashboard/manual-config — not the account login)

## Architecture (why it's built this way)

```
Claude Code ──HTTPS_PROXY──> sing-box 127.0.0.1:7890 ─┬─ ovpn   = gluetun HTTP proxy 127.0.0.1:8888
                                                      │          (colima VM → OpenVPN → PureVPN SG)   preferred
                                                      └─ direct  fallback, automatic when the tunnel is dead
everything else ──────────────────────────────────────────────────> direct (China OK)
127.0.0.1:7891 = tunnel-only test port (used by vpncheck; never point Claude at it)
```

- **Why OpenVPN, not WireGuard (2026-09-03).** PureVPN's WireGuard manual configs
  expire after **15 minutes**; the paid "Extend" add-on only reaches 24 hours. The
  first WireGuard build died the same afternoon. OpenVPN uses the static manual
  username/password and never expires.
- **Why gluetun in a colima VM.** sing-box cannot speak OpenVPN. A native `openvpn`
  needs root for the tun device plus scoped-route tricks to stay split. gluetun has
  PureVPN built in (server list, kill switch, HTTP proxy, auto-reconnect), runs
  without `sudo` in a 1 CPU / 1 GB colima VM, and the Mac only ever sees
  `127.0.0.1:8888`. Legacy WireGuard config kept at
  `~/.config/sing-box/config.wireguard-legacy.json` for reference only.
- **The proxy is fail-safe.** `route.final` is a `urltest` group `[ovpn, direct]`
  with a 30 s tolerance: it uses the tunnel while a health check every minute
  passes, and drops to `direct` on its own the moment it fails. A dead tunnel no
  longer strands Claude on networks that are not region-blocked (phone hotspot,
  a router-level VPN). **Leave the `env` block, sing-box, colima and gluetun on
  permanently — that is the design.** The only thing that still strands Claude is
  sing-box itself being stopped: `ECONNREFUSED 127.0.0.1:7890`.
- **Fallback is sticky, so a self-heal agent exists.** Once the group has dropped
  to `direct` it stays there until sing-box restarts. LaunchAgent
  `local.singbox-heal` runs `~/.config/sing-box/heal.sh` every 10 min: if
  7891 (tunnel) is alive but 7890 exits elsewhere, it restarts sing-box.
- **`$PREFIX` = `brew --prefix`, resolve it before using any path on a new
  machine.** Apple Silicon → `/opt/homebrew`; Intel → `/usr/local`. All paths below
  are written as `$PREFIX/...`. Getting this wrong is the most likely first-run
  failure on an Intel Mac (confirmed on a MacBookPro13,3, which is `/usr/local`).
- **sing-box changes no system routing** and colima is a plain user VM, so nothing
  here can break your connectivity. Safe to restart at any time.
- **The proxy is set in `settings.json` → `env`, not the shell.** A `.zshrc`
  function only works in Terminal; Warp launches Claude without it and gets the
  403. Claude Code reads `settings.json` itself, so the env layer works from Warp,
  Terminal, an alias, `caffeinate`, or Spotlight alike.
- **PureVPN SG also reaches mainland China** (gitee.com, docs.openluat.com,
  example.cn all 200), so routing everything through it is a valid fallback.
- **If your router is itself on the same VPN**, `direct IP` == tunnel exit on that
  Wi-Fi. That is not a leak; only a plain network shows a difference.

## Steps — MODE=diagnose

1. Run `vpncheck` (defined in `~/.zshrc`). Healthy output:
   ```
   sing-box  : UP
   direct IP : <your-direct-ip>                (whatever the Wi-Fi gives)
   tunnel    : ALIVE (exit <your-vpn-exit-ip>) (PureVPN SG)
   claude via: TUNNEL
   claude    : 405   (405 = OK, 403 = region-blocked, 000 = dead)
   ```
2. Match the symptom:

| Symptom | Cause | Fix |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:7890`, `OAuth error: connect ECONNREFUSED`, `sing-box: DOWN` | sing-box service stopped (it never stops by itself; someone ran `brew services stop`) | `brew services start sing-box`, then fully restart Claude. **Do not delete the `env` block** — that was the 2026-09-03 mistake |
| `tunnel: DEAD`, `claude via: DIRECT` | colima VM or gluetun down, or PureVPN creds changed. Claude still works on non-blocked networks, 403 on your blocked home Wi-Fi | `colima status` → `colima start`; `docker ps` → `docker start gluetun`; `docker logs --tail 30 gluetun` (AUTH_FAILED = `MODE=rotate-creds`) |
| `tunnel: ALIVE`, `claude via: DIRECT` | Group stuck on fallback (heal agent fixes within 10 min) | `brew services restart sing-box` now |
| `claude: 403` with `claude via: TUNNEL` | PureVPN exit itself refused (rare) | `docker restart gluetun` picks another SG server; or set `SERVER_CITIES` in the env file |
| `claude: 403`, `vpncheck` healthy | Claude not using the proxy: `env` missing (step 3), or that window started before the setting existed | Fix `env`, fully restart Claude |
| Works in Terminal, not Warp | Fix was only in `.zshrc` | Move it to `settings.json` → `env` |
| Claude fine, mainland sites dead | A router-level VPN is on | Switch back to the non-VPN Wi-Fi |
| Claude works direct on a phone hotspot | mobile data is not region-blocked | Nothing to fix — the fallback is doing its job |

3. Verify the env layer is actually in place:
   ```bash
   python3 -c "import json;print(json.load(open('~/.claude-config/settings.json'))['env'])"
   ```
   Must contain `HTTPS_PROXY`, `HTTP_PROXY` = `http://127.0.0.1:7890`, and
   `NO_PROXY` including `127.0.0.1` (or a local service on :<your-local-port> breaks).
   Also `readlink ~/.claude/settings.json` must print
   `~/.claude-config/settings.json`. Empty output = the symlink was
   replaced by a plain file (see Gotchas) and edits are landing in the wrong file.

4. **The definitive test** — strips the proxy from the environment *and* skips
   `.zshrc`, so it reproduces the exact Warp condition:
   ```bash
   env -u HTTPS_PROXY -u HTTP_PROXY -u NO_PROXY bash -c \
     'claude -p "Reply with exactly: TUNNEL_OK"'
   ```
   `TUNNEL_OK` = settings.json env is doing the work; Warp will work.

## Steps — MODE=rotate-creds

The PureVPN manual VPN password was pasted into a chat once (2026-09-03), so it
should be regenerated when convenient. **Ask for the file path or have the user edit
the file — never ask for the password in chat.**

1. Regenerate the manual-config password on the PureVPN dashboard.
2. Edit `OPENVPN_PASSWORD=` in `~/.config/gluetun/gluetun.env` (keep it `chmod 600`).
3. `docker restart gluetun && sleep 30 && vpncheck` → must show `tunnel: ALIVE`.

## Steps — MODE=rebuild (new machine, or from scratch)

0. `brew --prefix` → set `$PREFIX` for every path below. Don't assume
   `/opt/homebrew` — Intel Macs are `/usr/local`.
1. `brew install sing-box colima docker`
2. `colima start --cpu 1 --memory 1 --disk 10` (first run downloads a ~600 MB
   image from GitHub — an `unexpected EOF` just means run it again), then
   `brew services start colima` so it comes up at login.
3. Write `~/.config/gluetun/gluetun.env` (`chmod 600`):
   ```
   VPN_SERVICE_PROVIDER=purevpn
   VPN_TYPE=openvpn
   OPENVPN_USER=<manual-config username, purevpn0s…>
   OPENVPN_PASSWORD=<manual-config password>
   SERVER_COUNTRIES=Singapore
   HTTPPROXY=on
   HTTPPROXY_LISTENING_ADDRESS=:8888
   HTTPPROXY_LOG=off
   TZ=<your-timezone>
   ```
4. Start the tunnel container (restarts with the VM):
   ```bash
   docker run -d --name gluetun --restart unless-stopped --cap-add NET_ADMIN \
     --device /dev/net/tun -p 127.0.0.1:8888:8888 \
     --env-file ~/.config/gluetun/gluetun.env qmcgaw/gluetun:latest
   ```
   Wait for `docker logs gluetun` to say `Initialization Sequence Completed`, then
   `curl -x http://127.0.0.1:8888 https://api.ipify.org` must print a Singapore IP.
5. Write `$PREFIX/etc/sing-box/config.json` (`chmod 600`; pristine copy at
   `~/.config/sing-box/config.json`):
   ```json
   {
     "log": {"level": "warn"},
     "inbounds": [
       {"type": "mixed", "tag": "local-proxy", "listen": "127.0.0.1", "listen_port": 7890},
       {"type": "mixed", "tag": "tunnel-only", "listen": "127.0.0.1", "listen_port": 7891}
     ],
     "outbounds": [
       {"type": "urltest", "tag": "auto", "outbounds": ["ovpn", "direct"],
        "url": "https://www.gstatic.com/generate_204", "interval": "1m", "tolerance": 30000},
       {"type": "http", "tag": "ovpn", "server": "127.0.0.1", "server_port": 8888},
       {"type": "direct", "tag": "direct"}
     ],
     "route": {"rules": [{"inbound": ["tunnel-only"], "outbound": "ovpn"}], "final": "auto"}
   }
   ```
6. `sing-box check -c <file>` — **always validate before starting** — then
   `brew services start sing-box` (installs a LaunchAgent → survives reboot).
7. Install the self-heal agent: `~/.config/sing-box/heal.sh` (curl 7891 vs 7890,
   restart sing-box if they differ while 7891 is alive) and
   `~/Library/LaunchAgents/local.singbox-heal.plist` with
   `StartInterval` 600; `launchctl bootstrap gui/$(id -u) <plist>`.
8. Add the `env` block to `settings.json` (see Gotchas — it's a symlink).
9. **Add `vpncheck` to `~/.zshrc`** — `MODE=diagnose` assumes it exists:
    ```bash
    vpncheck() {
      local direct tunnel proxied code
      if lsof -nP -iTCP:7890 -sTCP:LISTEN >/dev/null 2>&1; then echo "sing-box  : UP"
      else echo "sing-box  : DOWN   -> brew services start sing-box"; return 1; fi
      direct=$(curl -s -m 8 https://api.ipify.org)
      tunnel=$(curl -s -m 12 -x http://127.0.0.1:7891 https://api.ipify.org)
      proxied=$(curl -s -m 12 -x http://127.0.0.1:7890 https://api.ipify.org)
      code=$(curl -s -m 15 -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}' https://api.anthropic.com/v1/messages)
      echo "direct IP : ${direct:-?}"
      if [ -n "$tunnel" ]; then echo "tunnel    : ALIVE (exit $tunnel)"
      else echo "tunnel    : DEAD   -> colima status; docker logs gluetun  (skill: manage-claude-split-tunnel)"; fi
      if [ -z "$proxied" ]; then echo "claude via: NOTHING (proxy broken)"
      elif [ "$proxied" = "$tunnel" ]; then echo "claude via: TUNNEL"
      else echo "claude via: DIRECT (fallback)$([ -n "$tunnel" ] && echo '  -> tunnel is back: brew services restart sing-box')"; fi
      echo "claude    : $code   (405 = OK, 403 = region-blocked, 000 = dead)"
    }
    ```
10. **Fully quit and reopen Claude Code** — in Warp, Terminal, or wherever it runs.
    `settings.json` → `env` is read at startup, so a session already running keeps
    the old environment and keeps failing with 403. `/exit` then relaunch.
11. Run the definitive test from `MODE=diagnose` step 4.

## Steps — MODE=teardown (only if the tunnel is genuinely no longer wanted)

"Not travelling" is not a reason: the proxy falls back to direct by itself. If it
really must go, do **all** of it, in this order:

1. Remove the `env` block with a **targeted edit on the real path**
   `~/.claude-config/settings.json` (Edit tool, or `jq` writing back to that same
   path). Never `jq … > tmp && mv tmp ~/.claude/settings.json` — that replaces
   the symlink.
2. `brew services stop sing-box`; `launchctl bootout gui/$(id -u)/local.singbox-heal`
3. `docker rm -f gluetun; brew services stop colima; colima stop`

Doing only part of it is exactly the 2026-09-03 incident.

## Gotchas that cost real time

- **Never `mv` a file over `~/.claude/settings.json`.** On 2026-09-03
  `jq 'del(.env)' … > /tmp/x && mv /tmp/x ~/.claude/settings.json` replaced the
  symlink with a plain file. From then on `/model`, `/effort` and `/config` wrote
  to a detached copy while `~/.claude-config` (and every other Mac) silently kept
  the old one. Check: `readlink ~/.claude/settings.json`. Repair:
  ```bash
  jq -n --slurpfile a ~/.claude/settings.json --slurpfile b ~/.claude-config/settings.json \
    '{env: $b[0].env} + $a[0]' > merged.json && jq -e . merged.json >/dev/null \
  && cp merged.json ~/.claude-config/settings.json \
  && mv ~/.claude/settings.json ~/.claude/settings.json.detached \
  && ln -s ~/.claude-config/settings.json ~/.claude/settings.json
  ```
- **Removing the `env` block is never the fix for a dead tunnel.** It only
  "works" on networks that are not region-blocked and leaves Claude dead again on
  your blocked home Wi-Fi. Fix the tunnel — or do nothing: the proxy falls back on its own.
- **`ECONNREFUSED 127.0.0.1:7890` means sing-box is stopped, nothing else.** It
  never stops by itself (launchd keeps it alive); someone ran `brew services stop`.
- **Don't reach for PureVPN WireGuard again.** 15-minute expiry, 24 h max with a
  paid add-on. IKEv2/IPsec on macOS are whole-Mac tunnels, not split. OpenVPN via
  gluetun is the only PureVPN protocol that is both permanent and containable.
- **`~/.claude/settings.json` is a symlink** to `~/.claude-config/settings.json`.
  Edits must target the real path or they're refused.
- **A PreToolUse hook blocks whole-file `Write` to settings.json** (it once wiped
  the config). Use the `Edit` tool for targeted changes.
- **`~/.claude-config` is a git repo that auto-pulls (`--ff-only`) at SessionStart.**
  The committed proxy `env` reaches every machine — each one needs sing-box on
  7890 (with this fallback design, sing-box alone is enough to stay safe).
- **Never switch the network or stop the tunnel mid-session** — it kills the live
  Claude connection. Set the replacement up first, then switch.
- **A shell function is the wrong layer.** It shadows `claude`, and Warp never
  sees it. `settings.json` → `env` is the correct fix.
- **`env` is read only at startup.** After editing `settings.json`, a running
  Claude Code session keeps the old environment and keeps returning 403. Quit and
  relaunch before concluding the fix didn't work.
- `curl https://api.anthropic.com/` (root) returns 403/404 normally — that is
  **not** a useful test. Always probe `/v1/messages`: `405` means healthy.

## Output format

Report: `vpncheck` output verbatim, the symptom matched from the table, the fix
applied, and the definitive-test result. Never print the PureVPN password — say
which file it lives in.

## Note

Written for PureVPN Singapore + macOS/Homebrew, but nothing here is provider-specific:
gluetun supports most VPN providers (or bring your own OpenVPN/WireGuard config), and
`sing-box` runs on Linux too. Replace the `<placeholders>` with your own values. The
`~/.claude-config` symlink gotchas apply only if you keep `settings.json` in a synced
repo like the author does.
