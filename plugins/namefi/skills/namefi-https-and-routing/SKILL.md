---
name: namefi-https-and-routing
description: >-
  Use when putting a reverse proxy in front of locally-developed services so a
  domain serves real HTTPS - getting a Let's Encrypt or ZeroSSL certificate for
  a home or VPS service, routing several apps behind one public IP and one port
  443 by Host header, choosing between Caddy and Traefik, debugging ACME
  HTTP-01/TLS-ALPN-01/DNS-01 failures, wildcard certificates, "serve my local
  app over https on my domain", or "connection refused from outside but works
  inside".
---
# HTTPS Termination and Hostname-Based Routing

Companion to the dynamic-DNS skills. Those get a hostname resolving to the
user's current IP (`namefi-dyndns-with-cli`, `namefi-dyndns-with-ddclient`,
`namefi-dyndns-without-tooling`). This one makes that hostname serve real HTTPS
and route to the right local service. Do not duplicate their content — if the
name does not resolve to the right IP yet, send the user there first.

## Plan first — `simple` (default) and `advanced`

This skill takes the family's mode argument (`simple`, the default, or
`advanced`) and keeps the mode of the hand-off that brought you here. When a
dyndns skill hands off with an **accepted plan**, its HTTPS lines are already
accepted — don't re-confirm them; speak up only if detection here contradicts
the plan (say, 443 stopped being reachable). Invoked directly in simple mode:
detect, fill the defaults below, present one numbered plan (choice + evidence
per line), one *"accept, or name a line to change"* prompt. Advanced mode stops
at each decision with the default marked *(recommended)*. The family-wide
defaults table lives in [`../namefi-dyndns/SKILL.md`](../namefi-dyndns/SKILL.md).

Defaults this skill owns:

- **Proxy** — Caddy. Traefik only when detection finds a labeled compose fleet
  or a Traefik already running (the section below has the full reasoning).
- **Runtime** — `docker` present → the container variant; absent → the native
  binary. Pick by `command -v docker`, don't ask.
- **Challenge** — 80 and 443 reachable from outside → HTTP-01 (Caddy adds its
  TLS-ALPN fallback itself); that is the default, not a question. Neither
  reachable → DNS-01 is the only path — present it with the honest Namefi
  caveats below.
- **Staging first** — always a plan line; the production switch is its own step.

## Pick one: Caddy by default

**Default to Caddy 2.x.** Automatic HTTPS is on by default, the config is a few
lines per hostname, and it works identically with or without Docker. For a
homelab or a single VPS with a handful of static services, it is the smaller,
less breakable answer.

**Choose Traefik v3 when** services are containers that come and go: Traefik
discovers them from Docker labels, so adding a service is a label on that
service, not an edit to a central file. Also pick it if the user already runs
Traefik or needs its middleware/observability ecosystem.

Not a reason to pick Traefik: "it is more production-grade". Both terminate TLS
correctly. The cost of Traefik is a static-vs-dynamic config split and a much
larger surface to misconfigure.

## Versions these configs target

- Caddy **2.x** (current stable line 2.11.x; `caddy:2.11-alpine` image).
- Traefik **v3** (current stable line v3.7.x; `traefik:v3.7` image).

Traefik v2 label and static-config syntax differs (`rule` syntax, `certificatesResolvers`
placement, `entryPoints` redirect config). If the user is on v2, say so and
either upgrade or translate deliberately — do not paste v3 config into a v2 file.

## Caddy: the multi-hostname case

`/etc/caddy/Caddyfile` (non-Docker) — Caddy 2.x:

```caddyfile
{
	email you@example.com
	# While iterating, use STAGING so you cannot burn production rate limits.
	# Comment this line out for the real certificate. See "Staging first" below.
	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
	log {
		output stdout
		format json
	}
}

app.example.com {
	reverse_proxy 127.0.0.1:3000
}

api.example.com {
	reverse_proxy 127.0.0.1:8080 {
		# Sensible timeouts; the default is unlimited for streaming responses.
		transport http {
			dial_timeout 5s
			response_header_timeout 30s
		}
	}
}
```

