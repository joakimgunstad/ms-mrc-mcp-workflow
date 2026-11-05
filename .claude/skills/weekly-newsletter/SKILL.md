---
name: weekly-newsletter
description: Build and email the weekly Microsoft AI at Work Roadmap newsletter for Dynamics 365 CRM and Power Platform (Norway/Europe-highlighted) using the MRC MCP, then update state and commit. Use when asked to run, send, preview or regenerate the roadmap newsletter.
---

# Weekly roadmap newsletter

Follow CLAUDE.md for all rules (scope, badges, format, guardrails). This file is the procedure.

Arguments (optional): `dry-run` means build and save the newsletter, but do NOT send the email,
update state, or commit. Print the Markdown instead.

## 1. Load config and state
- Read `config/products.json` and `state/roadmap-state.json`.
- Today's date, ISO week and year are computed in Europe/Oslo time.

## 2. Pull all in-scope items
- Build one OData filter that ORs every product in all groups:
  `products/any(p: p eq '<product 1>') or products/any(p: p eq '<product 2>') or ...`
- Call `get_recent_m365_roadmaps` with that filter and `skip` = 0, 50, 100, … until `hasMore` is false.
  Expect roughly 200–300 items. If you get 0, stop and report it (a filter or MCP problem).
- If the filter is rejected as too long, run one query per product group and merge by `id`.
- Apply the exclusions (`excluded_title_prefixes`, `excluded_products`) from CLAUDE.md → Scope before
  doing anything else. Excluded items are dropped completely.
- If the mrc-mcp tools aren't loaded, use the direct JSON-RPC fallback described in CLAUDE.md.

## 3. Diff against state
- Classify every item per CLAUDE.md → Change detection (NEW / STATUS CHANGE / DATE SHIFT /
  CLOUD/REGION CHANGE / REMOVED / UPDATED / unchanged).
- Empty state means a baseline run: treat it as a "current landscape" issue, using `baseline_limits` in config:
  - 🟢: items whose latest GA/preview month is within the last ~60 days, up to `green_full` in full detail.
  - 🟡: the first `yellow_full` items (by next date) in full detail. List the rest one line each.
  - ⚠️: all items, one line each.
  - 🔵: the `blue_list` most recently created, one line each.
  - Everything else is counted in the footer.

## 4. Enrich the items that will be shown
For every item in sections 2–6 (and every 🟢/🟡 item in the baseline):
- `get_m365_roadmap_by_id` → full description. Extract the Availability table's Feature Scope and
  any region or language wording.
- For 🟢/🟡 items with a `learn.microsoft.com` link: best effort, fetch it and read the "Geographic areas" /
  language section. If that's unavailable, skip it silently and use "region not stated".
- Assign a badge and region evidence per CLAUDE.md → Region relevance.

## 5. Write the newsletter
- Markdown → `newsletters/YYYY-MM-DD.md`.
- HTML version for the email body: same sections, inline styles only. Use a colored left border per badge
  (🟢 #2e7d32, 🟡 #f9a825, ⚠️ #e65100, 🔵 #1565c0, ⚪/❌ #9e9e9e) and keep it readable on mobile.
- Check the HTML size. If it's over 80 KB, shorten it per CLAUDE.md before sending.

## 6. Send
- Gmail `send_message` to `joakim.gunstad@gmail.com` only, with the subject from CLAUDE.md and the HTML body.
- If the Gmail connector is unavailable: don't update state. Save the newsletter file, commit it
  with message `newsletter: week <NN> <year> (NOT SENT)`, and report the failure.

## 7. Persist (only after a successful send)
- Overwrite `state/roadmap-state.json`:
  `{ "last_run": "<ISO datetime>", "items": { "<id>": { "title", "products", "status",
  "generalAvailabilityDate", "previewAvailabilityDate", "cloudInstances", "modified", "badge",
  "region_note", "feature_scope" } } }`. Include ALL pulled items, not just the ones shown.
  Keep `region_note`/`feature_scope` from the previous state for items you didn't enrich this run.
- `git add state newsletters && git commit -m "newsletter: week <NN> <year>" && git push`.

## 8. Report
Print a summary: number of items pulled, counts per change type and badge, whether the email was sent, and the commit hash.
