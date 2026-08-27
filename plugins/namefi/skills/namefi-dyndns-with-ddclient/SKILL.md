---
name: namefi-dyndns-with-ddclient
description: >-
  Use when a user wants to expose a locally-developed service on a real hostname
  without a static IP using ddclient — set up dynamic DNS with an existing
  ddclient install, write or debug a ddclient.conf for Namefi's DynDNS2
  endpoint, run ddclient as a daemon/service, read its output, or map a Dyn
  return code (good/nochg/badauth/nohost/numhost/abuse) to a fix. Covers both a
  VPS (single public IP) and a homelab behind NAT.
---
# Dynamic DNS on Namefi with `ddclient`

Goal: a hostname you own on Namefi resolves to this machine's **current** public
address and stays correct when that address changes. `ddclient` is the standard
cross-platform DynDNS2 client; Namefi speaks DynDNS2, so no Namefi-specific
software is required.

Ground truth for the protocol: `apps/backend/src/lib/ddns/README.md` and
`apps/backend/src/routers/ddns.ts`. TLS and hostname-based routing are a
separate concern — hand off to `namefi-https-and-routing`.

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

- **Config** — `usev4=webv4, webv4=ipify-ipv4`; add the v6 lines only when
  detection shows a routable global IPv6 address (§3's template).
- **Runner** — systemd unit on Linux, `brew services` on macOS — pick by the OS
  you are on, don't ask.
- **Credential** — the scoped dyndns secret is the one user-supplied item;
  surface it as a blocker above the plan.
- **Ports / HTTPS** — same probe-driven rules as the family table: router
  grants 80+443 → forward both and put the `namefi-https-and-routing` hand-off
  in the plan without asking; refuses them → a granted high port, plain HTTP,
  and the plan says why HTTPS is out.

## 0. Choose this path (or don't)

| Situation | Use |
|---|---|
| The `namefi` CLI is available or installable | **`namefi-dyndns-with-cli`** — easier, and it can open NAT mappings itself (PCP/NAT-PMP/UPnP) |
| `ddclient` already installed, or the user wants a standard, distro-packaged daemon | **this skill** |
| No client at all, and installing one is unwanted | `namefi-dyndns-without-tooling` (cron + `curl`) |

Then pick the topology — it changes only §4:

- **(a) VPS** — one machine, one public IP, no NAT. `ddclient` alone is enough.
- **(b) Homelab** — one public IP, several machines behind a router. `ddclient`
  keeps the record correct, but **DNS alone does not make you reachable**: the
  record points at the *router*, so inbound ports must be forwarded to this
  machine. See §4.

Distinguish them by comparing the machine's own address with the public one:

```bash
ip -4 addr show 2>/dev/null || ifconfig            # private 10./172.16-31./192.168. => behind NAT
curl -s https://api.ipify.org; echo                # what the world sees
```

If the public v4 is in `100.64.0.0/10` you are behind **CGNAT** and inbound IPv4
is impossible no matter what you configure (see §7).

## 1. The endpoint — verify the host before writing it down

Namefi's DynDNS2 updater is served at the **root** of the backend
(`/nic/update` and `/v3/update`) because router firmware hardcodes those paths
and they cannot be prefixed.

**Do not assume a hostname.** The server tells clients which host to use
(`getDdnsUpdateHost()`, `#lib/urls`), and it differs per deployment — it falls
back to the backend's own host. Read the value the dashboard/API gives you
(Domain settings → Dynamic DNS shows the exact snippet), or probe:

```bash
curl -sS -u 'wrong:wrong' -A 'probe/1' \
  'https://<candidate-host>/nic/update?hostname=example.com'
```

A working updater answers the plain-text body `badauth`. Anything else (a JSON
`not_found`, a TLS name mismatch) means that host does not serve DDNS. As of
this writing `api.namefi.dev` answers `badauth`; `ddns.namefi.io` is a CNAME to
`api.namefi.io` whose certificate does not cover it and whose route 404s, so it
is **not** usable yet — always probe rather than trusting a doc string.

`server=` in `ddclient.conf` takes a **bare host**, no scheme.

## 2. Get a Dynamic DNS credential

A DDNS credential is permission for one user to update **one hostname's** A/AAAA
records *while that user still owns the domain* — not a permanent bearer secret.
Ownership is re-verified on every update. Credentials do not expire, and the
secret is shown **once**.

Three ways to obtain one, in order of preference for an agent:

1. **MCP** — `https://api.namefi.io/mcp` (streamable-http) exposes the API as
   tools. Call the tool corresponding to `createDdnsKey`.
2. **REST**, with `X-API-Key: <key>` or an OAuth bearer. Base URL
   `https://api.namefi.io/v-next/` (from the live OpenAPI `servers`):

   | Method | Path | operationId |
   |---|---|---|
   | POST | `/dns/ddns/keys` | `createDdnsKey` |
   | GET | `/dns/ddns/keys` | `listDdnsKeys` |
   | POST | `/dns/ddns/keys/revoke` | `revokeDdnsKey` (body `{credentialId}`) |
   | PUT | `/dns/ddns/keys/archived` | `setDdnsKeyArchived` |

   ```bash
   curl -sS -X POST 'https://api.namefi.io/v-next/dns/ddns/keys' \
     -H "X-API-Key: $NAMEFI_API_KEY" -H 'content-type: application/json' \
     -d '{"normalizedDomainName":"example.com","name":"homelab ddclient","hostname":"home.example.com"}'
   ```

   Required: `normalizedDomainName`, `name`. Optional: `hostname` (host scope)
   and `zoneWide` (default `false`). Prefer the narrowest scope that works.
3. **Dashboard** — Domain settings → Dynamic DNS. Walk the user through it when
   no API credential is available; it also prints a ready-made snippet.

The username is the public `ddns_…` identifier; the password is the secret.
Capture the secret at creation — it cannot be retrieved later, only replaced.

**Archiving is not revoking.** Archiving only hides an already-dead key from the
list; a live key cannot be archived. To actually stop a key, `revokeDdnsKey`.

## 3. Write `ddclient.conf`

This repo already carries a working example at the root: **`ddclient.conf`**
(git-ignored via `ddclient*.conf`, because it holds a secret) and
**`start-ddclient.sh`**. Reuse and adapt those rather than inventing a parallel
pair. The checked-in example is the shape below.

Snippet targets **ddclient 4.0.0** (verified with the local
`ddclient --version`). 4.0 deprecated the single `use=`/`web=` pair in favour of
per-family `usev4=`/`usev6=` and `webv4=`/`webv6=`; `use=web` still runs but
warns. Do not copy pre-4.0 Dyn examples.

```conf
# /etc/ddclient/ddclient.conf  — mode 0600, root-owned. Contains a secret.
daemon=300                 # seconds between checks; 300 is polite (see §6, abuse)
ssl=yes

# How to discover this machine's public IPv4 (ddclient >= 4.0 syntax).
usev4=webv4, webv4=ipify-ipv4
# Add IPv6 only if the host genuinely has a routable v6 address:
# usev6=webv6, webv6=ipify-ipv6

# Namefi DynDNS2 endpoint. Bare host, no scheme. Probe it first (§1).
protocol=dyndns2
server=api.namefi.dev
login=ddns_xxxxxxxxxxxx
password='<the-secret>'
home.example.com
```

Notes that actually matter:

- **Quote the password.** Secrets can contain `#`, which otherwise starts a
  comment and silently truncates the value.
- The hostname is a bare line **after** the credential block; every setting above
  applies to the hostnames below it.
- On a **VPS** you may omit the discovery stanza entirely — with no `myip`, the
  server uses the request's source IP, which on a VPS is already correct. The
  explicit `webv4` stanza is what makes it work from behind NAT.
- `usev4=ifv4, ifv4=eth0` reads the *interface* address — correct on a VPS, wrong
  behind NAT (it would publish a private address, which the server ignores).

Permissions and location:

```bash
sudo install -o root -g root -m 0600 ddclient.conf /etc/ddclient/ddclient.conf
# macOS/Homebrew: /opt/homebrew/etc/ddclient.conf ; chmod 600, owned by the runner
```

## 4. Homelab only — the port forward

DNS gets a client to the router's front door; a forward gets it to this machine.
Without one, `dig` looks perfect and the service is still unreachable.

Offer to help rather than leaving it to the user:

- **Automatic** — `natprobe` is a diagnostic CLI for
  UPnP IGD / PCP / NAT-PMP. Optional, but the fastest way to find out whether the
  router will do it without a login:

  ```bash
  # Source of truth: github.com/samishal1998/natprobe
  git clone --depth 1 https://github.com/samishal1998/natprobe
  go build -o ~/bin/natprobe ./natprobe/cmd/natprobe
  # One-liner alternative: the repo's go.mod still declares the pre-move module
  # path, so `go install` must use it until that is updated (verified 2026-08-24):
  #   go install github.com/d3servelabs/natprobe/cmd/natprobe@latest
  natprobe check                    # what the gateway supports, and why things failed
  natprobe map --port 18080/tcp     # request a forward
  ```

- **Manual** — tell the user exactly what to click: router admin UI (usually
  `http://192.168.1.1`) → *Port Forwarding* / *Virtual Server* / *NAT* → add a
  rule with external port (80/443, or a high port — `18080`/`32400` — if the
  ISP blocks 80), protocol TCP,
  internal IP = this machine's LAN address, internal port = the service port.
  Give the machine a **DHCP reservation / static lease** in the same UI, or the
  forward breaks the next time the lease changes.

Mappings obtained via UPnP/PCP/NAT-PMP are *leases* and vanish on router reboot;
a manual rule persists. `ddclient` does not manage mappings at all — that is one
concrete advantage of `namefi-dyndns-with-cli`, which gates publishing on the
mapping actually being granted.

**Two traps before you conclude forwarding is impossible** (full detail in
[`../namefi-dyndns/references/inbound-reachability.md`](../namefi-dyndns/references/inbound-reachability.md)):

- **A `501 (Action Failed)` on external port 80/443 does not mean UPnP is
  broken.** Routers commonly reserve those for their own admin UI while granting
  `18080` / `8443` / `32400` instantly. Retry high before giving up. (Prefer
  `18080`/`32400` over `8443` for plain HTTP — `8443` implies TLS and confuses
  scanners and proxies.)
- **The host firewall blocks inbound before the router matters.** If loopback
  returns 200 but the LAN IP times out, the OS firewall is dropping it, and a
  perfect port-forward rule will not help. macOS allowlists *per binary* — the
  listening process, not the CLI you typed.

## 5. Run it

Test one-shot first, always, before enabling the service:

```bash
sudo ddclient -file /etc/ddclient/ddclient.conf -verbose -noquiet -force
```

- `-force` bypasses the local cache so you see a real update instead of a
  skipped no-op. Use it only for testing — leaving it on defeats caching and
  invites `abuse`.
- `-verbose -noquiet` prints the request and the server's plain-text answer.
- Expect `SUCCESS: … skipped: IP address was already set` (that's `nochg`, which
  is success) or an updated-record line.
- Remove `daemon=` from the config and add `-daemon=300` on the command line if
  you prefer supervisor-controlled intervals.

Then as a service:

```bash
sudo systemctl enable --now ddclient        # Linux (distro package)
sudo journalctl -u ddclient -f              # read its output
brew services start ddclient                # macOS/Homebrew
```

The repo's `start-ddclient.sh` is a *foreground test loop* (a handful of runs
with sleeps), useful for watching behaviour interactively — not a substitute for
a supervised daemon.

