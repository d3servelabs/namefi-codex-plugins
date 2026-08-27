---
name: namefi-dyndns-with-cli
description: >-
  Use when a user wants to expose a locally-developed service on a real hostname
  without a static IP — demo a local app publicly, reach a home server or VPS
  from outside, set up dynamic DNS, keep an A/AAAA record pointed at a changing
  public address, open router port forwarding (NAT-PMP/PCP/UPnP), or diagnose
  CGNAT — using the `namefi` CLI (`namefi dyndns run|update|check`, `namefi
  dyndns creds set`).
---
# Expose a Local Service on a Real Hostname with the namefi CLI

Source of truth: `projects/namefi-dyndns/README.md` (+ `internal/*/README.md`).
The binary is `namefi`; the dyndns feature is one command group under it.

Goal: a hostname you own on Namefi resolves to this machine's *current* public
address, and stays correct when that address changes. TLS/reverse proxy is a
separate concern — see the companion skill `namefi-https-and-routing`
(Traefik/Caddy).

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

- **Credential** — a scoped dyndns credential, never an account login (§2). Its
  secret is the one thing only the user can supply — a blocker to surface, not
  a mid-flow question.
- **Ports** — probe before planning (§3): the router grants 80+443 → map both;
  refuses them (`501`) → first granted high port, plain HTTP — probe `18080`,
  then `32400` (`8443` only when it will actually serve TLS; §3 explains why).
  On a VPS, no `--map` at all.
- **HTTPS** — 80+443 both granted (or a VPS with both free) → the plan includes
  the `namefi-https-and-routing` hand-off without asking. Only a high port →
  it does not, and the plan line says why (HTTP-01/TLS-ALPN-01 need 80/443).
- **Persistence** — anything meant to stay up gets the systemd/launchd service
  (§2 step 4) in the plan by default; a one-off demo gets the bare daemon.

## 0. Install and log in

```bash
curl -fsSL https://namefi.io/cli/setup.sh | sh   # verify it resolves first
namefi login                                     # browser; --device for headless
namefi status                                    # shows active credential + store
```

If the installer URL does not resolve, build from the repo instead:
`cd projects/namefi-dyndns && go build ./cmd/namefi`.

**macOS + a self-built binary cannot open the Keychain** (unsigned →
`Keychain Error (-60006)`). Use `--credentials-store file` or
`NAMEFI_CREDENTIALS_STORE=file`. The default `auto` probes the keyring and falls
back to `~/.namefi/credentials.json` (mode 0600, **not encrypted**) on its own.

## 1. Decide the topology — let the CLI answer it

```bash
namefi dyndns check --hostname home.example.com --map 443/tcp
```

Read the output:

| Signal | Topology | Path |
|---|---|---|
| Machine's own address == the public address, no gateway mapping needed | **VPS** (one public IP, no NAT) | §2 |
| Public address belongs to the router; machine holds a private LAN address (10./172.16./192.168.) | **Homelab** behind NAT | §3 |
| Gateway's external address is in `100.64.0.0/10` | **CGNAT** — no inbound IPv4 possible | §3, CGNAT |

`check` writes nothing. It reports the gateway, which mapping protocol worked
(PCP → NAT-PMP → UPnP IGD), the granted lease, and names CGNAT plainly.

**Do not treat a clean `check` as proof that inbound will work.** It detects
carrier NAT only in RFC 6598 (`100.64.0.0/10`); ISPs that carrier-NAT with
RFC1918 `10.x` pass it and still block every inbound packet. Before promising
inbound, confirm the **router's own WAN address** via UPnP `GetExternalIPAddress`
— see [`../namefi-dyndns/references/inbound-reachability.md`](../namefi-dyndns/references/inbound-reachability.md) §1
for the probe. Equal to the public address → single NAT, forwarding works.
Private → carrier NAT, and no flag or router setting will fix it.

## 2. VPS path (single public IP, no NAT)

1. Pick a hostname on a domain you own: `namefi domains list` (needs an account
   credential; a scoped dyndns credential cannot read the portfolio).
2. Create a **scoped Dynamic DNS credential** in the dashboard
   (Domain settings → Dynamic DNS), then store it:

   ```bash
   namefi dyndns creds set --username ddns_… --scope home.example.com
   namefi dyndns creds show          # never prints the secret
   ```

   Scope forms: `home.example.com` (exact), `*.example.com` (one label),
   `example.com` (zone-wide, apex + descendants).
3. One-shot to prove the wiring, then run the daemon:

   ```bash
   namefi dyndns update --hostname home.example.com
   namefi dyndns run    --hostname home.example.com
   ```

   `--hostname` may be omitted when the credential's scope is an exact host.
