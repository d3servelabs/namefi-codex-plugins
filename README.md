# Namefi plugin for ChatGPT and Codex

Search, register, and manage domains and DNS through Namefi's hosted MCP server. The plugin also includes guided workflows for dynamic DNS and serving a local service over HTTPS.

## Local installation

This repository includes a Codex marketplace for development and local use:

```bash
codex plugin marketplace add d3servelabs/namefi-codex-plugins
codex plugin add namefi@namefi
```

The marketplace entry requires sign-in when the plugin is installed. Every Namefi MCP request, including availability searches, requires authentication. The server supports OAuth; for headless use, generate an API key at <https://namefi.io/api-key> and configure it as an `x-api-key` header in your client. No credential is included in this repository.

The public OpenAI Plugins Directory uses a separate submission and review process. This repository is not yet a published directory listing; see [SUBMISSION.md](./SUBMISSION.md) for its preparation status.

## Package layout

```text
.agents/plugins/marketplace.json  Local Codex marketplace entry
plugins/namefi/
  plugin.json                Portable Agent Plugins manifest and OpenAI metadata
  mcp.json                   Streamable HTTP connection to Namefi
  .codex-plugin/plugin.json  Legacy Codex compatibility manifest
  .mcp.json                  Legacy Codex MCP configuration
  assets/                    Listing icon and original brand assets
  skills/                    Domain, dynamic DNS, and HTTPS workflows
```

The MCP server provides domain availability and pricing, registration and order tracking, DNS records, domain settings, and other Namefi API operations. The plugin's skills guide how to use those operations and how to set up a domain for a machine without a static IP. Before a paid registration, the domain skill requires confirmation of the exact name, duration, and price.

## Related projects

The [Claude Code plugin](https://github.com/d3servelabs/namefi-claude-plugins) uses the same hosted Namefi MCP server with platform-specific packaging.

## License

MIT