## 6. Verify — prove it, do not assume

```bash
dig +short home.example.com A
dig +short home.example.com AAAA
```

Confirm the answer equals `curl -s https://api.ipify.org`. Then test
**reachability from outside the LAN** — a phone on cellular, a VPS, any
off-network host:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://home.example.com/
```

**Hairpin-NAT caveat:** an in-LAN test proves nothing in either direction. Many
routers do not hairpin, so an inside failure is meaningless; others resolve the
name to the LAN host, so an inside success is meaningless too. Only an external
client counts. An inside connection that fails in **milliseconds** is the router
rejecting LAN→WAN-IP traffic; one that **times out** suggests a genuinely closed
port. Locally, use the LAN address and internal port.

No off-network host available? Use a distributed checker — and **control-test
it** against a known-good URL first, because a broken checker fails every URL and
reads exactly like a closed port:

```bash
R=$(curl -sS -H 'Accept: application/json' \
  'https://check-host.net/check-http?host=http%3A%2F%2Fhome.example.com%3A32400%2F&max_nodes=6')
RID=$(echo "$R" | python3 -c "import json,sys;print(json.load(sys.stdin)['request_id'])")
sleep 20; curl -sS -H 'Accept: application/json' "https://check-host.net/check-result/$RID"
```

Re-run with more nodes before believing a partial failure; a lone failing node is
usually the node.

Allow for TTL/propagation before concluding a push failed, and read the DynDNS2
**body**, not the HTTP status — see §7.

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

## 7. Failure modes

The endpoint always returns **HTTP 200**; the outcome is the plain-text body.
Clients that check the status code will report success on every failure.

| Body | Meaning | Fix |
|---|---|---|
| `good <ip>` | record updated | — |
| `nochg <ip>` | already correct | **success**, not an error. Do not force. |
| `badauth` | bad identifier or secret; deliberately undifferentiated (also covers revoked/invalidated keys) | re-check `login`/`password`, watch for an unquoted `#` truncating the secret; issue a new key if revoked |
| `nohost` | the credential's scope does not cover this hostname, **or** you no longer own the domain (ownership is re-verified every request) | widen/reissue the credential's scope, or confirm the NFT is still yours |
| `numhost` | several same-type records on that hostname — ambiguity is never guessed | delete the duplicate A (or AAAA) records so exactly one of each type remains. One A + one AAAA is fine. |
| `notfqdn` | hostname is not a fully-qualified name | use `home.example.com`, not `home` |
| `abuse` | rate limited (60 updates/hour per credential; 20 failed auths/15min per IP) | raise `daemon=` to ≥300s, drop `-force`, stop any loop |
| `badagent` | missing User-Agent | `ddclient` sends one; a hand-rolled client must too |
| `dnserr` / `911` | server-side | back off and retry later; nothing to fix locally |
| JSON `not_found`, or a TLS name error | wrong `server=` host | re-probe per §1 |
| `dig` correct but unreachable | no port forward, or ISP blocks the port | §4; try a high port (18080/32400); check carrier NAT |
| public v4 in `100.64.0.0/10` | **CGNAT** | inbound IPv4 impossible. Options: IPv6-only track, ask the ISP for a real IPv4, or a tunnel/relay. |
| public v4 looks ordinary, still nothing arrives | **carrier NAT via RFC1918** — the `100.64/10` test misses it | compare the **router's WAN IP** (UPnP `GetExternalIPAddress`) with the public address; private → no forwarding path exists |
| loopback 200, LAN IP times out | host firewall, not the router | allowlist the listening binary (macOS ALF is per-binary) |
| `501 (Action Failed)` mapping 80/443 | router reserves privileged ports | retry a high external port |

