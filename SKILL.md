---
name: manage-claude-split-tunnel
description: Diagnose and repair Claude Code's connection when it fails from a Hong Kong / China network — "API Error 403 Request not allowed", "Please run /login", "ECONNREFUSED 127.0.0.1:7890", works in Terminal but not Warp, or Claude works but gitee/mainland sites don't. Also sets up or rebuilds the sing-box split tunnel that sends ONLY Claude through Surfshark Singapore over WireGuard (falling back to direct by itself when the tunnel is dead) while everything else goes direct, and swaps in a new WireGuard config.
---

# Manage Claude split tunnel

Your network reaches mainland China but **Anthropic region-blocks your IP**.
The reverse is also true of a full VPN. This skill owns that conflict.

**The single most important fact: `403 Request not allowed` + "Please run /login"
is NOT an auth problem.** The login is fine. Traffic just isn't leaving Singapore.
Never run `/login` for it — it cannot help and wastes a re-auth.

## Parameters

- `MODE`: `diagnose` (default) | `rotate-config` | `rebuild` | `teardown`
- `PROXY_PORT`: `7890` (Claude's local proxy) — `7891` is the tunnel-only test port
- `WG_SOURCE`: Surfshark account → VPN → Manual setup → WireGuard → key pair →
  Locations → Singapore → download `.conf` (e.g. `~/Downloads/sg-sng.conf`)

## Architecture (why it's built this way)

```
Claude Code ──HTTPS_PROXY──> sing-box 127.0.0.1:7890 ─┬─ wg-sg  (userspace WireGuard → Surfshark SG)  preferred
                                                      └─ direct  fallback, automatic when the tunnel is dead
everything else ──────────────────────────────────────────────────> direct (China OK)
127.0.0.1:7891 = tunnel-only test port (used by vpncheck; never point Claude at it)
```

- **Provider: Surfshark, WireGuard, manual key pair.** Keys are generated once in
  the account page and stay valid until deleted. This replaced PureVPN on
  2026-09-03: PureVPN WireGuard manual configs expire after **15 minutes** (24 h max
  with a paid add-on), and its OpenVPN path needed a gluetun container in a colima
  VM — both torn down the same night. Don't go back to either.
- **`$PREFIX` = `brew --prefix`, resolve it before using any path on a new
  machine.** Apple Silicon → `/opt/homebrew`; Intel → `/usr/local`. All paths below
  are written as `$PREFIX/...`. Getting this wrong is the most likely first-run
  failure on an Intel Mac (confirmed on a MacBookPro13,3, which is `/usr/local`).
- **sing-box runs userspace WireGuard.** It changes *no* system routing, so it
  physically cannot break your connectivity. Safe to restart at any time.
- **The proxy is fail-safe.** `route.final` is a `urltest` group `[wg-sg, direct]`
  with a 30 s tolerance: it uses the tunnel while a health check every minute
  passes, and drops to `direct` on its own the moment it fails. A dead tunnel no
  longer strands Claude on networks that are not region-blocked (phone hotspot,
  a router-level VPN). **Leave the `env` block and sing-box on permanently — that
  is the design.** The only thing that still strands Claude is sing-box itself
  being stopped: `ECONNREFUSED 127.0.0.1:7890`.
- **Fallback is sticky, so a self-heal agent exists.** Once the group has dropped
  to `direct` it stays there until sing-box restarts. LaunchAgent
  `local.singbox-heal` runs `~/.config/sing-box/heal.sh` every 10 min: if
  7891 (tunnel) is alive but 7890 exits elsewhere, it restarts sing-box. It also
  appends `tunnel=<ip|DEAD>` to `~/.config/sing-box/heal.log` — read that to see
  whether the key ever stopped working.
- **The proxy is set in `settings.json` → `env`, not the shell.** A `.zshrc`
  function only works in Terminal; Warp launches Claude without it and gets the
  403. Claude Code reads `settings.json` itself, so the env layer works from Warp,
  Terminal, an alias, `caffeinate`, or Spotlight alike.
- **Surfshark SG also reaches mainland China** (gitee.com, docs.openluat.com,
  example.cn all 200 through it), so routing everything through it is a valid
  fallback.

## Steps — MODE=diagnose

1. Run `vpncheck` (defined in `~/.zshrc`). Healthy output:
   ```
   sing-box  : UP
   direct IP : <your-direct-ip>                (whatever the Wi-Fi gives)
   tunnel    : ALIVE (exit <your-vpn-exit-ip>)  (Surfshark SG; varies)
   claude via: TUNNEL
   claude    : 405   (405 = OK, 403 = region-blocked, 000 = dead)
   ```
2. Match the symptom:

| Symptom | Cause | Fix |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:7890`, `OAuth error: connect ECONNREFUSED`, `sing-box: DOWN` | sing-box service stopped (it never stops by itself; someone ran `brew services stop`) | `brew services start sing-box`, then fully restart Claude. **Do not delete the `env` block** — that was the 2026-09-03 mistake |
| `tunnel: DEAD`, `claude via: DIRECT` | WireGuard not handshaking: key deleted in the Surfshark account, endpoint changed, or UDP 51820 blocked on this network. Claude still works on non-blocked networks, 403 on your blocked home Wi-Fi | `MODE=rotate-config` (new key pair + `.conf`). Check `~/.config/sing-box/heal.log` for when it died |
| `tunnel: ALIVE`, `claude via: DIRECT` | Group stuck on fallback (heal agent fixes within 10 min) | `brew services restart sing-box` now |
| `claude: 403` with `claude via: TUNNEL` | Surfshark exit itself refused (rare) | Download a different Singapore `.conf` (other server) |
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

## Steps — MODE=rotate-config (new key pair or new server)

**Ask for the file path, never for the contents.** Read the values from the file
directly. Pasting a config into chat writes a live private key into the transcript.

1. Surfshark account → VPN → Manual setup → WireGuard → **"I don't have a
   key pair"** → name it → generate. Then **Locations** → Singapore → download.
   One key pair per client: never reuse a key that a router or another Mac uses.
2. Patch the live config in place — never retype secrets, never echo the key:
   ```bash
   python3 - <<'PY'
   import json, os
   src = os.path.expanduser("~/Downloads/sg-sng.conf")   # <-- new file
   kv = {}
   for line in open(src):
       line = line.strip()
       if "=" in line and not line.startswith("["):
           k, v = line.split("=", 1); kv[k.strip().lower()] = v.strip()
   host, port = kv["endpoint"].rsplit(":", 1)
   addr = [a.strip() if "/" in a else a.strip()+"/32" for a in kv["address"].split(",")]
   dst = "$PREFIX/etc/sing-box/config.json"
   cfg = json.load(open(dst)); ep = cfg["endpoints"][0]
   ep["address"] = addr; ep["private_key"] = kv["privatekey"]
   ep["peers"][0].update({"address": host, "port": int(port), "public_key": kv["publickey"]})
   json.dump(cfg, open(dst, "w"), indent=2); print("updated", dst)
   PY
   sing-box check -c $PREFIX/etc/sing-box/config.json \
   && brew services restart sing-box && sleep 8 && vpncheck
   ```
3. Mirror it to the backup at `~/.config/sing-box/config.json`, `chmod 600` both
   and the `.conf`. `vpncheck` must show `tunnel: ALIVE` **and** `claude via: TUNNEL`.

Field mapping, for checking by hand:

| WireGuard `.conf` | sing-box |
|---|---|
| `PrivateKey` | `endpoints[0].private_key` |
| `Address` | `endpoints[0].address` (CIDR; append `/32` if bare) |
| `PublicKey` | `endpoints[0].peers[0].public_key` |
| `Endpoint` host / port | `peers[0].address` / `peers[0].port` (split on the last `:`) |
| `AllowedIPs` | `peers[0].allowed_ips` |
| `DNS` | unused — sing-box resolves via the system |

## Steps — MODE=rebuild (new machine, or from scratch)

0. `brew --prefix` → set `$PREFIX` for every path below. Don't assume
   `/opt/homebrew` — Intel Macs are `/usr/local`.
1. `brew install sing-box`
2. Get a `.conf` from `WG_SOURCE` (a **new** key pair for this machine).
3. Write `$PREFIX/etc/sing-box/config.json` (`chmod 600`; pristine copy at
   `~/.config/sing-box/config.json`) — placeholders, not real values:
   ```json
   {
     "log": {"level": "warn"},
     "inbounds": [
       {"type": "mixed", "tag": "local-proxy", "listen": "127.0.0.1", "listen_port": 7890},
       {"type": "mixed", "tag": "tunnel-only", "listen": "127.0.0.1", "listen_port": 7891}
     ],
     "endpoints": [{"type": "wireguard", "tag": "wg-sg", "system": false, "mtu": 1408,
       "address": ["<Address, CIDR>"], "private_key": "<PrivateKey>",
       "peers": [{"address": "<Endpoint host>", "port": 51820, "public_key": "<PublicKey>",
                  "allowed_ips": ["0.0.0.0/0"], "persistent_keepalive_interval": 25}]}],
     "outbounds": [
       {"type": "urltest", "tag": "auto", "outbounds": ["wg-sg", "direct"],
        "url": "https://www.gstatic.com/generate_204", "interval": "1m", "tolerance": 30000},
       {"type": "direct", "tag": "direct"}
     ],
     "route": {"rules": [{"inbound": ["tunnel-only"], "outbound": "wg-sg"}], "final": "auto"}
   }
   ```
   sing-box ≥1.11 uses `endpoints`, not `outbounds`, for WireGuard.
4. `sing-box check -c <file>` — **always validate before starting** — then
   `brew services start sing-box` (installs a LaunchAgent → survives reboot).
5. Install the self-heal agent: `~/.config/sing-box/heal.sh` (curl 7891 vs 7890
   with `--noproxy "*"`, log to `heal.log`, restart sing-box if they differ while
   7891 is alive) and `~/Library/LaunchAgents/local.singbox-heal.plist`
   with `StartInterval` 600; `launchctl bootstrap gui/$(id -u) <plist>`.
6. Add the `env` block to `settings.json` (see Gotchas — it's a symlink).
7. **Add `vpncheck` to `~/.zshrc`** — `MODE=diagnose` assumes it exists:
   ```bash
   vpncheck() {
     local direct tunnel proxied code
     if lsof -nP -iTCP:7890 -sTCP:LISTEN >/dev/null 2>&1; then echo "sing-box  : UP"
     else echo "sing-box  : DOWN   -> brew services start sing-box"; return 1; fi
     direct=$(curl -s -m 8 --noproxy "*" https://api.ipify.org)
     tunnel=$(curl -s -m 12 -x http://127.0.0.1:7891 https://api.ipify.org)
     proxied=$(curl -s -m 12 -x http://127.0.0.1:7890 https://api.ipify.org)
     code=$(curl -s -m 15 -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}' https://api.anthropic.com/v1/messages)
     echo "direct IP : ${direct:-?}"
     if [ -n "$tunnel" ]; then echo "tunnel    : ALIVE (exit $tunnel)"
     else echo "tunnel    : DEAD   -> Surfshark WireGuard not handshaking: skill manage-claude-split-tunnel (rotate-config)"; fi
     if [ -z "$proxied" ]; then echo "claude via: NOTHING (proxy broken)"
     elif [ "$proxied" = "$tunnel" ]; then echo "claude via: TUNNEL"
     else echo "claude via: DIRECT (fallback)$([ -n "$tunnel" ] && echo '  -> tunnel is back: brew services restart sing-box')"; fi
     echo "claude    : $code   (405 = OK, 403 = region-blocked, 000 = dead)"
   }
   ```
8. **Fully quit and reopen Claude Code** — in Warp, Terminal, or wherever it runs.
   `settings.json` → `env` is read at startup, so a session already running keeps
   the old environment and keeps failing with 403. `/exit` then relaunch.
9. Run the definitive test from `MODE=diagnose` step 4.

## Steps — MODE=teardown (only if the tunnel is genuinely no longer wanted)

"Not travelling" is not a reason: the proxy falls back to direct by itself. If it
really must go, do **both**, in this order:

1. Remove the `env` block with a **targeted edit on the real path**
   `~/.claude-config/settings.json` (Edit tool, or `jq` writing back to that same
   path). Never `jq … > tmp && mv tmp ~/.claude/settings.json` — that replaces
   the symlink.
2. `brew services stop sing-box`; `launchctl bootout gui/$(id -u)/local.singbox-heal`

Doing only one half is exactly the 2026-09-03 incident.

## Gotchas that cost real time

- **Inside Claude Code, every shell command already goes through the proxy.**
  Claude Code injects `settings.json` → `env` into the Bash tool, so a plain
  `curl https://api.ipify.org` from Claude shows the *tunnel* exit, not the
  network's. This produced a false "the router is on the VPN" diagnosis on
  2026-09-03. For a truly direct probe use `curl --noproxy "*" …`; `vpncheck`
  and `heal.sh` already do.
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
- **One WireGuard key pair per client.** The same key on a router and on
  sing-box makes the server flip between them and both drop packets.
- **Providers to avoid for sing-box:** PureVPN (WireGuard expires), CyberGhost
  (WireGuard only inside its app; manual = OpenVPN). Surfshark and Mullvad both
  issue permanent manual WireGuard configs.
- **`~/.claude/settings.json` is a symlink** to `~/.claude-config/settings.json`.
  Edits must target the real path or they're refused.
- **A PreToolUse hook blocks whole-file `Write` to settings.json** (it once wiped
  the config). Use the `Edit` tool for targeted changes.
- **`~/.claude-config` is a git repo that auto-pulls (`--ff-only`) at SessionStart.**
  The committed proxy `env` reaches every machine — each one needs sing-box on
  7890 (with the fallback design, sing-box alone is enough to stay safe).
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
applied, and the definitive-test result. Never print the WireGuard private key —
mask it and say which file it went into.

## Note

Written for Surfshark Singapore + macOS/Homebrew, but nothing here is provider-specific:
any WireGuard config that does not expire works (Mullvad, Proton, AirVPN, IVPN, your own
VPS…), and `sing-box` runs on Linux too. Replace the `<placeholders>` with your own
values. The `~/.claude-config` symlink gotchas apply only if you keep `settings.json`
in a synced repo like the author does.
