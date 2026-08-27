---
name: namefi-dyndns-without-tooling
description: >-
  Use when a user wants to expose a locally-developed service on a real hostname
  without a static IP and without installing a dynamic DNS client — update a
  Namefi A/AAAA record with a plain curl one-liner driven by
  cron/systemd-timer/launchd, or have the agent itself push updates via the
  Namefi MCP tools / /v-next/dns REST API. Covers both a VPS (single public IP)
  and a homelab behind NAT, plus the Dyn return codes.
---
# Dynamic DNS on Namefi with No Client Software

Goal: a hostname you own on Namefi resolves to this machine's **current** public
address, using only what is already on the box (`curl`, `cron`/`systemd`/
`launchd`) — or, for a throwaway demo, the agent's own API access.

Ground truth: `apps/backend/src/lib/ddns/README.md`,
`apps/backend/src/routers/ddns.ts`. TLS and hostname routing are separate — hand
off to `namefi-https-and-routing`.

## Plan first — `simple` (default) and `advanced`

This skill takes the family's mode argument (`simple`, the default, or
`advanced`) and keeps the mode of the hand-off that brought you here. In
**simple** mode: run the detection this skill already prescribes, fill every
choice from the defaults below, and present **one numbered plan** — each line a
choice plus the detected fact behind it — with a single *"accept, or name a
line to change"* prompt. In **advanced** mode, stop at each decision and let
the user pick, the default marked *(recommended)*. Never ask what `command -v`
or a probe already answered. The family-wide defaults table lives in
[`../namefi-dyndns/SKILL.md`](../namefi-dyndns/SKILL.md).

Defaults this skill owns:

- **Approach** — §3 (`curl` + a timer) unless the user has *called* this a
  throwaway demo, in which case §2 (agent-driven) — that's a fact from their
  words, not a question to ask.
- **Scheduler** — systemd timer on Linux, launchd on macOS, plain cron as the
  fallback — pick by the OS you are on.
- **Credential** — the scoped dyndns secret is the one user-supplied item;
  surface it as a blocker above the plan.
- **Ports / HTTPS** — same probe-driven rules as the family table: router
  grants 80+443 → forward both and put the `namefi-https-and-routing` hand-off
  in the plan without asking; refuses them → a granted high port, plain HTTP,
  and the plan says why HTTPS is out.

## 0. Choose the approach

| Situation | Use |
|---|---|
| The `namefi` CLI is available or installable | **`namefi-dyndns-with-cli`** — easiest, and it opens NAT mappings itself |
| `ddclient` is available or acceptable to install | `namefi-dyndns-with-ddclient` |
| Nothing may be installed, service must survive the session | **§3 — `curl` + cron/timer** |
| Short-lived demo, agent stays running, no setup at all | **§2 — agent-driven updates** |

**Choose §2 only when the work is genuinely ephemeral.** It stops the moment the
agent session ends, leaving a record pointing at a stale address. Anything meant
to stay up gets §3.

Topology matters only for reachability, not for the DNS update:

- **(a) VPS** — one public IP, no NAT. Nothing else needed.
- **(b) Homelab** — the record points at the *router*; inbound ports must be
  forwarded to this machine (§5). `dig` looking perfect proves nothing here.

```bash
ip -4 addr show 2>/dev/null || ifconfig    # private 10./172.16-31./192.168. => NAT
curl -s https://api.ipify.org; echo        # what the world sees
```

Public v4 in `100.64.0.0/10` means **CGNAT** — inbound IPv4 is impossible (§6).

## 1. Get a Dynamic DNS credential

A DDNS credential is permission for one user to update **one hostname's** A/AAAA
records *while that user still owns the domain*. Ownership is re-checked on every
request. Credentials never expire and the secret is shown **once**.

- **MCP** — `https://api.namefi.io/mcp` (streamable-http) exposes the whole API
  as tools; call the one matching `createDdnsKey`.