That is the whole thing. Caddy listens on 80 and 443, redirects HTTP to HTTPS
automatically, obtains and renews certificates automatically, and routes by the
site address (the `Host:` header). `X-Forwarded-For` / `X-Forwarded-Proto` are
set for you.

Run it:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile   # syntax check, no network
sudo systemctl reload caddy                          # or: caddy run --config ...
```

Docker variant (`compose.yaml`):

```yaml
services:
  caddy:
    image: caddy:2.11-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data     # ACME account + certificates. Persist this.
      - caddy_config:/config
volumes:
  caddy_data:
  caddy_config:
```

With services in the same compose project, use their service names:
`reverse_proxy app:3000`. If the services run on the Docker host instead, use
`host.docker.internal:3000` (add `extra_hosts: ["host.docker.internal:host-gateway"]`
on Linux) or run the proxy with `network_mode: host`.

**Losing `/data` means re-issuing every certificate**, which is exactly how
people hit rate limits. Persist it.

## Traefik v3: the multi-hostname case

Static config `traefik.yml` (mounted at `/etc/traefik/traefik.yml`):

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt
    transport:
      respondingTimeouts:
        readTimeout: 60s
        writeTimeout: 60s
        idleTimeout: 180s

providers:
  docker:
    exposedByDefault: false   # opt-in per service. Keep this false.

certificatesResolvers:
  letsencrypt:
    acme:
      email: you@example.com
      storage: /letsencrypt/acme.json
      # STAGING while iterating. Delete this line (and acme.json) for production.
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      httpChallenge:
        entryPoint: web

log:
  level: INFO
accessLog: {}

api:
  dashboard: true
  insecure: false   # never true on a public host
```

`compose.yaml`:

```yaml
services:
  traefik:
    image: traefik:v3.7
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [edge]

  app:
    image: my/app
    networks: [edge]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.rule=Host(`app.example.com`)
      - traefik.http.routers.app.entrypoints=websecure
      - traefik.http.routers.app.tls.certresolver=letsencrypt
      - traefik.http.services.app.loadbalancer.server.port=3000

  api:
    image: my/api
    networks: [edge]
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.example.com`)
      - traefik.http.routers.api.entrypoints=websecure
      - traefik.http.routers.api.tls.certresolver=letsencrypt
      - traefik.http.services.api.loadbalancer.server.port=8080

networks:
  edge:
```

Non-Docker Traefik (services are plain processes on the host) — swap the Docker
provider for the file provider. In `traefik.yml`:

```yaml
providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
```

`/etc/traefik/dynamic/routes.yml`:

```yaml
http:
  routers:
    app:
      rule: "Host(`app.example.com`)"
      entryPoints: [websecure]
      service: app
      tls:
        certResolver: letsencrypt
    api:
      rule: "Host(`api.example.com`)"
      entryPoints: [websecure]
      service: api
      tls:
        certResolver: letsencrypt
  services:
    app:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:3000"
    api:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8080"
```

`chmod 600 ./letsencrypt/acme.json` — Traefik refuses to start if it is
world-readable, and it holds the ACME account key.

## The certificate story

Three ACME challenge types. Pick by what is reachable.

**HTTP-01** — the CA fetches `http://<host>/.well-known/acme-challenge/<token>`
on **port 80**. Default for both tools. Fails when: port 80 is blocked by the
ISP (very common on residential lines), not forwarded through NAT, or the host
is behind CGNAT. Cannot issue wildcards.

**TLS-ALPN-01** — same idea on **port 443** using a special TLS handshake.
Useful when 80 is blocked but 443 is not. Caddy tries it automatically as part
of its issuer set; Traefik: replace `httpChallenge` with `tlsChallenge: {}`.
Requires the proxy to own 443 directly — it breaks behind another TLS
terminator. Cannot issue wildcards.

**DNS-01** — prove control by publishing a `_acme-challenge.<host>` TXT record.
The **only** option when neither 80 nor 443 is reachable from the internet
(CGNAT, ISP blocks, internal-only services), and the **only** way to get a
**wildcard** (`*.example.com`) certificate. Needs credentials for the DNS zone.

