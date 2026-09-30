# OpenAI Plugins Directory submission

This repository now contains a portable Agent Plugins package at `plugins/namefi/`. It keeps the older Codex files for local marketplace installs. The portable package is a preparation draft; it has not been uploaded or submitted for review.

## Listing prepared in the package

- Intended publisher account: Namefi by D3Serve Labs Inc. The public developer name must match the identity selected and verified in the OpenAI portal.
- Name: Namefi. Subtitle: “Domains and DNS with Namefi” (27 characters).
- Category: Productivity. Starter prompts cover domain search, DNS records, and dynamic DNS.
- Website: <https://namefi.io/>.
- Privacy policy: <https://namefi.io/privacy>.
- Terms: <https://namefi.io/tos>.
- Commerce: paid registrations through Namefi tools after price confirmation, or checkout on Namefi's website.
- Publication targeting: all available countries (`countries: []`, which sets no country allowlist).
- Icon: `plugins/namefi/assets/namefi-icon.png`, a 512 × 512 PNG made from the existing Namefi mark.
- Review cases: five positive and three negative cases in `plugin.json`. These are drafted, not run.
- Release notes: included in `plugin.json`.

## Complete before public review

1. Publish [SUPPORT.md](./SUPPORT.md) as a public HTTPS support page and verify that it opens without signing in. Then put its final URL in `extensions.com.openai.interface.supportURL` in the portable manifest and `interface.supportURL` in the compatibility manifest. The provided support contact is `support@namefi.io`; an email address alone does not fill this URL field.
2. Select the Namefi by D3Serve Labs Inc organization in the OpenAI portal and confirm that its verified public developer identity matches both listing `developerName` fields. The organization ID belongs in portal setup, not the public package.
3. Prepare a dedicated reviewer account with at least one Namefi domain and a DNS zone. Put credentials, login instructions, and any fixture domain in the submission portal's secure reviewer-access fields, never in this package.
4. Run the five positive and three negative cases with the reviewer account. Compare the actual tool calls and results to the expectations in `plugin.json`; adjust any case that does not match intended behavior. No case in this draft has been marked passed.
5. Record the actual packaged plugin in ChatGPT or Codex using the reviewer account. Show domain availability and price, domain suggestions, owned domains and DNS records, then a boundary prompt such as asking it to create an email inbox. Keep tokens and private account details off screen. Host the recording where reviewers can view it without requesting access, verify playback, and add its URL as `extensions.com.openai.review.demo_recording_url`.
6. Update the hosted Namefi MCP server's tool annotations. Its current `registerTool` calls do not set `readOnlyHint`, `openWorldHint`, or `destructiveHint`; OpenAI's MCP review requires boolean values that match each tool's behavior. The server lives outside this plugin repository.
7. Validate the final package, upload it through the OpenAI Developer Portal with its existing HTTPS MCP endpoint, connect OAuth, inspect the imported metadata, and run the saved review cases. Domain and developer verification, scans, legal attestations, and submission for review are portal steps for the authorized publisher.

The published [OpenAI package format](https://developers.openai.com/plugins/build/plugins) and [submission requirements](https://developers.openai.com/plugins/deploy/submission) are the references for this checklist. The old community Codex Plugin Marketplace process in this file was separate from OpenAI's public Plugins Directory.