- **REST** — base `https://api.namefi.io/v-next/`, auth `X-API-Key: <key>` or an
  OAuth bearer:

  | Method | Path | operationId |
  |---|---|---|
  | POST | `/dns/ddns/keys` | `createDdnsKey` |
  | GET | `/dns/ddns/keys` | `listDdnsKeys` |
  | POST | `/dns/ddns/keys/revoke` | `revokeDdnsKey` (body `{credentialId}`) |
  | PUT | `/dns/ddns/keys/archived` | `setDdnsKeyArchived` |

  ```bash
  curl -sS -X POST 'https://api.namefi.io/v-next/dns/ddns/keys' \
    -H "X-API-Key: $NAMEFI_API_KEY" -H 'content-type: application/json' \
    -d '{"normalizedDomainName":"example.com","name":"vps curl updater","hostname":"home.example.com"}'
  ```

  Required `normalizedDomainName`, `name`; optional `hostname`, `zoneWide`
  (default false). Use the narrowest scope that works.
- **Dashboard** — Domain settings → Dynamic DNS, which also prints a snippet.

Username = the public `ddns_…` identifier, password = the secret. §2 does **not**
need this credential (it uses your account API key instead); §3 does.

## 2. Approach (i) — agent-driven updates via MCP / REST

The agent reads the public IP and writes the A/AAAA record directly through the
ordinary DNS endpoints.

**Be honest about the limit:** this runs only while the agent session runs. It is
fine for a demo you are actively watching; it is **not** an unattended service.
Tell the user that plainly, and say what happens when it stops (the record keeps
pointing at the last address it saw).

Endpoints (live OpenAPI, base `https://api.namefi.io/v-next/`):

| Method | Path | operationId |
|---|---|---|
| GET | `/dns/records` | `getDnsRecords` |
| POST | `/dns/records` | `createDnsRecord` |
| PUT | `/dns/record` | `updateDnsRecord` |
| DELETE | `/dns/record` | `deleteDnsRecord` |

Loop:

1. **Read the current public IP** — `curl -s https://api.ipify.org` (v4) or
   `https://api6.ipify.org` (v6). On a VPS the value the DynDNS2 endpoint would
   infer is the same thing.
2. **Read the existing record** with `getDnsRecords` and **compare**. Only call
   `updateDnsRecord` when the value actually differs. A write per tick burns rate
   limit, churns the record's `lastUpdatedAt`, and tells you nothing.
3. Poll on the order of minutes, not seconds.

Send **`X-Namefi-Dynamic-DNS: true`** on the write. It grants nothing — it only
marks the record as dynamically maintained so the dashboard can say so, instead
of the record looking like a hand edit.

Record metadata is never accepted from clients, so this header is the only way to
declare that intent.

## 3. Approach (ii) — `curl` against the DynDNS2 endpoint, on a timer

The durable no-extra-software option. `cron`/`systemd`/`launchd` runs a small
script; the script talks the same DynDNS2 protocol a router would.

### 3a. Find the endpoint host — probe, do not assume