> **Homelab reality check — expect to need DNS-01.** Consumer routers routinely
> refuse to map external **80 and 443** over UPnP (`HTTP 500: UPnP error 501
> Action Failed`) while granting high ports like `18080`/`32400` instantly, since
> the privileged ports are reserved for their own admin UI or blocked by the ISP.
> When that happens **both HTTP-01 and TLS-ALPN-01 are impossible** — they are
> defined on 80 and 443 specifically and no high-port mapping substitutes.
> Confirmed on a consumer router 2026-08-21: 80 and 443 refused, 18080/8443/32400
> granted. Test the mapping *before* choosing a challenge type, and note the
> service can still be perfectly reachable on a high port over plain HTTP — that
> proves nothing about 80/443. See
> [`../namefi-dyndns/references/inbound-reachability.md`](../namefi-dyndns/references/inbound-reachability.md) §2.

### DNS-01 with Namefi-hosted DNS

**Verified as of 2026-08-19: there is no Namefi DNS provider module for Caddy
(nothing in the `caddy-dns` org) and none in Traefik's lego provider set.** Do
not tell the user to install one. Options, honestly:

1. **Manual / one-off TXT.** Namefi manages records over its API and MCP
   (`/v-next/dns/records`), so you can create the `_acme-challenge` TXT record
   yourself and complete the challenge with a manual-mode client
   (`certbot certonly --manual --preferred-challenges dns`), then point the proxy
   at the resulting files (Caddy: `tls /path/cert.pem /path/key.pem`; Traefik: a
   file-provider `tls.certificates` entry). **This does not auto-renew** — you
   repeat it every ~60 days, or script it against the API.
2. **A hook-based renewal script.** Wrap the same API calls in a `certbot`
   `--manual-auth-hook` / `--manual-cleanup-hook` (or `lego --dns exec`) so
   renewal is automated. This is code that would need writing; it does not exist
   in this repo today.
3. **Delegate the challenge.** `CNAME _acme-challenge.app.example.com` to a zone
   or an `acme-dns` instance whose provider *is* supported, then use that
   provider's plugin. Keeps auto-renewal with no custom code.
4. **Nameservers elsewhere.** If the zone is served by Cloudflare/Route53/etc.,
   use the supported plugin directly (Caddy needs a custom build via `xcaddy`;
   Traefik ships lego providers in the standard binary).

Prefer HTTP-01 or TLS-ALPN-01 whenever ports are reachable. Only reach for
DNS-01 when they are not, or a wildcard is genuinely needed.

### Staging first — this is the mistake that bites people

Let's Encrypt production limits issuance per registered domain per week, and
failed validations count against a separate failure limit. A misconfigured proxy
retrying in a loop can lock the user out for days.

**Iterate against staging** (`https://acme-staging-v02.api.letsencrypt.org/directory`,
shown in both configs above). Staging certificates are untrusted, so browsers and
plain `curl` will complain — that is expected; use `curl -k` and inspect the
issuer. Only after the full path works end to end, remove the staging line,
**delete the stored ACME state** (Caddy: the staging entries under `/data/caddy`;
Traefik: `acme.json`), restart, and confirm the issuer is real Let's Encrypt.

Caddy also falls back to ZeroSSL/Google Trust Services automatically if Let's
Encrypt fails, so a "working" cert may come from a different issuer — check.

## Security basics

- **Never expose the Traefik dashboard publicly.** Keep `api.insecure: false`.
  If it is needed, put it behind a router with basicauth middleware and a
  hostname you control, or bind it to localhost and reach it over SSH.
