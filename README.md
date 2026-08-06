# Namefi plugin for Codex

Search, register, and manage domains and their DNS from OpenAI Codex.

## Install

```bash
npx codex-marketplace add d3servelabs/namefi-codex-plugins/plugins/namefi --plugin --project
```

Use `--global` instead of `--project` to install for your user rather than one repo.
Omit both and the CLI prompts for scope.

The singular `--plugin` flag requires the direct repository path shown above. The
plural form crawls the whole `plugins/` folder:

```bash
npx codex-marketplace add d3servelabs/namefi-codex-plugins --plugins --project
```

Remove with `npx codex-marketplace remove namefi --project`.

## What's in the plugin

| Component | What it does |
| --- | --- |
| MCP server `namefi` | The hosted Namefi MCP server (`https://api.namefi.io/mcp`) — availability search, registration + order polling, DNS records, domain config, outbound lead-finding |
| Skill `namefi-domains` | Teaches Codex the MCP-first flow: confirm before spending, register → poll order, cart hand-off for human checkout, and the REST fallback table |

## Authentication

Every Namefi call is authenticated, including availability search — there is no
anonymous access.

The bundled `.mcp.json` declares an `oauth_resource`, and the marketplace entry sets
`"authentication": "ON_INSTALL"`, so Codex prompts for sign-in at install time. The
server implements OAuth 2.1 + PKCE with dynamic client registration, so no
pre-registered secret is needed.

For headless or CI use, generate a key at <https://namefi.io/api-key> and add an
`x-api-key` header to the server entry in your Codex config. No credential is
bundled in the plugin.

## Layout

```
.agents/plugins/marketplace.json   # repo-level catalog (see note below)
plugins/namefi/
├── .codex-plugin/plugin.json      # required manifest
├── .mcp.json                      # bundled Namefi MCP server
├── skills/namefi-domains/SKILL.md # the skill
└── assets/                        # composer icon + logo
SUBMISSION.md                       # marketplace submission runbook
```

Only `plugin.json` goes inside `.codex-plugin/` — everything else sits at the
plugin root.

**On `.agents/plugins/marketplace.json`:** the installer writes its *own* copy of
this file into the target project (or `~/.agents/`) at install time, naming the
marketplace after the destination directory. The one in this repo is the
repo-level catalog the docs call for — it makes the repo self-describing for
`--plugins` crawls and review — but installing users never consume it directly.
Verified by installing the official `openai/plugins` notion plugin and diffing
what the CLI generated.

## Submitting

See [SUBMISSION.md](./SUBMISSION.md).

## Related

The Claude Code version of this plugin lives at
[d3servelabs/namefi-claude-plugins](https://github.com/d3servelabs/namefi-claude-plugins).
Both wrap the same hosted MCP server; the manifests and skill frontmatter differ per
platform.

## License

MIT
