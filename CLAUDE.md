# Project Context

Automated **weekly newsletter** about the **Microsoft AI at Work Roadmap** (formerly the M365 Roadmap),
sourced from the Microsoft Release Communications (MRC) MCP server.

- Reader: me, a Dynamics 365 CRM / Power Platform consultant based in **Norway** (Europe geo,
  Worldwide commercial cloud). Language: **English**. Tone: concise, technical, no marketing fluff.
- Delivery: email to **joakim.gunstad@gmail.com** via the Gmail connector (`send_message`),
  every Monday 07:00 Europe/Oslo (scheduled cloud routine). Manual run: `/weekly-newsletter`.
- The full procedure lives in [.claude/skills/weekly-newsletter/SKILL.md](.claude/skills/weekly-newsletter/SKILL.md).

## MRC MCP server

- Registered in [.mcp.json](.mcp.json) as `mrc-mcp`, via `npx -y mcp-remote <url>` (stdio, no auth).
  Don't switch it to `"type": "http"`: the Microsoft endpoint rejects Claude Code's native HTTP
  transport with a 400 "Bad Request".
- If the mrc-mcp tools aren't available in a session, call the same endpoint directly with JSON-RPC
  `tools/call` (POST, `Accept: application/json, text/event-stream`). It's the same server and the same data.
- Tools used: `get_recent_m365_roadmaps` (list, OData `filter`, `search` on titles, `skip` paging,
  50 per page, truncated descriptions, `include_facets`) and `get_m365_roadmap_by_id` (full detail).
- Never call the Azure tools (`get_recent_azure_updates`, `get_azure_update_by_id`). They're out of scope.
- Item fields: `id, baseId, title, description, status, created, modified, generalAvailabilityDate,
  previewAvailabilityDate (YYYY-MM), products[], platforms[], releaseRings[], cloudInstances[],
  availabilities[{ring, year, month}], moreInfoUrls[]`.
- Status values: `In development`, `Rolling out`, `Launched`, `Cancelled`.
- Full descriptions often end with an "Availability" table (Release Channel, Version, Applies to,
  **Feature Scope** such as "Users, automatically" vs "enabled by admins or makers"). Use it.

## Scope

- IN: roadmap items whose `products` include any value in `groups` of [config/products.json](config/products.json)
  (D365 Sales, Customer Service, Customer Insights – Journeys/Data, Field Service, Power Apps,
  Power Automate, Power Pages, Copilot Studio, Dataverse, PP governance & admin).
- OUT: **Dynamics 365 Contact Center and Microsoft Contact Center Intelligence**. Drop an item if its title starts
  with a value in `excluded_title_prefixes`, or if its only in-scope tags are in `excluded_products`. Items
  titled "Dynamics 365 Customer Service: …" that are also tagged Contact Center stay in. Excluded items
  are not shown, counted or stored in state.
- OUT: Azure Updates, D365 ERP (Finance, SCM, Business Central, etc.), and M365 productivity apps
  (Teams, Outlook, etc.) unless they're also tagged with an in-scope product.
- Use the exact product strings from config/products.json in OData filters. They're case- and dash-sensitive.

# Instructions

## Data rules

- Always get data from the mrc-mcp tools. Never answer from memory.
- Page with `skip` until `hasMore` is false. Never stop at the first 50.
- Call `get_m365_roadmap_by_id` for every item that appears in the main sections of the newsletter
  (list descriptions are truncated).
- Every item links to `https://www.microsoft.com/microsoft-365/roadmap?id=<id>`.
- Never invent dates, statuses or regions. If a field is missing, write "not stated".

## Region relevance (Norway / Europe)

- **Never drop an item for region reasons.** Global "In development" and newly announced items
  stay in the newsletter. Region only drives the badges, highlighting and ordering.
- The MRC data has **no per-country or geo field**. Region signals, strongest first:
  1. `cloudInstances` includes `Worldwide (Standard Multi-Tenant)`: applies to Norwegian tenants.
     Items that are GCC / GCC High / DoD only do **not** apply.
  2. Wording in the full description, e.g. "US only", "North America", "selected regions",
     "regions without … capacity", "Europe/EMEA/EU", language limits ("English only").
  3. Best effort, only for 🟢/🟡 items with a `learn.microsoft.com` link in `moreInfoUrls`: fetch the
     release-plan page (Microsoft Learn MCP) and read its "Geographic areas" / language notes.
