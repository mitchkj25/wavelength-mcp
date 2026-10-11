# Wavelength API reference

Wavelength exposes one API: an MCP (Model Context Protocol) server. Every integration (Zapier, Make, n8n, Claude, ChatGPT, Cursor, Gemini CLI) uses this same API.

## Endpoints

| Purpose | URL |
|---|---|
| API (JSON-RPC 2.0 over HTTP) | `https://wavelengthflow.com/api/mcp` |
| OAuth authorization | `https://mcp.wavelengthflow.com/authorize` |
| OAuth token and refresh | `https://mcp.wavelengthflow.com/token` |
| OAuth server metadata | `https://mcp.wavelengthflow.com/.well-known/oauth-authorization-server` |

All endpoints are production endpoints. Wavelength owns and operates every domain above.

## Authentication

Wavelength uses OAuth 2.0 authorization code flow with PKCE (S256). PKCE is required.

1. Send the user to `/authorize` with `response_type=code`, `client_id`, `redirect_uri`, `scope=flows`, `state`, `code_challenge`, and `code_challenge_method=S256`.
2. The user signs in to Wavelength and approves access.
3. Wavelength redirects to `redirect_uri` with `code` and `state`.
4. POST to `/token` as `application/x-www-form-urlencoded` with `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, and `code_verifier`.
5. The response has `access_token`, `token_type` (Bearer), `expires_in`, and `refresh_token`.
6. To refresh, POST to `/token` with `grant_type=refresh_token`, `refresh_token`, and `client_id`.

Send the access token on every API call: `Authorization: Bearer <access_token>`. An expired or revoked token returns HTTP 401.
MCP clients that support dynamic client registration can register at the `registration_endpoint` in the server metadata.

## Calling a tool

POST a JSON-RPC 2.0 request to `https://wavelengthflow.com/api/mcp` with headers `Content-Type: application/json` and `Accept: application/json, text/event-stream`.

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"list_flows","arguments":{"limit":10}}}
```

The response has `result.structuredContent` (a JSON object) and `result.content` (text). When a tool fails, `result.isError` is `true` and `result.content[0].text` has the reason.

Other methods: `initialize` and `tools/list` (returns every tool with its JSON Schema).

```json
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
```

## Limits

Plan limits apply to API calls the same way as in the app (for example, the Free plan saves up to 5 flows). The API has no extra charge.

## Tools (85)

### show_flow

Opens a saved flow as an interactive diagram in MCP clients that render UI. Takes `flow_id`.

### create_flow_from_text

Turn a transcript, call notes, or a spoken description of a process into a saved Wavelength flow. Nodes are auto-laid out. Returns the new flow id, title, node and edge counts, a short list of node labels, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `text` | string | yes | A transcript, call notes, or a spoken description of the process to diagram. |
| `title` | string | no | Optional title for the flow. If omitted, the first line of the text is used. |
| `direction` | string (vertical, horizontal) | no | Optional layout direction. vertical stacks ranks top-down (default); horizontal advances left-to-right. |
| `folder_id` | string | no | Optional folder id to create the flow in. |
| `team_id` | string | no | Optional team id to assign the flow to. |
| `org_id` | string | no | Optional organization id (from list_organizations). The flow is saved in that organization, where every member can open it. |

### list_flows

List the signed-in user's saved Wavelength flows, newest first. Optionally filter by a case-insensitive title query or a folder id, and limit the count. Returns per flow: id, title, updated_at, folder_id, is_public, shared (true when the user is not the owner), team, role (owner, team, editor, or viewer), and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Optional case-insensitive text to match against the flow title. |
| `folder_id` | string | no | Optional folder id to filter to flows in that folder. |
| `limit` | integer | no | Maximum number of flows to return, 1 to 100. Defaults to 20. |

### search_flows

Full-text search the signed-in user's saved Wavelength flows by title, node labels, and transcript. Returns matches with where they matched, a transcript snippet, and a link to open each. Query must be at least 2 characters.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | yes | The search query, at least 2 characters. |
| `limit` | integer | no | Maximum number of matches to return, 1 to 100. Defaults to 20. |

### get_flow

Read one saved Wavelength flow by id. Returns the title, folder_id, is_owner, my_role, owner, is_public, public_url, every node with its id label type owner and note, every edge with source and target labels, a Mermaid flowchart string of the graph, and a link to open it. By default the transcript is omitted and transcript_chars (its length) is returned instead; pass include_transcript true to include the full transcript. A node includes status (warning or error) when it has conditional formatting.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to read. |
| `include_transcript` | boolean | no | Optional. When true, include the full transcript. When false (default), omit it and return transcript_chars instead. |

### edit_flow

Apply a plain-language edit instruction to an existing Wavelength flow, for example 'add Snowflake after Fivetran'. Saves a versioned snapshot labeled 'Edited by AI agent'. Returns the new node and edge counts and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `instruction` | string | yes | A plain-language edit instruction, for example 'add Snowflake after Fivetran'. |

### add_nodes

Add nodes to a Wavelength flow. Node types: start, step, decision, data, system, actor, end. status (warning, error, or none) sets conditional formatting. New nodes are auto-placed. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### update_nodes

Change the label, type, owner, note, or status of existing nodes in a Wavelength flow, by node id. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### remove_nodes

Remove nodes from a Wavelength flow, by node id. Each removed node also loses its edges. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### add_edges

Connect nodes in a Wavelength flow. Edge kinds: sequence, data, conditional, reports_to. An edge with a missing source or target is rejected in the result. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### update_edges

Change the label or kind of existing edges in a Wavelength flow, by edge id. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### remove_edges

Remove edges from a Wavelength flow, by edge id. No AI call. Saves a versioned snapshot labeled 'Edited by AI agent (exact)'. Returns the applied count, a rejected list with reasons, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to edit. |
| `items` | array | yes | The items to apply, in order. |

### review_flow

Run a deterministic gap check on a Wavelength flow, with no AI call. Returns orphan nodes (no edges in or out), dead ends (non-end nodes with no outgoing edge), single-branch decisions, unlabeled decision branches, steps without an owner, AI-guessed edges, missing_start and missing_end flags, and a list of plain questions an SE can ask on the next call, one per finding.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to review. |

### summarize_flow

Summarize a Wavelength flow. systems_in_scope is deterministic: it groups system and data nodes with their inbound and outbound edge labels, plus actors. recap_email and sow_outline call the AI, writing only from the graph and transcript (no invented facts, no em dashes, short sentences). Returns {format, text}.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to summarize. |
| `format` | string (recap_email, sow_outline, systems_in_scope) | yes | The summary format. |
| `audience` | string | no | Optional audience for recap_email and sow_outline, for example 'the client CFO'. |

### compare_versions

Compare two saved versions of a Wavelength flow, or one version against the current graph. Nodes are matched by id; edges by source, target, and kind. Returns added, removed, and changed nodes (with the fields that changed), added and removed edges (with labels), and a one line summary.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `version_a` | string | yes | The id of the first version to compare. |
| `version_b` | string | no | Optional id of the second version. Omit to compare against the current graph. |

### get_account

Return the signed-in user's current plan, saved flow count, trial end date, flows created this month, folder limit and personal folder count, and mic time usage.

No parameters.

### delete_flow

Permanently delete a saved Wavelength flow by id. This cannot be undone. Only the owner can delete a flow. Returns {deleted: true, flow_id} on success.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to delete. |

### share_flow

Create or refresh a public read-only link anyone can open for a Wavelength flow, with no account needed. Optionally include a summary shown on the public page. hide_attribution is Pro only and the server enforces it. Returns the public url, slug, and is_public flag. Also returns embed_image_url, a live SVG image of the flow that updates when the flow changes, and embed_markdown, ready to paste into Notion, Confluence, GitHub, or any Markdown page.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to share. |
| `summary` | string | no | Optional summary shown on the public page, up to 300 characters. |
| `hide_attribution` | boolean | no | Optional. Hide the Wavelength attribution on the public page. Pro only; the server enforces the plan limit. |

### unshare_flow

Turn off the public read-only link for a Wavelength flow. The link stops working for everyone who has it. Returns is_public false.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to unshare. |

### export_flow

Export a saved Wavelength flow in the requested format. mermaid returns a Mermaid flowchart string. json returns the title, graph_nodes, and graph_edges. svg returns a self-contained SVG with node shapes, colors, and arrows. drawio returns a valid mxfile/mxGraphModel XML for draw.io. SVG and drawio use stored node positions, or auto-layout when the flow has none. No flow content is sent to any third-party service.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to export. |
| `format` | string (mermaid, json, svg, drawio) | yes | The export format: mermaid, json, svg, or drawio. |

### rename_flow

Change the title of a saved Wavelength flow. Only the owner can rename a flow. Returns the new title and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to rename. |
| `title` | string | yes | The new title. |

### duplicate_flow

Copy a saved Wavelength flow into a new flow with the same transcript and graph. The new title defaults to '<old title> (copy)'. Plan flow limits apply. Returns the new flow id, title, node and edge counts, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to copy. |
| `title` | string | no | Optional title for the copy. Defaults to '<old title> (copy)'. |

### create_flow_from_graph

Create a saved Wavelength flow from an explicit node and edge structure, with no AI call. Use this when the client already has the graph. Node types must be one of start, step, decision, data, system, actor, end. Edge kinds must be one of sequence, data, conditional, reports_to. Edges that point at a missing node are dropped. Nodes are auto-laid out. Returns the new flow id, node and edge counts, and the number of dropped edges.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `title` | string | yes | The title for the new flow. |
| `nodes` | array | yes | The nodes in the graph. |
| `edges` | array | no | The edges in the graph. |
| `direction` | string (vertical, horizontal) | no | Optional layout direction. vertical stacks ranks top-down (default); horizontal advances left-to-right. |
| `folder_id` | string | no | Optional folder id to create the flow in. |
| `team_id` | string | no | Optional team id to assign the flow to. |

### import_diagram

Turn an existing diagram into an editable Wavelength flow, with no AI call. Pass the diagram source as text: a Mermaid flowchart, draw.io or diagrams.net XML (plain or compressed), or an SVG (for example an SVG diagram you drew earlier in this chat). Shapes become nodes, connectors become edges with their labels, diamonds become decisions. Exact positions and colors are not kept; the flow is laid out again in process order. Returns the new flow id, detected format, node and edge counts, the first node labels, and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content` | string | yes | The diagram source text: Mermaid, draw.io XML, or SVG markup. |
| `format` | string (mermaid, drawio, svg) | no | Optional. Detected from the content when omitted. |
| `title` | string | no | Optional title. Defaults to the first node label. |
| `direction` | string (vertical, horizontal) | no | Optional layout direction. Default vertical. |
| `folder_id` | string | no | Optional folder id to create the flow in. |
| `team_id` | string | no | Optional team id to assign the flow to. |

