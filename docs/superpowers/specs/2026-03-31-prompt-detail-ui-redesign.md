# Prompt Detail Page UI Redesign

**Date:** 2026-03-31
**Status:** Approved

## Summary

Redesign the prompt detail page to make environments actionable. Replace static environment badges with clickable environment tabs that update the entire page. Add a version history table per environment showing who created each version.

## Design

### Environment Tabs

Replace the static "Environments" badge card with environment tabs at the top of the Overview section.

- Tabs rendered as clickable buttons, color-coded: green (development), yellow (staging), red (production)
- Tab label includes version number: `development (v3)`
- Default selection: first available environment
- Clicking a tab calls `getPromptInfo(accessToken, promptId, environment)` and updates the entire page

### Overview Cards (per environment)

When an environment tab is selected, these cards show data for that environment's latest version (or a selected older version):

- **Version** — version number with badge
- **Prompt Type** — db/config
- **Created By** — user who created this version
- **Created At** — timestamp

### Version History Table

A new card below the overview cards, titled "Version History". Shows all versions for the selected environment.

Columns: Version | Created By | Date

- Data comes from `GET /prompts/{prompt_id}/versions?environment=<selected>`
- Rows are clickable — clicking loads that version's content into the page (re-fetches with specific version)
- The latest/current version row is visually distinguished (bold text)
- When viewing an older (non-latest) version, show a banner above the overview cards: "Viewing v2 — not the latest version"

### Tabs Removed/Changed

- **Remove** the "Details" tab — it duplicates Raw JSON with less useful formatting
- **Keep** Overview, Prompt Template, Raw JSON tabs

### Backend

No new endpoints needed. Existing endpoints already support:
- `GET /prompts/{prompt_id}/info?environment=X` — returns prompt for specific environment
- `GET /prompts/{prompt_id}/versions?environment=X` — returns all versions for an environment
- Both return `created_by` and `environment` per version

One minor addition: the versions endpoint response should include `created_by` in each version entry (verify this works end-to-end).

### What Stays the Same

- Top-level header: Prompt ID, copy button, Get Code / Prompt Studio / Delete buttons
- Prompt Template tab content (updates when environment/version changes)
- Raw JSON tab
