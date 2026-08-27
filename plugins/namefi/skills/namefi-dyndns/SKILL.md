---
name: namefi-dyndns
description: >-
  Use when a user wants something they are developing reachable on a real
  hostname without a static IP — show off a local app on their domain, reach a
  home server or NAS or VPS from outside, an IP that keeps changing, set up
  dynamic DNS, point a Namefi domain at this machine, or check whether an ISP is
  CGNAT — and it is not yet clear whether they are on a VPS or behind home NAT,
  or which tool (namefi CLI, ddclient, none) they should use. This is the front
  door that diagnoses the situation and hands off to the right sub-skill. Takes
  an optional mode argument — simple (default; pick defaults from detection and
  present one plan to accept or modify) or advanced (confirm each decision
  individually).
---
# Expose a Local Service on a Real Hostname — Router Skill

This skill **diagnoses and routes**. It contains no configuration files and no
CLI walkthroughs; every one of those lives in a sub-skill. Detect → decide
defaults → present one plan → hand off. If you find yourself pasting a
`ddclient.conf` or a `Caddyfile` here, you are in the wrong skill.

## Modes — `simple` (default) and `advanced`

The whole family takes one optional argument: `/namefi-dyndns simple` (also the
default when no argument is given) or `/namefi-dyndns advanced`. Sub-skills
accept the same argument, and a hand-off keeps the mode it started with.

- **`simple`** — run the detection sweep, fill every choice from the defaults
  table below, and present **one consolidated plan**: numbered lines, each
  stating the choice *and the detected fact that drove it*. Then exactly one
  prompt — *"accept, or name a line to change."* Ask nothing else unless a hard
  blocker (no domain owned, carrier NAT with no IPv6, a credential secret only
  the user holds) makes a defaulted plan impossible; list blockers above the
  plan, never as questions scattered through the flow.
- **`advanced`** — same detection, but stop at each decision point (route,
  hostname, ports, HTTPS, proxy, persistence), show the options with the default
  marked *(recommended)*, and let the user pick before moving on.

In both modes: **never ask what detection already answered.** A question whose
answer is sitting in `command -v` or `namefi dyndns check` output is noise.

**Before promising anything inbound**, read
[`references/inbound-reachability.md`](references/inbound-reachability.md) — the
shared reference on carrier-NAT detection, UPnP port refusals, host firewalls,
hairpin NAT, and how to verify from outside without fooling yourself. All four
`namefi-dyndns-*` skills share it.

## Packaging — these five ship as a set

This family is five sibling skills plus one shared file:

```
namefi-dyndns/                 <- this skill (the front door)
  references/
    inbound-reachability.md    <- shared by all five
namefi-dyndns-with-cli/
namefi-dyndns-with-ddclient/
namefi-dyndns-without-tooling/
namefi-https-and-routing/
```

The sub-skills reach the shared file as
`../namefi-dyndns/references/inbound-reachability.md`, which resolves in any
layout that keeps them **flat siblings** — `.claude/skills/`, `.agents/skills/`,
and a Claude Code plugin's `plugins/<plugin>/skills/` all qualify.

**Copy them together.** Moving one sub-skill on its own leaves a dangling link.
Each sub-skill inlines the actionable summary, so it still functions alone — but
the depth is gone. When packaging into a plugin, take all five directories and
the `references/` folder.

## The two factors that decide everything

1. **Topology** — **VPS** (the machine's own interface holds the public IP, no
   NAT) vs **homelab** (interface holds an RFC1918 address; inbound traffic
   needs router port forwarding).
2. **Tooling** — `namefi` CLI, `ddclient`, or nothing installed.

Observe both. Ask the user only what you genuinely cannot detect.

## Diagnostics — run these, do not ask

### 1. What tooling exists

```bash
command -v namefi; command -v ddclient; command -v docker
```

That settles factor 2 without asking. `docker` is not required but changes the
follow-up HTTPS advice.

### 2. VPS or homelab

Compare the machine's own address with its public address.