### add_to_flow

Add new notes or transcript from a follow-up call to an existing flow. Gets the flow, asks the AI to merge the new notes into the graph reusing existing nodes, saves a versioned snapshot labeled 'Added notes by AI agent', and appends the text to the stored transcript. Returns the new node and edge counts and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to add to. |
| `text` | string | yes | The new notes or transcript to merge into the flow. |

### list_versions

List the saved version snapshots for a Wavelength flow, newest first. Returns per version: version_id, label, and created_at.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |

### restore_version

Restore a saved version of a Wavelength flow. First saves a snapshot of the current graph labeled 'Before restore by AI agent', then restores the chosen version. This overwrites the current graph. Returns the flow id and a link to open it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `version_id` | string | yes | The id of the version to restore. |

### list_folders

List the signed-in user's folders. Returns per folder: id, name, team_id, and flow_count. Also returns the folder limit for the plan, the personal folder count, and whether the user can create folders and team folders.

No parameters.

### create_folder

Create a new folder. Personal folders need a paid plan. Pass team_id for a team folder, or org_id (from list_organizations) for an organization folder that every member of the organization can use. Only organization flows can go in an organization folder. Returns the new folder.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | The folder name. |
| `team_id` | string | no | Optional team id to create a team folder. |
| `org_id` | string | no | Optional organization id. Creates an organization folder. |

