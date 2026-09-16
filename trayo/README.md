# Trayo plugin

Use Trayo from Claude Code, Claude Cowork, Codex, and other clients that support remote MCP servers. The plugin bundles the Trayo MCP server with nine job skills: build an account list, research an account or a person, scan for a signal, monitor accounts, find stakeholders, enrich contacts, recent movers, and the core discovery loop.

Every client connects to `https://api.trayo.ai/v1/mcp` with a Trayo workspace API key from **Admin → API keys**.

## Claude Code

Add the public Trayo marketplace and install the plugin:

```bash
claude plugin marketplace add trayoai/trayo-plugin
claude plugin install trayo@trayo-plugins
```

Then:

1. Run `/plugin configure trayo@trayo-plugins`.
2. Enter the key in the masked field.
3. Start a new Claude Code session.
4. Run `/mcp`, then ask Claude to call `trayo_whoami`.

The `trayo` server should be connected with 23 tools. The key is stored as sensitive plugin configuration and is never part of the plugin or conversation.

### Claude Code with `--strict-mcp-config`

Claude Code's strict flag loads MCP servers only from `--mcp-config`. It ignores the server bundled
with the installed plugin, so `/plugin configure` by itself will not connect Trayo in that session.
Add Trayo to the JSON file you pass to `--mcp-config` (or merge this entry into your existing file):

```json
{
  "mcpServers": {
    "trayo": {
      "type": "http",
      "url": "https://api.trayo.ai/v1/mcp",
      "headers": { "X-API-Key": "${TRAYO_API_KEY}" },
      "alwaysLoad": true,
      "timeout": 120000
    }
  }
}
```

Set `TRAYO_API_KEY` for the Claude process through your secret setup, then run
`claude --strict-mcp-config --mcp-config /path/to/mcp.json`. The file contains only a variable
reference, not the key. In an interactive session, check `/mcp` and call `trayo_whoami`; in a
`-p --output-format stream-json` run, check the `system/init` event's `mcp_servers` and
`mcp_server_errors`. Do not pass the plugin's own `.mcp.json` as this file: its
`${user_config.api_key}` placeholder is for plugin configuration, not this variable-based
setup. If your existing strict config lists other servers, keep them in the same file.
For an interactive OAuth connection, omit `headers` and sign in through `/mcp` instead. If `/mcp`
shows Trayo as disabled, re-enable it there; the strict flag does not reset a disabled-server choice.

## Claude Cowork and Desktop

1. Open **Customize → Plugins**.
2. Select **+ → Add marketplace → Add from repository**.
3. Add `https://github.com/trayoai/trayo-plugin`.
4. Install **Trayo** and enter the key in its masked configuration field.
5. Start a new task and ask Claude to call `trayo_whoami`.

Your organization's plugin policy may require an administrator to approve the marketplace.

## Codex

Install the marketplace and skills, then register Trayo through Codex's native MCP configuration:

```bash
codex plugin marketplace add trayoai/trayo-plugin
codex plugin add trayo@trayo-plugins
codex mcp add trayo --url https://api.trayo.ai/v1/mcp --bearer-token-env-var TRAYO_API_KEY
```

Make `TRAYO_API_KEY` available through your existing secret setup. For the Codex app, you can add `TRAYO_API_KEY=<key>` to `~/.codex/.env` yourself, set that file to owner-only access with `chmod 600 ~/.codex/.env`, and restart Codex. Do not paste the key into an agent conversation.

Check the installation with:

```bash
codex plugin list --json
codex mcp get trayo --json
```

The plugin list should show Trayo version 0.5.3. The MCP result should show the fixed URL and `TRAYO_API_KEY` as its bearer token variable. Then ask Codex to call `trayo_whoami`.

## Other MCP clients

Add a streamable HTTP MCP server with:

- URL: `https://api.trayo.ai/v1/mcp`
- Header: `Authorization: Bearer <key>`

The nine job skills are included for clients that support this plugin marketplace format. Call `trayo_whoami` after setup to verify the connection and key.

## Your key