4. Install as a service: copy `projects/namefi-dyndns/contrib/systemd/namefi-dyndns.service`
   (Linux) or `contrib/launchd/io.namefi.dyndns.plist` (macOS). For containers
   and services, pass the credential as `NAMEFI_DYNDNS_USERNAME` /
   `NAMEFI_DYNDNS_SECRET` instead of the keyring.
5. Verify per §4.

**Never leave a daemon on an account login.** Precedence is scoped dyndns
credential > `NAMEFI_API_KEY` > account login. A login is a session: it expires
(a `--device` login cannot renew at all) and can rewrite every record you own.
`dyndns run` warns when it lands on one.

## 3. Homelab path (one public IP, several machines behind a router)

Same as §2, plus port forwarding. The public v4 is the *router's* address, so a
record pointing at it is only useful while the router forwards your ports.

```bash
namefi dyndns run --hostname home.example.com --map 443/tcp --map 80/tcp
namefi dyndns run --hostname home.example.com --map 8443:443/tcp   # external:internal
```

- `--map` is repeatable; form is `PORT[:INTERNAL]/PROTO`.
- Mappings are leases, renewed at half-lifetime and re-established on link
  events (a router reboot wipes its table).
- **Publishing is reachability-gated**: with `--map`, the A record publishes only
  while *every* mapping is granted; a blocked v4 logs `withheld: …`. IPv6 is not
  gated (no NAT; v6 pinholes are not implemented yet).
- **Request-or-nothing**: a router that grants a *different* external port is
  treated as failure — dyndns publishes addresses, not ports.

### A `501` on port 80 or 443 is not "UPnP is broken" — retry high

Routers very commonly refuse `AddPortMapping` for external **80/443** (reserved
for their own admin UI, or ISP-blocked) while granting a high port instantly.
The refusal looks like `AddPortMapping answered HTTP 500: UPnP error 501 (Action
Failed)`. Always retry before concluding mapping is unavailable:

```bash
for P in 18080 8443 32400; do
  namefi dyndns check --hostname home.example.com --map ${P}:8080/tcp 2>&1 | grep -i 'port mapping'
done
```

Observed 2026-08-21 on a consumer router: 80 and 443 both refused with `501`;
18080, 8443 and 32400 all granted on the first ask, 2h leases.

Tell the two failure shapes apart — they mean different things:

| Router response | Meaning |
|---|---|
| `connection refused` on SSDP/1900 | no UPnP IGD service at all — enable it, or forward manually |
| `HTTP 500 / UPnP error 501 (Action Failed)` | UPnP works; **that port** was refused — retry high |
| granted on a *different* external port | request-or-nothing: the CLI treats this as failure |

**Pick the external port deliberately.** `8443` conventionally implies TLS, so
plain HTTP on it confuses scanners, corporate proxies, and any browser handed
`https://…:8443`. `18080` / `32400` carry no such expectation.

**No mapping protocol available?** `check` says so. Either enable UPnP IGD /
NAT-PMP / PCP in the router admin UI, or forward the ports manually there and
drop `--map` (you lose the reachability gate — the record then publishes
regardless of whether traffic actually arrives).

### The host firewall blocks inbound before the router ever matters

If `curl http://127.0.0.1:PORT/` returns 200 but `curl http://<LAN-IP>:PORT/`
**times out**, the OS firewall is dropping inbound — forwarded NAT traffic dies
the same way. Fix this before blaming the router.

macOS allowlists **per binary**, so allowlist the actual listening process, not
the CLI you typed — find it with
`lsof -p "$(lsof -ti tcp:PORT | head -1)" -a -d txt`. For Apple's `container`,
`-p 8080:80` is served by
`/usr/local/libexec/container/plugins/container-runtime-linux/bin/container-runtime-linux`.
Note **`sudo` cannot prompt without a TTY** (including under Claude Code's `!`
prefix) — use
`osascript -e "do shell script \"…\" with administrator privileges"`. Full
commands and the Linux equivalents are in
[`../namefi-dyndns/references/inbound-reachability.md`](../namefi-dyndns/references/inbound-reachability.md) §3.

**CGNAT** (gateway external address in `100.64.0.0/10`): inbound IPv4 is
impossible, no flag fixes it. Options: run the IPv6 track only
(`--ipv4=false --ipv6`) if both you and your visitors have v6; ask the ISP for a
real IPv4; or use a tunnel/relay — explicitly out of scope for this CLI.

## 4. Verify — prove it, do not assume

```bash
namefi dyndns check --hostname home.example.com --map 443/tcp   # local view, no writes
dig +short home.example.com A
dig +short home.example.com AAAA
```