### rename_folder

Rename a folder. Returns the folder id and the new name.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `folder_id` | string | yes | The id of the folder to rename. |
| `name` | string | yes | The new folder name. |

### delete_folder

Delete a folder. Flows inside the folder are moved back to unfiled; they are not deleted. Returns {deleted: true}.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `folder_id` | string | yes | The id of the folder to delete. |

### move_flow_to_folder

Move a flow into a folder. Pass folder_id null or empty to unfile it. Returns the flow id and the new folder_id.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to move. |
| `folder_id` | string/null | no | The destination folder id. Pass null or an empty string to remove the flow from its folder. |

### get_share_status

Read the public sharing status for a Wavelength flow. Returns is_public, url, summary, views, published_at, and hide_attribution.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |

### list_collaborators

List the collaborators on a Wavelength flow. Returns the owner, whether the signed-in user is the owner, and each collaborator's email and role (editor or viewer). Never returns the share token.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |

### add_collaborator

Invite a collaborator to a Wavelength flow by email, with role editor or viewer (default editor). Collaboration needs the Pro plan. Calling add_collaborator again for the same email with a new role changes the collaborator's role. Returns the new collaborator.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `email` | string | yes | The email of the person to invite. |
| `role` | string (editor, viewer) | no | The collaborator role. Defaults to editor. Calling again with a new role changes the role. |

