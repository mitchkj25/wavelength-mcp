# Wavelength MCP server

The official remote MCP server for [Wavelength](https://wavelengthflow.com). Turn calls, notes, and code into editable process flow diagrams, then send them to the tools your team uses.

- **URL:** `https://wavelengthflow.com/api/mcp`
- **Transport:** Streamable HTTP
- **Auth:** OAuth 2.1 with PKCE. Dynamic client registration and Client ID Metadata Documents are supported. Sign in with your Wavelength account. A free account works.
- **Registry name:** `com.wavelengthflow/wavelength` in the [official MCP Registry](https://registry.modelcontextprotocol.io)

This repository holds the public listing and setup docs. The server is hosted by Wavelength, so there is nothing to install or run.

## What it can do

- Create a process flow from a description, call notes, a transcript, or an exact graph.
- Find, open, edit, duplicate, rename, and delete saved flows. Compare and restore versions.
- Summarize a flow, review it for gaps, write call recaps, and suggest follow-up questions.
- Export a flow as Mermaid, SVG, draw.io, or JSON. Import Mermaid, draw.io, and SVG diagrams, or files from Google Drive.
- Send a flow to Jira, Confluence, Notion, Salesforce, ServiceNow, HubSpot deals, and Google Drive.
- Post a flow to Slack, Microsoft Teams, Google Chat, Discord, or webhooks for Zapier, Make, and n8n.
- Manage folders, teams, organizations, collaborators, share links, client portals, and live sessions.

Tools that return a flow also render it as an interactive diagram in clients that support MCP Apps.

## Connect

### Claude

Settings, Connectors, Add custom connector. Paste `https://wavelengthflow.com/api/mcp`.

### Claude Code

```bash
claude mcp add --transport http wavelength https://wavelengthflow.com/api/mcp
```

### ChatGPT

Add the URL as an app in Developer mode. The Wavelength listing in the ChatGPT directory is in review.

### Cursor

Install the [Wavelength plugin](https://github.com/mitchkj25/wavelength-cursor-plugin), or add to `mcp.json`:

```json
{ "mcpServers": { "wavelength": { "url": "https://wavelengthflow.com/api/mcp" } } }
```

### VS Code

```json
{ "servers": { "wavelength": { "type": "http", "url": "https://wavelengthflow.com/api/mcp" } } }
```

### Gemini CLI

```bash
gemini extensions install https://github.com/mitchkj25/wavelength-gemini-extension
```

### n8n

Install the community node [`n8n-nodes-wavelength`](https://www.npmjs.com/package/n8n-nodes-wavelength).

## Privacy and security

Each tool acts only as the signed-in user. Turn on Read-only mode in Wavelength Settings, then AI connector, to block all writes. Clients that read code send only the flow (node and edge text) to Wavelength.

- Privacy policy: https://wavelengthflow.com/privacy.html
- Security: https://wavelengthflow.com/security.html
- Support: support@wavelengthflow.com

## License

The docs in this repository are MIT licensed. The Wavelength service is subject to the [Wavelength terms](https://wavelengthflow.com/terms.html).
