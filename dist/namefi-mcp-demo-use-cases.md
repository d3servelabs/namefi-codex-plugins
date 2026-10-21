# Namefi MCP demo use cases

Recording: <https://www.loom.com/share/dd12975679b34e518fd937cb0a69fe76>

Use `https://api.namefi.io/mcp-simple` for this recording. It has two public, read-only tools, `dnsOverHttp` and `searchAvailability`; no sign-in or test account is needed. Show each prompt, the tool calls, and the returned result. DNS and availability answers can change, so report what the tools return during recording.

## 1. Check email DNS

**Prompt:** “Validate the public email DNS setup for namefi.io: check MX, SPF, and DMARC, and tell me what these records do and do not prove.”

**Show:** `dnsOverHttp` queries for MX and TXT at `namefi.io`, then TXT at `_dmarc.namefi.io`. Identify the mail exchangers, the SPF record, and the DMARC policy. Explain that these DNS records alone do not prove DKIM signing or successful mail delivery.

## 2. Check website A records

**Prompt:** “Does namefi.io have an A record? Show the IPv4 address and TTL returned by public DNS.”

**Show:** A `dnsOverHttp` query for `namefi.io` with type `A`. Report the returned address and TTL, or accurately say if the answer is empty. An A record alone does not prove that the website responds.

## 3. Check domain availability

**Prompt:** “Is codex-agent.xyz available to register through Namefi?”

**Show:** A `searchAvailability` call for `codex-agent.xyz`. Report `available`, `supported`, and `importable` separately. Do not invent a price or claim this plugin can register the domain.

## 4. Inspect a CNAME

**Prompt:** “What CNAME does www.namefi.io currently point to? Show the target and TTL.”

**Show:** A `dnsOverHttp` query for `www.namefi.io` with type `CNAME`. Report the returned target and TTL. DNS alone does not establish that the target's web service is healthy.

## 5. List nameservers

**Prompt:** “Which nameservers does public DNS return for namefi.io?”

**Show:** A `dnsOverHttp` query for `namefi.io` with type `NS`. List the returned nameserver hostnames. Note that a public resolver may return a cached answer.