### remove_collaborator

Remove a collaborator from a Wavelength flow by email. Only the owner can remove collaborators. Returns the removed email.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `email` | string | yes | The email of the collaborator to remove. |

### list_teams

List teams. Without org_id: the teams the signed-in user belongs to, with id, name, owner, is_owner, org_id, my_role, member_count, and member emails. With org_id: every team in that organization, with each member's team role (team_admin, editor, or viewer).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | no | Optional organization id (from list_organizations) to list every team in that organization. |

### move_flow_to_team

Assign a flow to a team so every team member can see and edit it. Pass team_id null to remove it from the team. Returns the flow id and the new team_id.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to move. |
| `team_id` | string/null | no | The destination team id. Pass null or an empty string to remove the flow from its team. |

### crm_status

Read the connected CRM (HubSpot) status for the signed-in user. Returns whether HubSpot is connected and the provider.

No parameters.

### search_crm_deals

Search the connected HubSpot for deals by name. With no query, returns the most recently changed deals. If HubSpot is not connected, returns an error telling the user to connect HubSpot in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Optional deal name search query. Omit to return the most recently changed deals. |

### attach_flow_to_crm

Attach a Wavelength flow to a HubSpot deal as a note, with the flow title, an optional summary, and a share link. If the flow is public, the public link is used; otherwise the open_url is used. If HubSpot is not connected, returns an error telling the user to connect HubSpot in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to attach. |
| `deal_id` | string | yes | The HubSpot deal id to attach the flow to. |
| `summary` | string | no | Optional summary to include in the note, up to 1000 characters. |

### ask_flow

Ask a question about one saved Wavelength flow. The answer comes only from the steps, owners, systems, and links in the flow. If the flow does not say, the answer says so and names what is missing. Uses one AI help call from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `question` | string | yes | The question about the flow, up to 1000 characters. |

### suggest_followups

Find the gaps in a flow and write 3 to 6 follow-up questions to ask the customer. It looks for steps with no owner, a missing trigger or end, missing decision branches, links that AI guessed, and systems with no clear input or output. It uses the saved call transcript of the flow when there is one. Each question has the step it is about and a kind. Uses one AI help call from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `use_transcript` | boolean | no | Set false to ignore the saved transcript. Default true. |