Only A and AAAA are ever touched; a CNAME sitting on the hostname blocks updates
and must be removed.

## Secret hygiene

- **Never put the secret in a URL on the command line.** `curl -u user:pass` and
  `?password=` both land in shell history, `ps` output, and server access logs.
  `ddclient` reads it from the config file — keep it there.
- Config file **0600**, owned by the account that runs the daemon (root for a
  system service). It is git-ignored here (`ddclient*.conf`); keep it that way.
- Never echo the secret into logs, issues, or a chat transcript. Redact it
  (`password=<redacted>`) when showing a config.
- Scope the credential to the single hostname unless zone-wide is genuinely
  needed; revoke (`revokeDdnsKey`) rather than archive when retiring one.

## Before handoff

- The `server=` host was **probed** and answered `badauth` for bad credentials.
- A one-shot `-force` run printed `good`/`nochg` and the config is back to
  daemon settings without `-force`.
- `dig +short` matches the real public address.
- An **external** client (not on the LAN) reached the service.
- Config is 0600 and the secret appears in no log, URL, or transcript.
- For TLS and hostname routing, hand off to `namefi-https-and-routing`.
- For a NAT topology, the result was reported as **both** `local:` (loopback) and `external:` (hostname) URLs, with the `/etc/hosts` note for reaching the external one from inside the LAN.

---

*Part of the five-skill `namefi-dyndns*` family. The shared reference
`../namefi-dyndns/references/inbound-reachability.md` (carrier-NAT detection,
UPnP port refusals, host firewalls, hairpin NAT, external verification) ships in
the `namefi-dyndns` skill — copy the whole set, or that link dangles. The
summaries inline above stand on their own if it is missing.*