The updater lives at the **root** (`/nic/update`, `/v3/update`) because router
firmware hardcodes those paths. The host is deployment-specific
(`getDdnsUpdateHost()`, `#lib/urls`, falling back to the backend's own host), so
take it from the dashboard snippet or probe candidates:

```bash
curl -sS -u 'wrong:wrong' -A 'probe/1' \
  'https://<candidate-host>/nic/update?hostname=example.com'
```

A live updater answers the plain-text body `badauth`. As of this writing
`api.namefi.dev` does; `ddns.namefi.io` is a CNAME to `api.namefi.io` whose
certificate does not cover it and whose route 404s — **not** usable yet. Probe
every time.

`/v3/update` is the same service and additionally accepts comma-separated
dual-stack `myip=<v4>,<v6>`.

### 3b. The script

Store the credential in a 0600 file, one `identifier:secret` line — **never** in
the command line, where it lands in shell history, `ps` output, and the server's
access log.

```bash
umask 077
printf 'ddns_xxxxxxxxxxxx:%s\n' "$SECRET" > /etc/namefi-ddns.auth
chmod 600 /etc/namefi-ddns.auth && chown root:root /etc/namefi-ddns.auth
```

`/usr/local/bin/namefi-ddns-update.sh`:

```sh
#!/bin/sh
# Update a Namefi A record over DynDNS2. POSIX sh; needs only curl.
set -eu

HOSTNAME_TO_UPDATE="${NAMEFI_DDNS_HOSTNAME:?set NAMEFI_DDNS_HOSTNAME}"
SERVER="${NAMEFI_DDNS_SERVER:-api.namefi.dev}"   # probe this first, see 3a
AUTH_FILE="${NAMEFI_DDNS_AUTH_FILE:-/etc/namefi-ddns.auth}"
STATE_FILE="${NAMEFI_DDNS_STATE_FILE:-/var/lib/namefi-ddns/last-ip}"

[ -r "$AUTH_FILE" ] || { echo "auth file not readable: $AUTH_FILE" >&2; exit 2; }

# Omitting myip lets the server use the request source IP, which is correct on a
# VPS and also correct from behind NAT (the router's address is what it sees).
# Discover it anyway so we can skip pointless writes.
ip=$(curl -fsS --max-time 10 https://api.ipify.org) || {
  echo "could not determine public IP" >&2; exit 1; }

if [ -r "$STATE_FILE" ] && [ "$(cat "$STATE_FILE")" = "$ip" ]; then
  exit 0                       # unchanged: do not spend an update
fi

# Credential via --netrc-like file semantics: -K keeps it out of argv and ps.
response=$(printf 'user = "%s"\n' "$(cat "$AUTH_FILE")" \
  | curl -fsS --max-time 30 -K - \
      -A 'namefi-ddns-sh/1.0' \
      --get --data-urlencode "hostname=$HOSTNAME_TO_UPDATE" \
      --data-urlencode "myip=$ip" \
      "https://$SERVER/nic/update") || {
  echo "update request failed (network/TLS)" >&2; exit 1; }

# HTTP status is always 200. The BODY is the result.
case "$response" in
  good*|nochg*)
    mkdir -p "$(dirname "$STATE_FILE")"
    printf '%s\n' "$ip" > "$STATE_FILE"
    exit 0
    ;;
  abuse*)
    # Rate limited. Do NOT retry now; the timer will come back.
    echo "namefi-ddns: rate limited (abuse) - increase the interval" >&2
    exit 1
    ;;
  badauth*|nohost*|numhost*|notfqdn*|badagent*)
    # Permanent config/ownership problems: retrying cannot help.
    echo "namefi-ddns: $response" >&2
    exit 2
    ;;
  *)
    echo "namefi-ddns: unexpected response: $response" >&2
    exit 1
    ;;
esac
```

Why it is shaped this way:

- `set -eu`, every expansion quoted, and the credential read from a file — it is
  piped in as a `curl` config on stdin (`-K -`), so it never appears in `argv`
  and never reaches `ps` or the shell history.
- **The state file is the abuse guard.** Without a "did it change" check, every
  tick spends an update; the server allows 60/hour per credential and answers
  `abuse` beyond that.
- **Exit codes carry meaning**: `2` = permanent, stop and look; `1` = transient,
  the next tick may fix it. That keeps a misconfiguration from generating a mail
  storm while still surfacing it.
- It never prints the secret, and never puts it in the URL.

`shellcheck`-clean (verified: no findings).

### 3c. Schedule it

**cron** (every 5 minutes; `MAILTO` empty avoids a mail per failure — read the
log instead):

```cron
MAILTO=""
*/5 * * * * NAMEFI_DDNS_HOSTNAME=home.example.com /usr/local/bin/namefi-ddns-update.sh >>/var/log/namefi-ddns.log 2>&1
```

**systemd timer** (preferred on Linux: real logs, no mail, jitter):

```ini
# /etc/systemd/system/namefi-ddns.service
[Service]
Type=oneshot
Environment=NAMEFI_DDNS_HOSTNAME=home.example.com
ExecStart=/usr/local/bin/namefi-ddns-update.sh

# /etc/systemd/system/namefi-ddns.timer
[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
RandomizedDelaySec=60
[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now namefi-ddns.timer
journalctl -u namefi-ddns -f
```

**launchd** (macOS): a `~/Library/LaunchAgents/io.namefi.ddns.plist` with
`StartInterval` `300` and the same `ProgramArguments`.

**Etiquette:** 5 minutes is the conventional floor and matches `ddclient`'s
default. `abuse` is a real return code, not a warning — do not poll every
minute, and never loop on failure.

## 4. Verify — prove it, do not assume

```bash
dig +short home.example.com A
dig +short home.example.com AAAA
curl -s https://api.ipify.org; echo        # should match
```

Check the DynDNS2 **body**, not the HTTP status — the endpoint answers 200 for
success *and* every failure, so a status-code check reports success always:

```bash
/usr/local/bin/namefi-ddns-update.sh; echo "exit=$?"
```

Then test reachability **from outside the LAN** — a phone on cellular, a VPS,
any off-network host:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://home.example.com/
```

**Hairpin-NAT caveat:** an in-LAN test proves nothing either way. Many routers do
not hairpin (inside failure is meaningless); others resolve the name to the LAN
host (inside success is meaningless). Only an external client counts. An inside
connection failing in **milliseconds** is the router rejecting LAN→WAN-IP
traffic; a **timeout** points at a genuinely closed port. Locally, use the LAN
address and internal port.

No off-network host? Use a distributed checker — and **control-test it** against
a known-good URL first, since a broken checker fails every URL and looks exactly
like a closed port:

```bash
R=$(curl -sS -H 'Accept: application/json' \
  'https://check-host.net/check-http?host=http%3A%2F%2Fhome.example.com%3A32400%2F&max_nodes=6')
RID=$(echo "$R" | python3 -c "import json,sys;print(json.load(sys.stdin)['request_id'])")
sleep 20; curl -sS -H 'Accept: application/json' "https://check-host.net/check-result/$RID"
```

Re-run with more nodes before believing a partial failure — a lone failing node
is usually the node, not you.

### Reporting the result — both URLs, always (NAT only)

Behind NAT, the external URL does **not** work from the machine the user is
sitting at. Handing them only that reads as "it's broken." Print both, as the
default:

```
local:    http://127.0.0.1:8080/hello.html
external: http://home.example.com:32400/hello.html
```

and always add:

> To reach the **external** URL from inside this LAN, add the hostname to
> `/etc/hosts` (`192.168.1.11  home.example.com`) — this router does not hairpin.
> That covers this machine only; other LAN devices need their own entry or a
> router DNS override. `/etc/hosts` maps a name to an **address, not a port** —
> if the external URL carries `:32400` while the service listens on `:8080`,
> publish it on the matching port too.

On a **VPS** there is no NAT and one URL is correct everywhere — a `local:` line
there is just noise.

## 5. Homelab only — the port forward

DNS reaches the router's front door; a forward reaches this machine. Offer to do
it rather than leaving it to the user:

- **Automatic** — `natprobe` is a diagnostic CLI for
  UPnP IGD / PCP / NAT-PMP. Optional, but the quickest way to learn whether the
  router will cooperate:

  ```bash
  # Source of truth: github.com/samishal1998/natprobe
  git clone --depth 1 https://github.com/samishal1998/natprobe
  go build -o ~/bin/natprobe ./natprobe/cmd/natprobe
  # One-liner alternative: the repo's go.mod still declares the pre-move module
  # path, so `go install` must use it until that is updated (verified 2026-08-24):
  #   go install github.com/d3servelabs/natprobe/cmd/natprobe@latest
  natprobe check                    # what the gateway supports, and why things fail
  natprobe map --port 8080/tcp      # request a forward
  ```

- **Manual** — say exactly what to click: router admin UI (usually
  `http://192.168.1.1`) → *Port Forwarding* / *Virtual Server* / *NAT* → external
  port (80/443, or 8080 if the ISP blocks 80), protocol TCP, internal IP = this
  machine's LAN address, internal port = the service port. Also give the machine
  a **DHCP reservation**, or the rule breaks when the lease changes.

Protocol-obtained mappings are leases and vanish on router reboot; a manual rule
persists. Neither approach in this skill re-establishes a lease, so for a homelab
that reboots, prefer the manual rule (or `namefi-dyndns-with-cli`, which renews
mappings and gates publishing on them).

**Two traps before concluding forwarding is impossible** (detail in
[`../namefi-dyndns/references/inbound-reachability.md`](../namefi-dyndns/references/inbound-reachability.md)):

- **A `501 (Action Failed)` on external 80/443 is not "UPnP is broken."** Routers
  routinely reserve privileged ports while granting `18080` / `8443` / `32400`
  on the first ask. Retry high. Prefer `18080`/`32400` for plain HTTP — `8443`
  implies TLS and confuses scanners and proxies.
- **The host firewall blocks inbound before the router matters.** Loopback 200 +
  LAN-IP timeout means the OS firewall is dropping it, and no port-forward rule
  will help. macOS allowlists *per binary* — the listening process, not the CLI
  you typed.

## 6. Failure modes

| Body / symptom | Meaning | Fix |
|---|---|---|
| `good <ip>` | updated | — |
| `nochg <ip>` | already correct | **success**. Do not treat as an error, do not retry. |
| `badauth` | wrong identifier or secret; deliberately undifferentiated (also covers revoked/invalidated keys) | re-read the auth file (a stray newline or a truncated secret does this); reissue if revoked |
| `nohost` | scope does not cover this hostname, **or** you no longer own the domain (re-verified every request) | reissue the credential with the right scope, or confirm the NFT is still yours |
| `numhost` | several same-type records on the hostname — ambiguity is never guessed | delete duplicates so exactly one A (and/or one AAAA) remains. One A + one AAAA is fine. |
| `notfqdn` | not a fully-qualified name | `home.example.com`, not `home` |
| `abuse` | rate limited (60 updates/hour per credential; 20 failed auths/15min per IP) | lengthen the timer, add the state-file check, stop any retry loop |
| `badagent` | missing User-Agent | send one — the script's `-A` flag |
| `dnserr` / `911` | server-side | back off; nothing to fix locally |
| JSON `not_found`, TLS name mismatch | wrong `SERVER` host | re-probe per §3a |
| `dig` right, still unreachable | no port forward, or the ISP blocks the port | §5; try a high port (18080/32400) |
| public v4 in `100.64.0.0/10` | **CGNAT** | inbound IPv4 impossible. IPv6-only track, ask the ISP for a real IPv4, or a tunnel/relay. |
| public v4 looks ordinary, still nothing arrives | **carrier NAT via RFC1918** — the `100.64/10` test misses it | compare the **router's WAN IP** (UPnP `GetExternalIPAddress`) with the public address; private → no forwarding path exists |
| loopback 200, LAN IP times out | host firewall, not the router | allowlist the listening binary (macOS ALF is per-binary) |
| `501 (Action Failed)` mapping 80/443 | router reserves privileged ports | retry a high external port |

Only A and AAAA are ever touched; a CNAME on the hostname blocks updates and must
be removed first.

## Secret hygiene

- **Never `curl -u user:pass` and never `?password=` in a URL.** Both reach shell
  history, `ps`, and server access logs. Read from a 0600 file (`-K -` as above)
  or an environment variable owned by the service unit.
- Auth file **0600**, owned by whoever runs the timer (root for a system unit).
  Never commit it; this repo git-ignores `ddclient*.conf` for the same reason.
- Never echo the secret to a log, a transcript, or an issue. Redact when showing
  config.
- Scope to the single hostname unless zone-wide is genuinely needed. Retire a key
  with `revokeDdnsKey` — **archiving is not revoking** (it only hides an already
  dead key, and a live key cannot be archived).

## Before handoff

- The endpoint host was **probed** and answered `badauth` for bad credentials.
- A real run printed `good`/`nochg` and the state file now holds the address.
- `dig +short` matches the real public address.
- The timer/cron entry is installed and its interval is ≥5 minutes.
- An **external** client reached the service.
- If §2 (agent-driven) was used, the user was told it stops with the session, and
  a durable path (§3, or the CLI skill) was offered.
- For TLS and hostname routing, hand off to `namefi-https-and-routing`.
- For a NAT topology, the result was reported as **both** `local:` (loopback) and `external:` (hostname) URLs, with the `/etc/hosts` note for reaching the external one from inside the LAN.

---

*Part of the five-skill `namefi-dyndns*` family. The shared reference
`../namefi-dyndns/references/inbound-reachability.md` (carrier-NAT detection,
UPnP port refusals, host firewalls, hairpin NAT, external verification) ships in
the `namefi-dyndns` skill — copy the whole set, or that link dangles. The
summaries inline above stand on their own if it is missing.*
