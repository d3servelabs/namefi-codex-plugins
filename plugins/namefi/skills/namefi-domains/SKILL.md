---
name: namefi-domains
description: Search, register, and manage domain names and their DNS through Namefi; use when checking domain availability or pricing, buying/registering a domain, listing owned domains, editing DNS records (A, AAAA, CNAME, MX, TXT, NS, CAA...), configuring parking, forwarding, or auto-renew, or finding outbound leads for a domain.
metadata:
  short-description: Search, register, and manage domains and DNS via Namefi
---

# Namefi domains

This plugin bundles the Namefi MCP server, so its tools are available as
`Namefi:<toolName>`. **Always prefer those tools over `curl`/REST.**

## Auth

Every Namefi call is authenticated, including availability search and DNS reads —
there is no anonymous access.

The bundled server declares an `oauth_resource`, so Codex prompts for sign-in on
install. If tools are missing or return 401/unauthorized, ask the user to complete
that sign-in rather than retrying. If OAuth isn't usable (headless, CI), they can
generate an API key at <https://namefi.io/api-key> and pass it as an `x-api-key`
header on the server config.

Don't retry a failing tool more than once before surfacing the auth step.

## Registration flow

1. If the user is still choosing a name, use the suggestions tool — don't guess
   candidates and check them one at a time.
2. Check availability first — never register a name the user hasn't seen priced.
   The response carries the price, registrar, and the valid duration range; use
   that range rather than assuming one.
3. Confirm the exact name, duration, and price with the user before submitting an
   order. Registration spends real money and is not reversible.
4. Register (`normalizedDomainName`, `durationInYears`, optional NFT-receiving
   wallet address). Alternative payment paths exist — USDC via x402, MPP, and a
   trial registration for eligible domains — so offer them if the user can't or
   won't pay by card.
5. Poll the order until it reaches a terminal state; report the order URL
   (`https://namefi.io/orders/<orderId>`).

### Hand off to a human instead

When the user would rather pay in the browser, or agent-side payment isn't set up,
build a cart link and stop there:

```
https://namefi.io/cart/add-from-url?add_to_cart=example.com,example2.com
```

## DNS

Read records before editing so you can show a diff of what changes. Use the batch
tools for multi-record edits rather than looping single-record calls. Supported
types: A, AAAA, CNAME, MX, TXT, NS, SOA, PTR, SRV, CAA, DS, TLSA, SSHFP, HTTPS,
SVCB, NAPTR, SPF.

## REST fallback

Only if MCP is genuinely unavailable or the user asks for raw HTTP.
Base URL `https://api.namefi.io/v-next/`, auth via `x-api-key: nfk_...` or
`Authorization: Bearer <token>`.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/search/availability?domain=X` | Availability |
| GET | `/search/bulk-availability?domains[]=X` | Bulk availability |
| POST | `/orders/register-domain` | Register |
| POST | `/orders/register-domain/records` | Register + apply DNS |
| GET | `/orders/{orderId}` | Poll order |
| GET | `/user/domains` | Domains owned |
| GET | `/dns/records?zoneName=X` | List records |
| POST/PUT/DELETE | `/dns/record`, `/dns/records`, `/dns/records/batch` | Edit records |
| PUT | `/dns/park`, `/dns/forwarding`, `/domain-config/auto-renew` | Domain config |
| POST/GET | `/outbound/runs`, `/outbound/runs/{runId}/leads` | Lead finding |

Full reference: <https://namefi.io/llms-full.txt>
