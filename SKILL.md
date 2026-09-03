---
name: manage-claude-split-tunnel
description: Diagnose and repair Claude Code's connection when it fails from a Hong Kong / China network — "API Error 403 Request not allowed", "Please run /login", "ECONNREFUSED 127.0.0.1:7890", works in Terminal but not Warp, or Claude works but gitee/mainland sites don't. Also sets up or rebuilds the sing-box split tunnel that sends ONLY Claude through PureVPN Singapore (falling back to a direct connection by itself when the tunnel is dead) while everything else goes direct, and rotates the WireGuard config when it expires.
---

# Manage Claude split tunnel

Your network reaches mainland China but **Anthropic region-blocks your IP**.
The reverse is also true of a full VPN. This skill owns that conflict.

**The single most important fact: `403 Request not allowed` + "Please run /login"
is NOT an auth problem.** The login is fine. Traffic just isn't leaving Singapore.
Never run `/login` for it — it cannot help and wastes a re-auth.

## Parameters

- `MODE`: `diagnose` (default) | `rotate-config` | `rebuild`
- `PROXY_PORT`: `7890` (local mixed HTTP+SOCKS inbound)
- `WG_SOURCE`: fresh WireGuard config from https://my.purevpn.com/v2/dashboard/manual-config

## Architecture (why it's built this way)

```
Claude Code ──HTTPS_PROXY──> sing-box 127.0.0.1:7890 ─┬─ wg-sg  (WireGuard → PureVPN SG)  preferred
                                                      └─ direct  fallback, automatic when the tunnel is dead
everything else ──────────────────────────────────────────────────> direct (China OK)
127.0.0.1:7891 = tunnel-only test port (used by vpncheck; never point Claude at it)
```

- **The proxy is fail-safe (since 2026-09-03).** `route.final` is a `urltest`
  group `[wg-sg, direct]` with a 30 s tolerance: it uses the tunnel while the
  tunnel passes a health check every minute, and drops to `direct` on its own the
  moment it fails. A dead or expired tunnel therefore no longer strands Claude on
  networks that are not region-blocked (phone hotspot, a router-level VPN).
  **Leave the `env` block and sing-box on permanently — that is the design.** The
  only thing that still strands Claude is sing-box itself being stopped, which
  shows up as `ECONNREFUSED 127.0.0.1:7890`.
- **Fallback is sticky.** Once the group has dropped to `direct` it stays there
  until sing-box restarts (rotate-config restarts it anyway). `vpncheck` showing
  `tunnel: ALIVE` + `claude via: DIRECT` → `brew services restart sing-box`.
- **`$PREFIX` = `brew --prefix`, resolve it before using any path on a new
  machine.** Apple Silicon → `/opt/homebrew`; Intel → `/usr/local`. All paths below
  are written as `$PREFIX/...`. Getting this wrong is the most likely first-run
  failure on an Intel Mac (confirmed on a MacBookPro13,3, which is `/usr/local`).
- **sing-box runs userspace WireGuard.** It changes *no* system routing, so it
  physically cannot break your connectivity. Safe to restart at any time.
- **The proxy is set in `settings.json` → `env`, not the shell.** This is the whole
  point: a `.zshrc` function only works in Terminal. Warp launches Claude without
  it and gets the 403. Claude Code reads `settings.json` itself, so the env layer
  works from Warp, Terminal, an alias, `caffeinate`, or Spotlight alike.
- **PureVPN SG also reaches mainland China** (verified: gitee.com, docs.openluat.com,
  example.cn all 200). So routing everything through it is a valid fallback.

## Steps — MODE=diagnose

1. Run `vpncheck` (defined in `~/.zshrc`). Healthy output:
   ```
   sing-box  : UP
   direct IP : <your-direct-ip>               (whatever the Wi-Fi gives)
   tunnel    : ALIVE (exit <your-vpn-exit-ip>)  (PureVPN SG, Datacamp Limited)
   claude via: TUNNEL
   claude    : 405   (405 = OK, 403 = region-blocked, 000 = dead)
   ```
2. Match the symptom:

| Symptom | Cause | Fix |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:7890`, `OAuth error: connect ECONNREFUSED`, `sing-box: DOWN` | sing-box service stopped (it never stops by itself; someone ran `brew services stop`) | `brew services start sing-box`, then fully restart Claude. **Do not delete the `env` block** — that was the 2026-09-03 mistake |
| `tunnel: DEAD`, `claude via: DIRECT` | **WireGuard config expired** — the usual culprit. Claude still works on non-blocked networks, 403 on your blocked home Wi-Fi | `MODE=rotate-config` |
| `tunnel: ALIVE`, `claude via: DIRECT` | Group stuck on fallback after a blip (sticky by design) | `brew services restart sing-box` |
| `claude: 403` with `claude via: TUNNEL` | PureVPN exit itself refused (rare) | Download a different Singapore config |
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

## Getting a fresh config from PureVPN

**Source:** https://my.purevpn.com/v2/dashboard/manual-config

1. You must be logged in. The page offers **WireGuard configs per city** — no
   SOCKS5 endpoint is listed there, so WireGuard is the path.
2. Type `Singapore` into the **"Search by Location"** box (`input[name="search"]`),
   then hit **Download** on the Singapore row.
3. It lands in `~/Downloads/`, typically `Singapore-wg.conf` — **or
   `Singapore-wg (1).conf`** on a repeat download. That space and the parentheses
   must be quoted in every shell command:
   `"~/Downloads/Singapore-wg (1).conf"`
4. The page shows an **"Expires in: NN min"** countdown with *"Want to extend
   duration?"*. Use **Extend WireGuard Configuration Time** to get a longer-lived
   config, or this rotation comes round again quickly.

**Ask for the file path, never for the contents.** Read the values from the
file directly. Pasting a config into chat writes a live private key into the
transcript (it happened once — the key then needs regenerating).

Expected shape — **values below are placeholders, not real credentials**:

```ini
[Interface]
PrivateKey=<44-char base64, SECRET — never echo, never commit>
Address=172.21.233.3
DNS=<provider-dns>
[Peer]
PublicKey=<44-char base64, server key, not secret>
AllowedIPs=0.0.0.0/0
Endpoint=<endpoint-host>:51820
PersistentKeepalive=21
```

Field mapping into `$PREFIX/etc/sing-box/config.json` (the script below does
this automatically — the table is for when it needs checking by hand):

| WireGuard `.conf` | sing-box |
|---|---|
| `PrivateKey` | `endpoints[0].private_key` |
| `Address` | `endpoints[0].address` — **append `/32`** |
| `PublicKey` | `endpoints[0].peers[0].public_key` |
| `Endpoint` host / port | `peers[0].address` / `peers[0].port` (split on the last `:`) |
| `AllowedIPs` | `peers[0].allowed_ips` |
| `PersistentKeepalive` | `peers[0].persistent_keepalive_interval` |
| `DNS` | unused — sing-box resolves via the system |

Sanity check before using a new file: `Endpoint` should resolve into PureVPN's
Singapore range (`<your provider's range>`) — `dig +short <endpoint-host>`.

## Steps — MODE=rotate-config

PureVPN WireGuard configs **expire** (see the countdown above). This is the most
likely future failure.

1. Download a **Singapore** WireGuard config from `WG_SOURCE` (above) —
   **after clicking *Extend WireGuard Configuration Time***. The default
   15-minute config installed on 2026-09-03 was dead the same afternoon.
2. Patch the live config in place — never retype secrets, never echo the key:
   ```bash
   python3 - <<'PY'
   import json
   src = "~/Downloads/Singapore-wg.conf"   # <-- new file
   kv = {}
   for line in open(src):
       line = line.strip()
       if "=" in line and not line.startswith("["):
           k, v = line.split("=", 1); kv[k.strip().lower()] = v.strip()
   host, port = kv["endpoint"].rsplit(":", 1)
   dst = "$PREFIX/etc/sing-box/config.json"   # <-- resolve $PREFIX (brew --prefix) first
   cfg = json.load(open(dst)); ep = cfg["endpoints"][0]
   ep["address"] = [kv["address"] + "/32"]
   ep["private_key"] = kv["privatekey"]
   ep["peers"][0].update({"address": host, "port": int(port),
                          "public_key": kv["publickey"]})
   json.dump(cfg, open(dst, "w"), indent=2)
   print("updated", dst)
   PY
   brew services restart sing-box && sleep 5 && vpncheck
   ```
3. Mirror it to the backup at `~/.config/sing-box/config.json`, `chmod 600` both.
4. `vpncheck` must now show `tunnel    : ALIVE` **and** `claude via: TUNNEL`.

## Steps — MODE=rebuild (new machine, or from scratch)

0. `brew --prefix` → set `$PREFIX` for every path below. Don't assume
   `/opt/homebrew` — Intel Macs are `/usr/local`.