### write_recaps

Write three short recaps of one flow: customer_recap (an email body to the customer after the call), handoff (notes for the solution engineer or implementation team), and leadership (2 to 4 sentences for a manager). It uses only facts in the flow and its saved transcript. Uses one AI help call from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `use_transcript` | boolean | no | Set false to ignore the saved transcript. Default true. |

### group_notes

Group a list of notes (for example sticky notes from a workshop) by theme. Every note goes in exactly one group, and each group gets a short title. Nothing is saved. Uses one AI help call from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `notes` | array | yes | The notes, one short note per item (for example sticky notes from a workshop). Up to 300 characters each. |
| `title` | string | no | Optional name of the board or workshop. |

### notes_to_flow

Turn a list of notes (for example sticky notes from a workshop) into a process flow and save it as a new flow. Only facts in the notes are used. Notes that are not process facts (questions, opinions, risks) are left out. The notes are saved as the transcript of the flow. Uses one AI help call from the plan. Returns the new flow_id and open_url.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `notes` | array | yes | The notes, one short note per item (for example sticky notes from a workshop). Up to 300 characters each. |
| `title` | string | yes | The title of the new flow. |
| `folder_id` | string | no | Optional folder id from list_folders. |
| `team_id` | string | no | Optional team id from list_teams. |
| `direction` | string (vertical, horizontal) | no | Layout direction. Default vertical. |

### format_flow

Mark steps of a flow yellow (warning) or red (error) from a plain-language rule, or clear the marks. Examples: "flag every step that waits more than a day", "mark steps with no owner red", "clear all highlights". Only the step status changes. Labels, links, and positions stay the same. A version is saved first, so restore_version can undo it. Uses one AI build or edit from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `rule` | string | yes | The formatting rule in plain language, up to 1000 characters. |

### get_insights

Find patterns across every flow the user owns or can edit (up to 200 flows): the systems and people that appear most, steps that repeat across flows, flagged steps, the slowest steps, flows with links that AI guessed, and flows due for a check-in. Set use_ai to true to also get AI patterns with the flows as evidence and questions for the next calls. use_ai needs two or more flows and uses one AI insight run from the plan.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `use_ai` | boolean | no | Also find AI patterns. Default false. |

### get_checkin_status

Read the check-in status of one flow: when someone last confirmed it still matches the work, the check-in interval in days, the due date, and whether it is stale. Wavelength emails the owner when a flow is due.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |

### update_checkin

Confirm that a flow still matches the work today (confirm true), or set how often Wavelength asks the owner to check the flow (interval_days), or both. Only the owner can change the interval. interval_days 0 turns check-ins off.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `confirm` | boolean | no | True marks the flow as still right today. |
| `interval_days` | integer (0, 7, 14, 30, 60, 90) | no | Days between check-ins. 0 turns them off. |

### list_portals

List the client portals of the user. A portal is a public page at wavelengthflow.com/p/... that shows a set of flows to a client, who can comment and vote on steps without an account. Returns each portal id, name, url, flow ids, and settings, plus the plan limit.

No parameters.

### save_portal