- `chmod 600` the ACME storage (`acme.json`, Caddy's `/data`). It holds account
  keys and private keys.
- Redirect HTTP to HTTPS (Caddy: automatic; Traefik: the `web` entrypoint
  redirection above).
- `exposedByDefault: false` in Traefik so a new container is not silently
  published.
- Mount `docker.sock` read-only. It is root-equivalent; a compromised Traefik
  with write access owns the host.
- Set read/write/idle timeouts (above) so a slow client cannot pin a connection.
- Only route hostnames you actually own. Do not run wide-open on-demand TLS
  (see the repo note below).

## Repo precedent

This repo runs Caddy in two places. Read them for house patterns, but **neither
is a good starting point for a homelab user** — both are on-demand-TLS multi-tenant
edges for parked customer domains, not per-hostname routing:

- `deploy/park-https-proxy/caddy/conf/Caddyfile`
- `deploy/kustomize/base/caddy/Caddyfile`

Worth borrowing: JSON logging to stdout, and the kustomize file's comments about
`on_demand_tls` needing a real `ask` endpoint plus `interval`/`burst`. An
on-demand config whose `ask` always answers "ok" lets anyone pointing DNS at your
IP trigger certificate issuance on your account and exhaust your rate limits. If
a user asks for on-demand TLS, that gate is mandatory.

## Verification that proves it works

Do all of these from **outside the network** — a phone on cellular, a VPS, or a
web-based checker. Hairpin NAT makes tests from inside the LAN succeed even when
the service is unreachable from the internet.

```bash
# 1. Does the name point where you think, and does 443 answer?
dig +short app.example.com A
dig +short app.example.com AAAA
curl -vI https://app.example.com

# 2. Inspect the certificate: subject, issuer, validity.
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 3. Routing actually keys on Host - both names must hit different backends.
curl -sI https://api.example.com | head -1

# 4. HTTP redirects to HTTPS.
curl -sI http://app.example.com | head -2
```

The issuer line must read a real Let's Encrypt intermediate (e.g.
`O = Let's Encrypt`), **not** `(STAGING)` / `Fake LE`. If it still says staging,
the staging config or staging ACME state was not removed.

Read the proxy's own logs — they say precisely why ACME failed:

```bash
journalctl -u caddy -f              # or: docker compose logs -f caddy
docker compose logs -f traefik
```

## Failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| ACME HTTP-01 times out, works on LAN | Port 80 not forwarded, or ISP blocks it | Forward 80+443 to the proxy machine; if 80 is blocked, switch to TLS-ALPN-01; if both blocked, DNS-01 |
| Works inside, `connection refused` outside | NAT/port-forward missing, or proxy bound to `127.0.0.1` | Check the forward rule and that the proxy listens on `0.0.0.0`; test from cellular |
| Public IP is `100.64.x.x`–`100.127.x.x` | CGNAT — no inbound is possible at all | DNS-01 for the cert, plus a tunnel (Tailscale funnel, Cloudflare Tunnel, VPS + WireGuard) for traffic |
| Cert issued for the wrong name / NXDOMAIN | DNS record missing, stale, or on the wrong subdomain | Re-check with `dig`; the dyndns skills own this half |
| `too many certificates already issued` | Production rate limit hit | Wait out the window; switch to staging to finish debugging; stop the retry loop first |
| Renewal keeps failing silently | ACME storage not persisted across restarts | Persist Caddy `/data` / Traefik `acme.json`, `chmod 600` |
| Traefik router never matches | `traefik.enable=true` missing, wrong network, or wrong `server.port` | `docker compose logs traefik`; the port label is the **container** port |
| Works over IPv4, fails for some users | An AAAA record exists but v6 is not routed/forwarded | Forward 80/443 over IPv6 too, or remove the AAAA record |
| Browser warns but `curl -k` works | Staging certificate | Remove the staging CA, delete ACME state, restart |

## Before handoff

- Confirm the hostname resolves to the current public IP (dyndns skills).
- Confirm 80 and 443 reach the proxy from outside, or that DNS-01 is in use.
- Confirm the issuer is production, not staging.
- Confirm each hostname routes to its own backend, not just the first one.
- Confirm ACME storage is persisted and mode 0600.

---

*Part of the five-skill `namefi-dyndns*` family. The shared reference
`../namefi-dyndns/references/inbound-reachability.md` (carrier-NAT detection,
UPnP port refusals, host firewalls, hairpin NAT, external verification) ships in
the `namefi-dyndns` skill — copy the whole set, or that link dangles. The
summaries inline above stand on their own if it is missing.*