Then check reachability **from outside the LAN** — a phone on cellular, a VPS,
or any host off-network:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://home.example.com/
```

**Hairpin-NAT caveat:** testing from inside the LAN can succeed (or fail) for
reasons unrelated to the outside world. Many routers do not hairpin, so an
inside test failing proves nothing; some resolve the name to the LAN host, so an
inside test passing proves nothing either. Only an external client counts.

The timing is the tell: an inside connection that fails in **milliseconds**
(`Couldn't connect`) is the router rejecting LAN→WAN-IP traffic — classic
no-hairpin, and it says nothing about the outside. One that **times out** points
at a genuinely closed port or a host firewall.

Locally, reach the service by its **LAN address and internal port**
(`http://192.168.1.11:8080/`). To make one URL work on both sides you need
*both* halves: an `/etc/hosts` entry mapping the hostname to the LAN address
(that machine only — other LAN devices need their own, or a router DNS
override), **and** the service published on the same port that appears in the
URL. A hosts entry alone does not fix a port mismatch.

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

### No phone handy? Use a distributed checker — and control-test it

```bash
R=$(curl -sS -H 'Accept: application/json' \
  'https://check-host.net/check-http?host=http%3A%2F%2Fhome.example.com%3A32400%2F&max_nodes=6')
RID=$(echo "$R" | python3 -c "import json,sys;print(json.load(sys.stdin)['request_id'])")
sleep 20
curl -sS -H 'Accept: application/json' "https://check-host.net/check-result/$RID"
```

An HTTP entry whose first field is `1` with status `200` is open;
`{"error": "Connection timed out"}` is closed. Expect an occasional flaky node —
**re-run with more nodes before believing a partial failure**; a 5/6 result is
almost always the node, not you.

**Always control-test the checker before believing a negative.** Generic fetch
proxies (`api.allorigins.win`, `api.codetabs.com`) were themselves down on
2026-08-21 and returned `522` for *every* URL including `example.com` — reading
that as "my port is closed" would have been wrong. Point the same tool at a
known-good URL first; if the control fails, the verifier is broken and the test
is void.

Records are pushed with TTL 300s by default (`--ttl`, account-credential path
only); allow a propagation window before concluding a push failed, and re-check
with `dig` against the authoritative answer.

## 5. Failure modes

| Symptom / exit code | Meaning | Fix |
|---|---|---|
| exit 1 | transient (network, upstream) | supervisor restarts; daemon already backs off with jitter |
| exit 2 | config error | fix flags/config; retrying will not help |
| exit 3 | auth failure | re-run `namefi login`, or re-set the scoped credential / `NAMEFI_API_KEY` |
| exit 4 | scope violation | the credential's scope does not cover the hostname; issue one that does |
| `withheld: …`, no A record | a `--map` mapping was refused or remapped | enable PCP/NAT-PMP/UPnP, or forward manually and drop `--map` |
| `check` names CGNAT | ISP-level NAT | see §3 CGNAT |
| record not visible | TTL / propagation, or the push never happened | `namefi dyndns check`, then `dig +short` the authoritative server |
| warning about running on an account login | daemon on a session credential | switch to a scoped dyndns credential or `NAMEFI_API_KEY` |
| `Keychain Error (-60006)` | unsigned macOS binary | `--credentials-store file` |
| `UPnP error 501 (Action Failed)` on `--map 80:…` or `443:…` | router reserves privileged ports | retry a high external port (18080 / 32400); UPnP itself is fine |
| loopback 200, LAN IP times out | host firewall dropping inbound | allowlist the **listening binary** (macOS ALF is per-binary); see reference §3 |
| `check` says clean, but nothing reaches you | carrier NAT outside `100.64/10` | compare router WAN IP (UPnP `GetExternalIPAddress`) with the public address |
| inside test fails in **milliseconds** | no hairpin NAT | meaningless — verify from off-network; use the LAN address locally |
| external checker fails for *every* URL | the checker is down, not your port | control-test against `http://example.com/` before concluding |

## Before handoff

- `namefi dyndns check` clean, `dig` returns the expected address, and an
  **external** client reached the service — a phone on cellular or a distributed
  checker, never an inside test.
- If you ever told the user inbound would work, you confirmed the **router's WAN
  address** is the public one, not merely that `check` found no `100.64/10`.
- The daemon runs under a supervisor on a non-expiring credential, not a login.
- For TLS and hostname-based routing, hand off to `namefi-https-and-routing`.
- For a NAT topology, the result was reported as **both** `local:` (loopback) and `external:` (hostname) URLs, with the `/etc/hosts` note for reaching the external one from inside the LAN.

---

*Part of the five-skill `namefi-dyndns*` family. The shared reference
`../namefi-dyndns/references/inbound-reachability.md` (carrier-NAT detection,
UPnP port refusals, host firewalls, hairpin NAT, external verification) ships in
the `namefi-dyndns` skill — copy the whole set, or that link dangles. The
summaries inline above stand on their own if it is missing.*