1. `brew install sing-box`
2. Write `$PREFIX/etc/sing-box/config.json` (`chmod 600`; a pristine copy
   lives at `~/.config/sing-box/config.json`). Shape — the key material comes
   from the `.conf` via the rotate-config script:
   - `inbounds`: `mixed` on `127.0.0.1:7890` tag `local-proxy`, and `mixed` on
     `127.0.0.1:7891` tag `wg-only`
   - `endpoints`: one `wireguard` endpoint tag `wg-sg` (sing-box ≥1.11 uses
     `endpoints`, not `outbounds`), `"system": false`, `"mtu": 1408`
   - `outbounds`: `{"type":"urltest","tag":"auto","outbounds":["wg-sg","direct"],
     "url":"https://www.gstatic.com/generate_204","interval":"1m","tolerance":30000}`
     and `{"type":"direct","tag":"direct"}`
   - `route`: `{"rules":[{"inbound":["wg-only"],"outbound":"wg-sg"}],"final":"auto"}`
3. `sing-box check -c <file>` — **always validate before starting.**
4. `brew services start sing-box` (installs a LaunchAgent → survives reboot).
5. Add the `env` block to `settings.json` (see Gotchas — it's a symlink).
6. **Add `vpncheck` to `~/.zshrc` if it isn't there** — a fresh machine has
   nothing, and `MODE=diagnose` assumes it. Append:
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
     else echo "tunnel    : DEAD   -> PureVPN config expired: get a new one (skill: rotate-config)"; fi
     if [ -z "$proxied" ]; then echo "claude via: NOTHING (proxy broken)"
     elif [ "$proxied" = "$tunnel" ]; then echo "claude via: TUNNEL"
     else echo "claude via: DIRECT (fallback)$([ -n "$tunnel" ] && echo '  -> tunnel is back: brew services restart sing-box')"; fi
     echo "claude    : $code   (405 = OK, 403 = region-blocked, 000 = dead)"
   }
   ```
7. **Fully quit and reopen Claude Code** — in Warp, Terminal, or wherever it runs.
   `settings.json` → `env` is read at startup, so a session already running keeps
   the old environment and keeps failing with 403. `/exit` then relaunch.
8. Run the definitive test from `MODE=diagnose` step 4.

## Steps — MODE=teardown (only if the tunnel is genuinely no longer wanted)

"Not travelling" is not a reason: the proxy falls back to direct by itself. If it
really must go, do **both** halves, in this order:

1. Remove the `env` block with a **targeted edit on the real path**
   `~/.claude-config/settings.json` (Edit tool, or `jq` writing back to that same
   path). Never `jq … > tmp && mv tmp ~/.claude/settings.json` — that replaces
   the symlink.
2. `brew services stop sing-box`

Doing only one half is exactly the 2026-09-03 incident.

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
  your blocked home Wi-Fi. Fix the tunnel (rotate-config) — or do nothing: the proxy falls back
  on its own.
- **`ECONNREFUSED 127.0.0.1:7890` means sing-box is stopped, nothing else.** It
  never stops by itself (launchd keeps it alive); someone ran `brew services stop`.
  `brew services start sing-box` and move on.
- **`~/.claude/settings.json` is a symlink** to `~/.claude-config/settings.json`.
  Edits must target the real path or they're refused.
- **A PreToolUse hook blocks whole-file `Write` to settings.json** (it once wiped
  the config). Use the `Edit` tool for targeted changes.
- **`~/.claude-config` is a git repo that auto-pulls (`--ff-only`) at SessionStart.**
  Committing the proxy `env` propagates it to every machine — fine only if each
  runs sing-box on 7890. Leaving it uncommitted keeps it local, but a future
  upstream change to `settings.json` will make the pull fail *silently*.
- **Never switch the network or stop the tunnel mid-session** — it kills the live
  Claude connection. Set the replacement up first, then switch.
- **A shell function is the wrong layer.** It shadows `claude`, and Warp never
  sees it. `settings.json` → `env` is the correct fix; the `.zshrc` `claude()`
  wrapper that remains is redundant belt-and-braces.
- **`env` is read only at startup.** After editing `settings.json`, a running
  Claude Code session keeps the old environment and keeps returning 403. Quit and
  relaunch before concluding the fix didn't work — this exact confusion cost a
  round: the first relaunch was made before the setting existed, so it still 403'd,
  and the fix looked broken when it wasn't.
- `curl https://api.anthropic.com/` (root) returns 403/404 normally — that is
  **not** a useful test. Always probe `/v1/messages`: `405` means healthy.

## Output format

Report: `vpncheck` output verbatim, the symptom matched from the table, the fix
applied, and the definitive-test result. Never print the WireGuard private key —
mask it and say which file it went into.

## Note

Written for PureVPN Singapore + macOS/Homebrew, but nothing here is provider-specific:
any WireGuard config works, and `sing-box` runs on Linux too. Replace the
`<placeholders>` with your own values. The `~/.claude-config` symlink gotchas apply
only if you keep `settings.json` in a synced repo like the author does.
