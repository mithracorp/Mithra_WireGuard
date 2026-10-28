# Mithra_WireGuard

WireGuard VPS installer with an HTTPS management API (Node.js behind Caddy), a
built-in firewall, per-key limits, health tooling, and a restricted SSH account
(`wg-monitor`) that lets a central management server pull traffic stats and
enforce peer limits.

Everything is installed by one script: `install-wg-full.sh`.

> **Moved from `arkh91/Arkh91_WireGuard`.** The old `install-wg3.sh`
> (253-key pool, no firewall layer) is superseded. Existing nodes keep working;
> see [Updating an existing node](#updating-an-existing-node).

---

## Supported platforms

| Family | Package manager | Distributions |
|---|---|---|
| Debian | `apt-get` | Ubuntu 20.04+, Debian 11+ and derivatives |
| RHEL | `dnf` | AlmaLinux, Rocky, CentOS Stream, RHEL 9+, Fedora (EPEL / Caddy COPR enabled when needed) |
| Arch | `pacman` | Arch Linux and derivatives (uses `pacman -S`, never `-Syu`) |

Requirements: root, systemd. EL8 is refused (its kernel has no WireGuard
module). FreeBSD and Windows are detected and refused with exit code 3.

---

## Quick start

```bash
# Interactive menu
sudo bash <(curl -Ls https://raw.githubusercontent.com/mithracorp/Mithra_WireGuard/refs/heads/main/install-wg-full.sh)

# Non-interactive
sudo bash <(curl -Ls https://raw.githubusercontent.com/mithracorp/Mithra_WireGuard/refs/heads/main/install-wg-full.sh) \
    --domain=vpn-api.example.com --port=51820 --https-port=8443 --yes
```

Anything after the closing `)` is passed to the script as its own arguments.
Putting a flag *before* it (`curl -Ls <url> --add-monitor`) does not work:
`curl` treats it as its own option.

**Safer than piping from a branch:** download a tagged release, verify its
SHA-256 against the release notes, then run it.

```bash
curl -fsSLo install-wg-full.sh https://raw.githubusercontent.com/mithracorp/Mithra_WireGuard/<tag>/install-wg-full.sh
sha256sum install-wg-full.sh
sudo bash install-wg-full.sh
```

To read or change the script locally, open it with `vi install-wg-full.sh`.

### What the interactive install asks

1. WireGuard UDP port (default `51820`)
2. Domain or public IP clients use to reach the API
3. HTTPS port for the API. The installer scans `443 8443 2053 2083 2087 2096`
   (the ports Cloudflare proxies) and offers the free ones
4. TLS mode: self-signed (default, pinned by SHA-256) or Let's Encrypt (port 443 only)

The `wg-monitor` SSH account is **opt-in**: pass `--monitor-key` /
`--monitor-key-file`, or run `--add-monitor` later.

---

## What gets installed

| Component | Purpose |
|---|---|
| WireGuard `wg0` | Default pool `10.66.0.0/16` (~65,000 keys). `--network253` selects `10.66.66.0/24` (253 keys). NAT in `PostUp`/`PostDown`, `ip_forward` persisted in `/etc/sysctl.d` |
| `wg-api` (Node.js, localhost only) | Management API, behind Caddy on the HTTPS port you chose. Nothing listens on port 80 |
| `/usr/local/bin/wg-firewall.sh` | wg0's filter rules, run by wg-quick. Clients cannot reach each other, private/metadata networks or the server's own services. Paused, over-limit and suspended keys are dropped first |
| `/usr/local/bin/wg-peer-ctrl.sh` | Restricted dispatcher used by `wg-monitor` (suspend, speed limits, ...) |
| `/usr/local/bin/wg-api-helper` | Root helper the API calls through `sudo` (the only thing the `wgapi` user may run as root) |
| `/usr/local/bin/wg-healthcheck.sh` | Health check |
| `/usr/local/bin/vpn-doctor.sh` | Diagnose and fix |
| `wg-healthcheck.timer` | Daily health check, report in `/var/log/wg-healthcheck.log` |
| `wg-traffic.timer` | Saves traffic counters every minute and before wg0 stops, so usage survives restarts and reboots |
| `wg-monitor` SSH account | Optional; see [below](#wireguard-monitoring-account-wg-monitor) |

---

## Command-line reference

```bash
sudo ./install-wg-full.sh                         # interactive menu
sudo ./install-wg-full.sh --no-api                # WireGuard + tools only (no Caddy/Node)
sudo ./install-wg-full.sh --reconfigure-api       # change API HTTPS port/certificate; WireGuard keys untouched
sudo ./install-wg-full.sh --add-monitor --monitor-key-file=/root/wg_monitor_key.pub
sudo ./install-wg-full.sh --update-tools          # re-fetch healthcheck/doctor scripts
sudo ./install-wg-full.sh --update-api            # rewrite wg-api-helper + server.js (and firewall); keys/peers untouched
sudo ./install-wg-full.sh --update-firewall       # install wg-firewall.sh on an older node, live (no restart)
sudo ./install-wg-full.sh --uninstall
```

| Option | Meaning |
|---|---|
| `--domain=NAME` | Domain (or IP) clients use to reach the API |
| `--port=N` | WireGuard UDP port (default 51820) |
| `--https-port=N` | Public HTTPS port of the API |
| `--api-port=N` | Internal API port (default 4242) |
| `--tls=selfsigned\|letsencrypt` | `selfsigned` is the default and needs no port 80/443; `letsencrypt` works only with `--https-port=443` |
| `--tls-cert=FILE --tls-key=FILE` | Use your own certificate (copied; does not follow renewals) |
| `--network253` | Use `10.66.66.0/24` instead of `10.66.0.0/16` |
| `--dns=IP[,IP]` | DNS handed to clients (default 1.1.1.1) |
| `--monitor-key="ssh-..."` / `--monitor-key-file=FILE` | Public key of the management server |
| `--no-api` | Skip Caddy, Node and the API |
| `--no-caddy` | Install the API without Caddy (use when nginx already owns 80/443; a snippet is printed) |
| `--allow-client-to-client` | Let VPN clients reach each other (default: blocked) |
| `--allow-private-nets` | Let clients reach private, link-local (cloud metadata) and CGNAT addresses (default: blocked) |
| `--force` | Replace an existing `wg0.conf` (backs it up first) |
| `--yes`, `-y` | Do not ask questions |

---

## Management API

Authentication is either `Authorization: Bearer <token>` or an Outline-style
secret path (`https://host:port/<secret>/...`, no header needed). Both are saved
at the end of the install:

```text
/etc/wg-api/access.json           {"apiUrl":"https://host:port","apiToken":"...","certSha256":"..."}
/etc/wg-api/access-outline.json   {"apiUrl":"https://host:port/<secret>","certSha256":"..."}
```

Either line can be pasted into Mithra WireGuard Manager. **Treat both like
passwords.** `GET /health` is public.

| Route | Effect |
|---|---|
| `POST /create` | New key; client config as text (`?format=json` for JSON). Accepts a name, optional fixed address, traffic limit and expiry |
| `POST /remove` `{"ipAddress":"10.66.0.5"}` | Remove a key |
| `GET /keys`, `GET /keys/<ip>` | Every key with full client settings and usage |
| `GET /usage`, `GET /usage/<ip>` | Light traffic counters, suited to polling (one call per minute is fine) |
| `GET /server`, `GET /metrics/transfer` | Server info / bytes per key |
| `GET/POST /access-keys`, `DELETE /access-keys/<ip>`, `PUT /access-keys/<ip>/name` | Outline-style equivalents |
| `POST /keys/<ip>/pause`, `/resume` | Block / unblock a key without deleting it |
| `PUT /keys/<ip>/limit` `{"traffic":"20G"}` or `"unlimited"` | Traffic cap per key (upload + download) |
| `PUT /keys/<ip>/expiry` | Time limit per key, see below |

Expiry body fields (one per request):

```json
{"expires": "30d"}                              // from now (h, d, w, m = calendar months, y)
{"expiresOn": "2026-12-31 18:00"}               // on the server's clock; a zone suffix (Z, +02:00) is honoured
{"expiresAfterFirstConnection": "1m"}           // clock starts at the key's first connection
```

`"never"` removes the time limit. A key past its end is blocked like a paused
one until it gets a later end. Limits are enforced every minute by the traffic
saver.

Example:

```bash
# Create a key (self-signed cert, so pin it with the fingerprint printed at install)
curl --insecure --pinnedpubkey 'sha256//...' -X POST https://vpn-api.example.com:8443/create \
     -H "Authorization: Bearer $TOKEN"
```

Counters come from the kernel and are saved to `/var/lib/wg-api/traffic.db`, so
usage survives wg0 restarts and reboots. Rotate the token by editing
`/etc/wg-api/env` with `vi` and running `systemctl restart wg-api`.

---

## Firewall options

`wg-firewall.sh` reads `/etc/wireguard/wg-firewall.conf` (1 = on). Edit it with
`vi /etc/wireguard/wg-firewall.conf`, then apply with `wg-firewall.sh up`.

```text
CLIENT_ISOLATION=1       # VPN clients cannot reach each other
BLOCK_PRIVATE_NETS=1     # nor private, link-local (cloud metadata) or CGNAT addresses
BLOCK_SERVER_ACCESS=1    # nor the server's own services (ping and DNS still work)
```

Suspension state lives in files (`/etc/wg-api/policy.db`,
`/etc/wireguard/suspended.list`), so it survives restarts and reboots without
`netfilter-persistent`. Inspect the live rules with `wg-firewall.sh status`.

If you use `ufw` or `firewalld`, the installer opens the WireGuard UDP port and
the API HTTPS port for you. On SELinux-enforcing EL hosts it labels a
non-standard HTTPS port so Caddy can bind.

---

# WireGuard Monitoring Account (`wg-monitor`)

This is what lets the management server pull `wg show wg0 dump` output and send
enforcement commands (disable/enable/suspend/unsuspend/speed-limit) over SSH,
without ever giving it a real login on the VPN node.

All of the setup below is **automated** by `install-wg-full.sh` (during install
with `--monitor-key`, or later with `--add-monitor`). This section documents
what it does and why, so you can verify it or do it by hand.

## 1. Generate the key (once, on the management server)

```bash
ssh-keygen -t ed25519 -f /root/.ssh/wg_monitor_key -N ""
```

One keypair is reused across every VPN node. Only the **public** half
(`wg_monitor_key.pub`) ever leaves the management server.

## 2. What the installer does on each VPN node

```bash
useradd -r -m -d /home/wg-monitor -s /bin/bash wg-monitor
```

**The shell must be `/bin/bash`, not `nologin`.** `sshd` runs `ForceCommand`
*through* the account's login shell (`<shell> -c "<forced command>"`).
`nologin` ignores the command and prints `This account is currently not
available.`, which silently breaks `ForceCommand`. The account is still locked
down: `PermitTTY no` gives no usable terminal and `ForceCommand` always
overrides what the client sends.

```bash
# /etc/sudoers.d/wg-monitor  (validated with visudo before install)
Defaults!/usr/local/bin/wg-peer-ctrl.sh env_keep += "SSH_ORIGINAL_COMMAND"
wg-monitor ALL=(root) NOPASSWD: /usr/local/bin/wg-peer-ctrl.sh
```

The `env_keep` line matters: `sudo` strips `SSH_ORIGINAL_COMMAND` by default,
and without it every call would fall through to the no-argument case and just
return the traffic dump.

```text
# appended to /etc/ssh/sshd_config, marked "# wg-monitor (install-wg-full.sh)"

Match User wg-monitor
    PasswordAuthentication no
    PermitTTY no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
    ForceCommand sudo /usr/local/bin/wg-peer-ctrl.sh
```

`sshd -t` must pass before `sshd` is reloaded; on failure the original
`sshd_config` is restored. If your `sshd_config` has an `AllowUsers` line, add
`wg-monitor` to it (`vi /etc/ssh/sshd_config`) or SSH will refuse the account.

## 3. `wg-peer-ctrl.sh`, the dispatcher

This is the *only* thing `wg-monitor` can execute. It reads
`SSH_ORIGINAL_COMMAND` (or plain `argv` for local calls) and validates every
argument before touching anything.

| Command | Effect |
|---|---|
| *(empty)* | `wg show wg0 dump`, for traffic sync |
| `peer-disable <pubkey>` | Remove peer from WireGuard entirely (expired) |
| `peer-enable <pubkey> <cidr>` | Re-add peer with its allowed-ips |
| `peer-suspend <cidr>` | Block the peer via the `WG-SUSPEND` chain (peer stays in wg, handshakes still work). Recorded in `/etc/wireguard/suspended.list` |
| `peer-unsuspend <cidr>` | Remove the block |
| `peer-setlimit <cidr> <kbps>` | Cap bandwidth both directions via `tc` (htb, plus an `ifb0` device for upload) |
| `peer-clearlimit <cidr>` | Remove the cap |

Every call is logged and can be followed with:

```bash
journalctl -t wg-peer-ctrl -f
```

Anything outside this list, including `rm -rf /` sent over the same channel, is
rejected with `ERROR: command not permitted` and logged as `DENY`.

**Speed-limit note:** the `tc` class ID is the hex value of
`(third octet * 256 + fourth octet)` of the peer's address, unique inside any
single pool up to a `/16`. If a node ever hands out addresses from more than
one pool, this needs a real class-ID allocator.

---

## Health tools

```bash
sudo wg-healthcheck.sh [--capture=20]        # health check
sudo vpn-doctor.sh [--fix] [--dry-run]       # diagnose and fix
cat /var/log/wg-healthcheck.log              # daily timer report
```

Both are installed to `/usr/local/bin` by the installer. They are copied from
next to the script when present, otherwise downloaded and checked against a
pinned SHA-256 before install (they run as root). Refresh them with
`--update-tools`.

---

## Testing

Work from safest to riskiest.

### 1. Sanity-check the account, on the VPN node

```bash
id wg-monitor                                     # exists, /bin/bash, no error
grep -A6 "Match User wg-monitor" /etc/ssh/sshd_config
cat /etc/sudoers.d/wg-monitor
visudo -c
```

Ask `sshd` what it will actually apply (more reliable than re-reading the file):

```bash
sshd -T -C user=wg-monitor,host=<MGMT_HOSTNAME>,addr=<MGMT_IP> \
    | grep -iE "forcecommand|permittty|passwordauthentication"
```

Expect:

```text
passwordauthentication no
permittty no
forcecommand sudo /usr/local/bin/wg-peer-ctrl.sh
```

### 2. Read-only test, from the management server

```bash
ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP
```

Expect the `wg show wg0 dump` output and no shell prompt. `PTY allocation
request failed on channel 0` is expected noise from `PermitTTY no`. If you get
`This account is currently not available.`, run
`usermod -s /bin/bash wg-monitor` on that node and retest.

### 3. Write verbs, against a throwaway peer

```bash
ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP "peer-suspend 10.66.0.X/32"
# on the node: wg-firewall.sh status | grep WG-SUSPEND
ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP "peer-unsuspend 10.66.0.X/32"

ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP "peer-setlimit 10.66.0.X/32 2000"
# on the node: tc class show dev wg0 ; tc class show dev ifb0
ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP "peer-clearlimit 10.66.0.X/32"
```

Each should print `OK: ...`, and `journalctl -t wg-peer-ctrl -n 20` should show a
matching `result=OK` line per command.

### 4. Confirm rejection works

```bash
ssh -i /root/.ssh/wg_monitor_key wg-monitor@VPN_SERVER_IP "rm -rf /"
```

Expect `ERROR: command not permitted` and a `result=DENY` line in the journal.
If this ever returns anything else, stop and fix the sudoers/`ForceCommand`
setup before relying on it.

### 5. Check the API

```bash
curl -sk https://DOMAIN:HTTPS_PORT/health        # {"status":"ok",...}
systemctl status wg-api caddy --no-pager
journalctl -u wg-api -n 30
```

---

## Updating an existing node

| Node was installed with | Do this |
|---|---|
| Older `install-wg-full.sh` | `--update-api` (new routes, firewall, limits/expiry; keys and peers kept) |
| Only the firewall rules are old | `--update-firewall` (applied live, no restart) |
| Old `install-wg3.sh` | Run `--add-monitor` for the account, then `--update-firewall`. A full reinstall of the API needs a fresh `--domain` run; do not use `--force` unless you accept replacing the server keys |

Known issues in nodes from the original `install-wg3.sh`, both fixed by
`--add-monitor`:

1. **No `wg-monitor` account**: SSH from the management server falls back to a password prompt.
2. **`wg-monitor` shell set to `nologin`**: every session fails with `This account is currently not available.` Fix directly with `usermod -s /bin/bash wg-monitor`.

---

## Uninstall

```bash
sudo ./install-wg-full.sh --uninstall
```

Removes the interface, API, helpers, timers, accounts, sudoers entries, the
sshd `Match` block and our Caddyfile. WireGuard keys and config are first saved
to `/root/wg-uninstall-backup-<timestamp>`. Package removal is asked
separately.

---

## Security notes

* Keep `wg_monitor_key` (the private half) on the management server only, mode `600`.
* `wg-monitor` can never get an interactive shell: no PTY, no forwarding, and `ForceCommand` always wins.
* Every accepted and rejected command is logged (`journalctl -t wg-peer-ctrl`).
* The API runs as the unprivileged `wgapi` user; its code is root-owned and read-only to it, and it can run only `wg-api-helper` as root.
* The Caddy log drops request paths and the `Authorization` header, so the secret API path and token never reach disk.
* The bearer token and secret path in `/etc/wg-api/` are separate from the SSH account and should be rotated if ever pasted somewhere they could leak.
* Client keys are stored in `/etc/wg-api/clients.db` (root only) so `GET /keys` can show a config again.
