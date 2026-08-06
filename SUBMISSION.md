# Submitting the Namefi plugin to the Codex Plugin Marketplace

The Codex Plugin Marketplace (<https://www.codex-marketplace.com>) is a
community-run catalog, **not affiliated with OpenAI**. Submissions are validated
automatically and anything ambiguous goes to manual review before publication.

## Pre-submission checklist

- [ ] Repo is **public** on GitHub at `d3servelabs/namefi-codex-plugins`
- [ ] `plugins/namefi/.codex-plugin/plugin.json` parses and has `name`, `version`,
      `description` (the three required fields)
- [ ] `skills`, `mcpServers`, `composerIcon`, and `logo` paths all resolve
- [ ] Skill directory name matches its frontmatter `name` (`namefi-domains`)
- [ ] `interface.shortDescription` is ≤125 characters
- [ ] Install works end-to-end from the public repo:

  ```bash
  npx codex-marketplace add d3servelabs/namefi-codex-plugins/plugins/namefi --plugin --project
  ```

- [ ] Install writes `[plugins."namefi@<marketplace>"] enabled = true` into
      `~/.codex/config.toml`, and the plugin lands under `./plugins/namefi`
- [ ] In Codex, the sign-in prompt appears, and after authenticating a live call
      works — e.g. "is acme-robotics.com available and what does it cost?"
- [ ] `npx codex-marketplace remove namefi --project` cleans up

The CLI itself was verified at v0.2.1: the flags above are real, and a test install
of the official `openai/plugins` notion plugin round-tripped cleanly.

## Submit

Go to <https://www.codex-marketplace.com/submit> (sign-in required) and enter the
repository URL:

```
https://github.com/d3servelabs/namefi-codex-plugins
```

A GitHub tree URL also works and is the way to pin an exact branch, tag, or commit.

### Values the manifest declares

| Field | Value |
| --- | --- |
| Plugin name | `namefi` |
| Version | `0.1.0` |
| Display name | Namefi |
| Category | Productivity |
| Capabilities | Interactive, Read, Write |
| Keywords | domains, dns, domain-registration, namefi, web3, mcp |
| Homepage / website | https://namefi.io |
| License | MIT |
| Developer | D3Serve Labs |

## Example use cases

Example 1: check if a domain you want like `acme-robotics.com` is available and what it costs.

Example 2: brainstorm names — "find me a short .com for an AI invoicing startup" — and check the whole shortlist at once.

Example 3: register `acme-robotics.com` for 2 years and watch the order through to completion.

Example 4: pay in USDC instead of by card, or grab a cart link and check out yourself in the browser.

Example 5: list the domains you already own and turn on auto-renew across all of them.

Example 6: point `acme-robotics.com` at your server — "add an A record for 76.76.21.21".

Example 7: migrate email — "move all my MX records over to Google Workspace" — in one batch instead of record by record.

Example 8: park a domain you're not using yet and forward it to your main site.

Example 9: find buyers for the domains you're sitting on and draft the outreach emails.

## What reviewers will look at

- The bundled MCP server (`https://api.namefi.io/mcp`) is a network egress point, so
  expect questions about what leaves the machine and how auth works. Answer: OAuth
  2.1 + PKCE with dynamic client registration, or a user-supplied `x-api-key`; no
  credentials are bundled in the plugin.
- The skill instructs Codex to confirm with the user before submitting a paid
  registration order. Keep that behavior — it's the main safety property here.

## Also worth doing

- **Official OpenAI catalog:** <https://github.com/openai/plugins> is the curated
  first-party collection (`.agents/plugins/marketplace.json`). It has no open
  submission form; inclusion is at OpenAI's discretion.
- **Awesome list:** <https://github.com/hashgraph-online/awesome-codex-plugins>
  accepts PRs and is linked from the marketplace docs — cheap extra discovery.

## After approval

Users install with:

```bash
npx codex-marketplace add d3servelabs/namefi-codex-plugins/plugins/namefi --plugin --project
```

(`--plugin` needs the direct plugin path; use `--plugins` with the bare repo.)

Ship updates by bumping `version` in `plugins/namefi/.codex-plugin/plugin.json` and
pushing.