Create a client portal, or update one when portal_id is given. A portal is a public page that anyone with the link can open. Only flows that the user owns can go on a portal. Other flow ids are skipped. On an update, fields that are not given keep their current value. The plan limits the number of portals.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `portal_id` | string | no | The portal to update. Leave out to create a new portal. |
| `name` | string | no | The portal name, up to 80 characters. Required for a new portal. |
| `intro` | string | no | A short intro shown at the top, up to 600 characters. |
| `flow_ids` | array | no | The flows to show, in order. This replaces the current list. |
| `accent` | string (#7B5CFF, #E1509E, #FF8A5C, #4D8CFE, #16a34a, #0f172a) | no | The accent color. Default #7B5CFF. |
| `allow_comments` | boolean | no | Let visitors comment. Default true. |
| `allow_votes` | boolean | no | Let visitors vote on steps. Default true. |

### delete_portal

Delete a client portal. The public portal link stops working for everyone. The flows on it are not deleted.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `portal_id` | string | yes | The portal id from list_portals. |

### list_connections

Show which outside apps the user has connected to Wavelength (HubSpot, Google Drive, Notion, Confluence, Jira, Salesforce, ServiceNow), the connected account, and the tool that sends a flow to each one. Also shows how many chat channels and webhooks are set up. Connecting an app needs a browser sign-in, so this tool tells the user where to connect it in Wavelength settings.

No parameters.

### list_organizations

List the organizations the user belongs to, with the user's role (owner, admin, or member), the member count, and the Pro seat count. Use org_id with create_flow_from_text or move_flow_to_organization.

No parameters.

### create_organization_team

Create a team inside an organization. Only an organization owner or admin can do this. Then add people with add_team_member. Only members of the organization can join its teams.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `name` | string | yes | The team name. |

### add_team_member

Add one person to a team with a role. Roles: team_admin (manages the team), editor (edits team flows), viewer (opens team flows only). Only a team admin, the team owner, or an organization owner or admin can do this. For an organization team, the person must be in the organization.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `team_id` | string | yes | Team id from list_teams. |
| `email` | string | yes | The person's email address. |
| `role` | string (team_admin, editor, viewer) | no | Team role for add or set_role. Default editor. |

### set_team_member_role

Change the role of one person on a team. Roles: team_admin (manages the team), editor (edits team flows), viewer (opens team flows only). Only a team admin, the team owner, or an organization owner or admin can do this.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `team_id` | string | yes | Team id from list_teams. |
| `email` | string | yes | The person's email address. |
| `role` | string (team_admin, editor, viewer) | yes | Team role for add or set_role. Default editor. |

### remove_team_member

Take one person off a team. They lose access to the team flows. Only a team admin, the team owner, or an organization owner or admin can do this.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `team_id` | string | yes | Team id from list_teams. |
| `email` | string | yes | The person's email address. |

### move_flow_to_organization

Move a flow into an organization so every member can open it, or out of it (org_id empty) so it is personal again. The flow owner can move a flow in. Only an organization owner or admin can move a flow out.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The flow id. |
| `org_id` | string | no | The organization id. Leave it out to move the flow out of its organization. |

### list_channels

List the chat channels and webhooks the user set up in Wavelength: Slack, Microsoft Teams, Google Chat, Discord, and generic webhooks (Zapier, Make, n8n). Returns each channel_id, label, kind, and last delivery status. Use the channel_id with send_flow_to_channel.

No parameters.

### send_flow_to_channel

Post a flow to a Slack, Microsoft Teams, Google Chat, or Discord channel, or to a webhook. The message has the flow title, a short summary, questions to confirm, and a link to the flow. A webhook also gets the steps, links, and Mermaid code as JSON. The link is the public link when the flow is shared, otherwise the private app link. Get the channel_id from list_channels.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `channel_id` | string | yes | The channel id from list_channels. |
| `summary_lines` | array | no | Optional summary lines to send instead of the default summary. |

### find_notion_pages

Search the Notion pages that the user shared with Wavelength. Use a page_id as the parent for send_flow_to_notion. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Words in the page title. Leave empty for recent pages. |

### send_flow_to_notion

Save a flow as a new Notion page under a parent page. The page has a link to the live flow, a summary, questions to confirm, and the Mermaid diagram code. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `parent_page_id` | string | yes | The parent page id from find_notion_pages. |
| `title` | string | no | Optional page title. Default is the flow title. |
| `summary` | string | no | Optional summary to use instead of the default, one point per line. |

### find_confluence_targets

Search the connected Confluence site for spaces and pages. Use a space_key with send_flow_to_confluence to make a new page, or a page_id to add the flow to an existing page. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Words in the space name or page title. |
| `space_key` | string | no | Optional space key to search pages in one space. |

### send_flow_to_confluence

Make a new Confluence page from a flow (give space_key, and optionally parent_page_id), or add the flow to the end of an existing page (give page_id). The text has a summary, questions to confirm, and a link to the live flow. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `space_key` | string | no | The space for a new page. |
| `parent_page_id` | string | no | Optional parent page for a new page. |
| `page_id` | string | no | An existing page to add the flow to. |
| `title` | string | no | Optional title. Default is the flow title. |
| `summary` | string | no | Optional text to send instead of the default summary, up to 15000 characters. For example, the handoff text from write_recaps. The flow link is added at the end. |

### find_jira_targets

Search the connected Jira site for projects and issues. Give project_id to also get the issue types of that project. Use project_id and issue_type_id with send_flow_to_jira to make a new issue, or issue_key to comment on an existing issue. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Words in the project name or issue summary, or an issue key such as ABC-12. |
| `project_id` | string | no | Optional project id to list its issue types. |

### send_flow_to_jira

Make a new Jira issue from a flow (give project_id and issue_type_id), or add the flow as a comment on an existing issue (give issue_key). The text has a summary, questions to confirm, and a link to the live flow. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `project_id` | string | no | The project for a new issue. |
| `issue_type_id` | string | no | The issue type for a new issue. |
| `issue_key` | string | no | An existing issue to comment on, such as ABC-12. |
| `title` | string | no | Optional issue summary. Default is the flow title. |
| `summary` | string | no | Optional text to send instead of the default summary, up to 15000 characters. For example, the handoff text from write_recaps. The flow link is added at the end. |

### find_salesforce_records

Search the connected Salesforce org for opportunities or accounts by name. With no query, returns the most recently changed records. Use a record_id with send_flow_to_salesforce. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | no | Words in the record name. |
| `type` | string (Opportunity, Account) | no | The record type. Default Opportunity. |

### send_flow_to_salesforce

Attach a flow to a Salesforce opportunity or account as a note with a summary, questions to confirm, and a link to the live flow. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `record_id` | string | yes | The record id from find_salesforce_records. |
| `title` | string | no | Optional note title. Default is the flow title. |
| `summary` | string | no | Optional text to send instead of the default summary, up to 15000 characters. For example, the handoff text from write_recaps. The flow link is added at the end. |

### find_servicenow_records

Search the connected ServiceNow instance for incidents, change requests, problems, or requested items by number or description. Use a sys_id with send_flow_to_servicenow. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `table` | string (incident, change_request, problem, sc_req_item) | no | The record type. Default incident. |
| `query` | string | no | A record number or words in the short description. |

### send_flow_to_servicenow

Add a flow as a work note on an existing ServiceNow record (give sys_id), or make a new record from the flow (leave out sys_id). The text has a summary, questions to confirm, and a link to the live flow. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `table` | string (incident, change_request, problem, sc_req_item) | yes | The record type. |
| `sys_id` | string | no | An existing record from find_servicenow_records. |
| `title` | string | no | Optional title for a new record. Default is the flow title. |
| `summary` | string | no | Optional text to send instead of the default summary, up to 15000 characters. For example, the handoff text from write_recaps. The flow link is added at the end. |

### save_flow_to_drive

Save a flow as a file in the Wavelength folder of the connected Google Drive. google_doc makes a Google Doc with a summary, the steps and owners, questions to confirm, a link to the live flow, and the Mermaid code. drawio, svg, mermaid, and json save the diagram in that format. Returns the Drive link. If the app is not connected, the error tells the user where to connect it in Wavelength settings.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow. |
| `format` | string (google_doc, drawio, svg, mermaid, json) | no | The file format. Default google_doc. |

### get_organization

Show one organization: members and their roles, Pro seats, seats in use, pending invitations, settings, and the flow count. Only an owner or an admin sees pending invitations.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |

### create_organization

Create a new organization. You become its owner. To invite people, the owner buys Pro seats in Wavelength Settings, Organization, Billing. You can own up to 5 organizations.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | The organization name. |

### update_organization

Rename an organization or change its settings. Only an owner or an admin can do this. Settings: member_flow_role (editor or viewer on organization flows), allow_member_invites, allow_public_publish.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `name` | string | no | New organization name. |
| `member_flow_role` | string (editor, viewer) | no | What members can do on organization flows. |
| `allow_member_invites` | boolean | no | true lets members invite other members. |
| `allow_public_publish` | boolean | no | true lets members publish organization flows to a public link. |

### invite_member

Invite one person to an organization by email, as member or admin. Wavelength emails an invitation that works only for that address and expires in 7 days. Each member and each pending invitation uses one Pro seat. If no seat is free, the error says so; the owner adds seats in Billing.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `email` | string | yes | The person's email address. |
| `role` | string (member, admin) | no | Role after they accept. Default member. |

### revoke_invite

Cancel a pending invitation so its link stops working. Only an owner or an admin can do this. Get invitation_id from get_organization.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `invitation_id` | string | yes | Invitation id from get_organization. |

### set_member_role

Change a member's organization role to owner, admin, or member. Only an owner or an admin can do this. Only an owner can make or change an owner. Nobody can change their own role. An organization keeps at least one owner.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `email` | string | yes | The member's email address. |
| `role` | string (owner, admin, member) | yes | The new role. |

### remove_member

Remove a person from an organization. Their organization flows, folders, and teams go to transfer_to, or to you. They lose the organization's Pro seat. An owner can remove anyone; an admin can remove members only.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `email` | string | yes | The member's email address. |
| `transfer_to` | string | no | Optional. Email of a member who stays and gets their organization flows. |

### leave_organization

Leave an organization. Your organization flows go to transfer_to, or to the first other owner. The only owner cannot leave; make another member an owner first, or delete the organization.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `transfer_to` | string | no | Optional. Email of a member who stays and gets your organization flows. |

### get_org_audit_log

List recent organization events, newest first: who did what and when (invites, role changes, removals, settings, billing). Only an owner or an admin can see it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `limit` | integer | no | How many events, 1 to 200. Default 50. |

### delete_organization

Permanently delete an organization. This cannot be undone. Only an owner can do this, and only after the organization subscription is cancelled. confirm_name must equal the organization name exactly. Members lose the organization's Pro seats. Organization flows go back to their creators as personal flows.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `org_id` | string | yes | Organization id from list_organizations. |
| `confirm_name` | string | yes | The exact organization name, to confirm. |

### prepare_live_session

Get ready to present a flow live on a call. This tool does not start the session: a live room exists only in the presenter's browser. It checks that you can present (owner or editor) and returns the presenter link plus steps. Open the link on a computer and click Go live; Wavelength then makes the guest link (wavelengthflow.com/live/<room id>). Guests need no account.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `flow_id` | string | yes | The id of the flow to present. |

### import_from_google_drive

Read a file from the connected Google Drive and make a flow from it, the same way as create_flow_from_text. draw.io, Mermaid, and SVG files become a flow without AI. Reads Google Docs, Sheets, Slides, and .txt, .md, .vtt, .srt, .json, .csv, .pdf, .docx, .drawio, .mmd, .svg files. Set import_only to get the text back without making a flow. Wavelength can open only files it created or files you picked in Wavelength. If Drive is not connected, the error says where to connect it.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_id` | string | no | The Drive file id, or a Drive or Docs url for the file. |
| `url` | string | no | Optional. A Drive or Docs url, in place of file_id. |
| `import_only` | boolean | no | true returns the file text and makes no flow. |
| `title` | string | no | Optional flow title. Default: the file name. |
| `direction` | string (vertical, horizontal) | no | Optional layout direction. Default vertical. |
| `folder_id` | string | no | Optional folder id for the new flow. |
| `team_id` | string | no | Optional team id for the new flow. |
| `org_id` | string | no | Optional organization id. The flow is saved in that organization. |

## Support

Email support@wavelengthflow.com.