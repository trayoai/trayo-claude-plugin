# Trayo plugin

The official Trayo plugin gives AI coding and work agents access to Trayo's company and people search, account research, contact enrichment, signals, discovery, and events. It includes nine skills that turn those tools into common GTM workflows.

The packaged plugin uses a workspace API key from **Admin → API keys**. For OAuth, connect your MCP client directly to `https://api.trayo.ai/v1/mcp` and sign in to Trayo.

## Claude Code

```bash
claude plugin marketplace add trayoai/trayo-plugin
claude plugin install trayo@trayo-plugins
```

Run `/plugin configure trayo@trayo-plugins` and enter the key in the masked field.

If you start Claude Code with `--strict-mcp-config`, its plugin MCP server is excluded. Add Trayo
to the file passed with `--mcp-config` as shown in [the plugin guide](trayo/README.md).

## Codex

```bash
codex plugin marketplace add trayoai/trayo-plugin
codex plugin add trayo@trayo-plugins
codex mcp add trayo --url https://api.trayo.ai/v1/mcp --bearer-token-env-var TRAYO_API_KEY
```

Make `TRAYO_API_KEY` available to Codex through your existing secret setup or `~/.codex/.env`, then restart Codex.

Claude Cowork and other MCP clients are also supported. After setup, call `trayo_whoami`; a successful response confirms the connection and key.

See [the plugin guide](trayo/README.md) for complete setup steps, the available skills, required key scopes, and current limitations.