The key needs the scopes listed under Notes. Keep it in the client's masked secret field or secret file; never add it to this repository or paste it into an agent conversation.

## What you get

- Twenty-three tools, always loaded (no tool-search deferral): `trayo_whoami`, `trayo_get_workspace`, `trayo_set_workspace`, `trayo_import_accounts`, `trayo_list_signals`, `trayo_create_signal`, `trayo_run_discovery`, `trayo_get_discovery`, `trayo_list_events`, `trayo_find_companies`, `trayo_find_people`, `trayo_search_stakeholders`, `trayo_research_company`, `trayo_research_person`, `trayo_search_job_changes`, `trayo_add_to_list`, `trayo_list_lists`, `trayo_get_list_members`, `trayo_add_people`, `trayo_list_people`, `trayo_enrich_emails`, `trayo_enrich_phones`, `trayo_get_contacts`.
- Nine skills, invoked automatically when you describe the job: `/trayo:build-account-list`, `/trayo:research-account`, `/trayo:research-person`, `/trayo:scan-for-signal`, `/trayo:monitor-accounts`, `/trayo:find-stakeholders`, `/trayo:enrich-contacts`, `/trayo:recent-movers`, `/trayo:discover-signals`.

## What it does

- **Build accounts from criteria** — `/trayo:build-account-list`: `trayo_find_companies` → `trayo_import_accounts` → `trayo_add_to_list`.
- **Research an account, or a person** — `/trayo:research-account` (`trayo_research_company` + `trayo_search_stakeholders` + `trayo_list_events`) and `/trayo:research-person` (`trayo_research_person`, by professional profile or workspace person).
- **Read the signals found for an account** — `trayo_list_events` with `accountId` returns what discovery found. The events carry no significance or relevance score, so they come back unranked.
- **Scan a list of accounts for one signal** — `/trayo:scan-for-signal`: `trayo_create_signal` → `trayo_run_discovery` → `trayo_list_events`.
- **Run the whole loop end to end** — `/trayo:discover-signals`: import the companies, define the signal, run the discovery, read the events.
- **Keep watching a set of accounts** — `/trayo:monitor-accounts`: a 90-day backfill, then `trayo_list_events` with `discoveredSince` — what Trayo FOUND since you last looked — re-read on your own schedule. Set the stakeholder definition first with `trayo_set_workspace`, or no event ever comes back with people attached.
- **Watch people for job changes** — `/trayo:recent-movers`: `trayo_search_job_changes`.
- **Find the stakeholders at a company** — `/trayo:find-stakeholders`: `trayo_search_stakeholders` or `trayo_find_people` → `trayo_add_people` → `trayo_enrich_emails` → `trayo_add_to_list`. The chain ends with people you can write to, not with a search result.
- **Look up contact details** — `/trayo:enrich-contacts`: `trayo_list_people` or `trayo_add_people` → `trayo_enrich_emails` (or `trayo_enrich_phones`) → `trayo_get_contacts` → a list or a CSV. Every lookup draws on the workspace's lookup allowance; `trayo_whoami` reports what is left of it.

## What it does not do

Lookalike accounts. Market research. Ranking accounts by signal. Outbound, CRM push and routing — anything that acts on what you found. Alerts are polling, not push: monitoring re-reads on the schedule you run it on, nothing arrives on its own, and no schedule lives inside Trayo. Between your checks, Trayo scans your accounts on its own — `trayo_whoami` reports whether your workspace is covered at all. For an output beyond a list — a CSV, a team-chat digest, a push into your CRM — the skills will help you write a script against the REST API these tools wrap.

## Notes

- The key needs the scopes of the routes the tools wrap: `accounts:read`, `accounts:write`, `people:read`, `people:write`, `people:enrich_email`, `people:enrich_phone`, `settings:read`, `settings:write`, `events:read`, `events:write`, `research:trigger`. The two enrichment scopes are separate on purpose: a key can be allowed to find email addresses and refused phone numbers.
- Full API reference: `https://api.trayo.ai/v1/openapi.json`, with `https://api.trayo.ai/llms.txt` as the agent-facing summary.