```bash
# macOS
ipconfig getifaddr "$(route -n get default | awk '/interface:/{print $2}')"
# Linux
ip -4 -o addr show "$(ip route show default | awk '/default/{print $5; exit}')" | awk '{print $4}'
# public address (both)
curl -4 -fsS https://api.ipify.org; echo
```

- Interface address **equals** the public address → **VPS**, no NAT, no port
  forwarding needed.
- Interface address is **RFC1918** (`10/8`, `172.16/12`, `192.168/16`) →
  **homelab behind NAT**; inbound ports must be forwarded on the router.

When the `namefi` CLI is installed, `namefi dyndns check` answers this directly
(it reports the local address, the public address, NAT state, and CGNAT) — use
it instead of the raw commands.

### 3. Carrier NAT — check before promising anything inbound

If the **public** IPv4 falls inside `100.64.0.0/10` (RFC 6598), the user is
behind carrier-grade NAT. They do not control the router that owns that address,
so **port forwarding cannot work at all**. Say this plainly and stop them before
they spend an evening on it. The only ways out are IPv6 (if the ISP provides it)
or an outbound tunnel. A `100.x` address in the `100.64–100.127` second-octet
range is the tell; `100.128.0.1` and above is ordinary public space.

**But `100.64/10` is not the only form, and the range check misses the rest.**
Real ISPs also carrier-NAT using RFC1918 `10.x`, which shows a perfectly ordinary
public address in `curl ifconfig.co` and passes the test above. `namefi dyndns
check` only tests the RFC 6598 range, so **it will call a carrier-NAT connection
clean.** Do not promise inbound on the strength of that check alone.

Equally, **private hops in `traceroute` do not prove carrier NAT** — ISPs
routinely number core links with RFC1918 while still routing a real public
address to the CPE. Two Egyptian connections tested on 2026-08-21 looked
identical in traceroute (`10.x` at hops 2-6) and had opposite verdicts.

**The decisive test is the router's own WAN address**, via unauthenticated UPnP
`GetExternalIPAddress`. If it equals the address the internet sees → single NAT,
forwarding works. If it is private → carrier NAT, and no router setting helps.
The ready-to-run probe is in
[`references/inbound-reachability.md`](references/inbound-reachability.md) §1.
Run it whenever you are about to tell a user that inbound will work.

### 4. Does the user own a domain on Namefi

Dynamic DNS needs a hostname they control. `namefi domains list` with the CLI;
otherwise the Namefi MCP tools or the API. If they own nothing, that is the
first blocker to resolve — no route works without it.

### 5. IPv6 availability

```bash
curl -6 -fsS --max-time 5 https://api64.ipify.org; echo
```

A working global IPv6 address sidesteps NAT entirely (AAAA record, no port
forwarding) and is the escape hatch when CGNAT blocks IPv4.

### Hairpin-NAT caveat (mention once)

Testing the hostname **from inside the LAN** can succeed while the outside world
still cannot connect, and it can also fail on routers without hairpin NAT while
the outside world is fine. Neither inside result proves anything — verify from a
phone on cellular or an external checker.

The timing distinguishes them: an inside connection that fails in
**milliseconds** is the router rejecting LAN→WAN-IP traffic (no hairpin, says
nothing about the outside); one that **times out** is more likely a genuinely
closed port or a host firewall. Locally, reach the service by its LAN address and
internal port instead.

## Defaults — decided by detection, not by asking

Simple mode fills the plan from this table; advanced mode presents the same rows
as marked recommendations.