- Badges (every item gets exactly one; apply the rules top to bottom, first match wins):
  - ❌ **CANCELLED**: status `Cancelled`.
  - ⚪ **NOT FOR MY TENANT (YET)**: Government clouds only, or explicitly limited to non-European regions.
  - 🟢 **AVAILABLE TO ME**: status `Launched`, or `Rolling out` with the earliest GA/preview month ≤ current
    month, Worldwide cloud, and no region exclusion found.
  - 🟡 **COMING SOON TO ME**: `Rolling out`, or any GA/preview month from the current month to +90 days.
  - ⚠️ **PAST TARGET DATE**: `In development` but the GA month is before the current month (slipping
    or about to land). Watch these for status changes.
  - 🔵 **GLOBAL PIPELINE**: everything else (`In development`, no date or GA more than 90 days out).
- Mark each 🟢/🟡 item with its region evidence: `Worldwide cloud` / `Learn: Europe listed` /
  `region not stated`. Never claim "available in Norway" without evidence. Say "Worldwide cloud, region not stated".
- Show **Feature Scope**: "auto-enabled" vs "admin/maker must enable". The admin-enable items are
  the ones I need to act on.

## Change detection

Compare the current pull against [state/roadmap-state.json](state/roadmap-state.json) (keyed by id):

- **NEW**: id not in state.
- **STATUS CHANGE**: e.g. In development → Rolling out → Launched, or → Cancelled.
- **DATE SHIFT**: `generalAvailabilityDate` / `previewAvailabilityDate` changed. Flag delays (later) with ⚠️.
- **CLOUD/REGION CHANGE**: `cloudInstances` or the stored `region_note` changed.
- **REMOVED**: in state but no longer returned.
- **UPDATED**: `modified` changed with no change to the fields above (list briefly, appendix only).
- First run (empty state): no diff. Send a "baseline / current landscape" issue instead.
- Only after the email is sent: overwrite the state file, save the archive, commit and push.

## Newsletter format

- Subject: `D365 & Power Platform Roadmap – Week <ISO week> <year>`
- Sections, in order:
  1. **TL;DR**: 3–5 bullets, with the most actionable item for me in Norway first.
  2. **⭐ Available to me now**: 🟢 items that are NEW or changed status this week.
  3. **🟡 Coming soon to me (next 90 days)**: new/changed, sorted by date.
  4. **🆕 Newly announced globally**: NEW items that are In development.
  5. **📅 Date changes & delays**, plus ⚠️ items that are past their target date (one line each).
  6. **⚪ Not for my tenant (yet) / ❌ cancelled**: brief.
  7. **Appendix**: everything changed this week, grouped by product group → product, badged.
  8. Footer: counts per badge, and a note that MRC has no geo field (link to
     https://aka.ms/FeatureGeographicAvailabilityReport for authoritative geo checks).
- Each item in sections 2–4: badge · title (linked) · product · status · preview/GA month · region
  evidence · feature scope · 1–2 sentences on "why it matters" for a CRM/Power Platform consultant.
  ⚠️, 🔵, ⚪ and appendix items are one line each (badge · linked title · product · status · dates).
- Send as an HTML email (simple inline styles, no images, no external CSS). Save the same content as
  Markdown in `newsletters/YYYY-MM-DD.md`.
- **Size limit: keep the HTML body under 80 KB**, because Gmail clips messages over ~102 KB. If it's
  bigger, shorten sections 2–3 to the top items (actionable and admin-enable first) and list the
  rest as one-liners.
- If nothing changed: still send a short "No roadmap changes this week" email with the
  🟡 upcoming list.

## Guardrails

- Send email **only** to joakim.gunstad@gmail.com. Never to anyone else, never cc/bcc.
- Send exactly one newsletter per run. If a send fails, don't retry in a loop. Report the error.
- During a run, only write to `state/` and `newsletters/`. Don't touch other files unless asked.
- Commit message: `newsletter: week <NN> <year>`.

# Project Structure

- `.mcp.json`: MRC MCP server registration (project scope)
- `.claude/skills/weekly-newsletter/SKILL.md`: run procedure (`/weekly-newsletter`)
- `config/products.json`: in-scope product facet values, recipient, timezone, coming-soon window
- `state/roadmap-state.json`: last snapshot used for change detection
- `newsletters/`: Markdown archive of every newsletter sent