| Decision | Default rule (evidence → choice) |
|---|---|
| Route / tool | `namefi` CLI installed → `namefi-dyndns-with-cli`. Else `ddclient` installed → `namefi-dyndns-with-ddclient`. Neither → the plan proposes **installing the namefi CLI** (one plan line the user can veto); `namefi-dyndns-without-tooling` is the fallback when installing anything is off the table. |
| Domain | Exactly one Namefi domain owned → use it. Several → propose the most recently registered and name the alternatives on that plan line. None → hard blocker, surfaced above the plan. |
| Hostname | A service-named label on the chosen domain (`app.`, `demo.`, the project's name). |
| Ports | Probe, don't guess. VPS with 80/443 free, or a NAT router that **grants** 80+443 mappings → use 80/443. Router refuses them (`UPnP error 501` is the classic) → first granted high port, plain HTTP — probe `18080`, then `32400` (skip `8443` for plain HTTP: it conventionally implies TLS). |
| HTTPS | 80 **and** 443 confirmed reachable → **HTTPS is in the plan; do not ask** (hand off to `namefi-https-and-routing`). Only a high port reachable → HTTPS is *not* in the default plan; say why (HTTP-01 and TLS-ALPN-01 are defined on 80/443) and offer DNS-01 as the opt-in line. |
| Reverse proxy | `docker` present → Caddy in a container. Traefik **only** when a labeled compose fleet or a running Traefik already exists. No docker → the native `caddy` binary. |
| Persistence | Anything meant to outlive this session → a supervisor (systemd on Linux, launchd on macOS) is in the plan by default. |
| Verification | Always in the plan: `dig` plus an **external**, control-tested check — never an inside-the-LAN test. |
| Carrier NAT | Detected → the normal plan is impossible. The alternatives (IPv6-only if available, else a tunnel) *are* the plan. |

## Present the plan, then route

Open with the detected facts, one line each, then the plan. Every plan line is a
choice **plus its evidence**:

```
Detected: NAT homelab · no carrier NAT · namefi CLI + docker · you own example.com

Plan (defaults — reply "go", or name a line to change):
1. Route     namefi CLI daemon — already installed; opens the NAT mapping itself
2. Hostname  app.example.com — your only Namefi domain
3. Ports     80 + 443 — the router granted both mappings on probe
4. HTTPS     yes — Caddy in docker (docker found), automatic Let's Encrypt
5. Service   reverse_proxy → 127.0.0.1:3000 — the app found listening
6. Verify    dig + external checker (control-tested), then hand you both URLs
```

In simple mode that block is the **only** confirmation. In advanced mode, walk
the same lines one at a time. Then hand off to the sub-skill with the accepted
plan and the mode.

**Whenever the topology is NAT (not a VPS), every result you report carries two
URLs, as the default:**

```
local:    http://127.0.0.1:8080/hello.html
external: http://home.example.com:32400/hello.html
```

plus the note that reaching the **external** URL from inside the LAN needs an
`/etc/hosts` entry (`192.168.1.11  home.example.com`), because the router does
not hairpin — and that `/etc/hosts` maps a name to an address, **not a port**, so
the service must also be published on the port the external URL names. On a VPS
one URL is correct everywhere; the `local:` line is noise there.

## Routing table

| Topology | Tooling found | Hand off to |
|---|---|---|
| VPS | `namefi` CLI | `namefi-dyndns-with-cli` |
| VPS | `ddclient` | `namefi-dyndns-with-ddclient` |
| VPS | none | Default: propose installing the CLI → `namefi-dyndns-with-cli`; `namefi-dyndns-without-tooling` when installing is off the table |
| Homelab (NAT) | `namefi` CLI | `namefi-dyndns-with-cli` — it also opens the NAT mapping |
| Homelab (NAT) | `ddclient` | `namefi-dyndns-with-ddclient` + forward ports manually |
| Homelab (NAT) | none | Default: propose installing the CLI → `namefi-dyndns-with-cli` (it opens the mapping too); `namefi-dyndns-without-tooling` + forward ports manually when installing is off the table |
| Any | Carrier NAT (RFC6598 **or** private router WAN) | No port forwarding path. IPv6/AAAA, or a tunnel. |

**Follow-up rule:** once the hostname resolves, hand off to
`namefi-https-and-routing` whenever HTTPS is in the plan (80+443 reachable puts
it there by default), the user wants a browser-trusted `https://name/` URL, or
several apps share one IP. It is a companion, never an alternative.
