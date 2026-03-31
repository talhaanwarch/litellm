# Prompt Environment Support & User Tracking

**Date:** 2026-03-31
**Status:** Approved

## Summary

Add environment support (development, staging, production) and user tracking (created_by) to the LiteLLM prompt management system. Same `prompt_id` can exist independently in different environments with separate version histories. Every prompt version records who created it.

## Decisions

- **Approach:** Add columns directly to `LiteLLM_PromptTable` (no separate tables).
- **Environment is part of prompt identity:** `prompt_id + version + environment` is the unique key.
- **Default environment:** `development`.
- **Backfill:** Existing prompts get `environment = 'development'`.
- **User tracking:** `created_by` only (no `updated_by`), since every edit creates a new version row.
- **Future:** Promotion flow (dev -> staging -> production) will be built later on top of this data model.

## Database Schema Changes

### `LiteLLM_PromptTable`

Add two columns:

| Column | Type | Default | Nullable | Notes |
|--------|------|---------|----------|-------|
| `environment` | String | `"development"` | No | Free-form string, not enum — allows custom environments later |
| `created_by` | String | — | Yes | References `LiteLLM_UserTable.user_id` |

Constraint changes:

| What | Before | After |
|------|--------|-------|
| Unique constraint | `[prompt_id, version]` | `[prompt_id, version, environment]` |
| New index | — | `[prompt_id, environment]` |
| Existing index | `[prompt_id]` | Kept |

### Migration

1. Add `environment` column with default `'development'` — backfills existing rows.
2. Add `created_by` column as nullable.
3. Drop old unique constraint `[prompt_id, version]`, create new `[prompt_id, version, environment]`.
4. Add index `[prompt_id, environment]`.
5. Apply to all `schema.prisma` copies.

## Backend API Changes

### Pydantic Models

- `PromptInfo` — add `environment: Optional[str] = "development"`.
- `Prompt` (create request) — `created_by` set server-side from auth user, not from request body.
- `PatchPromptRequest` — add `environment: Optional[str] = None`.
- Response models — include `environment` and `created_by`.

### Endpoint Changes

| Endpoint | Change |
|----------|--------|
| `POST /prompts` | Accept optional `environment` (default "development"). Set `created_by` from `user_api_key_auth.user_id`. |
| `PUT /prompts/{prompt_id}` | Accept optional `environment`. New version inherits or overrides environment. Set `created_by` from auth. |
| `GET /prompts/list` | Add optional `environment` query param filter. Include `environment` and `created_by` in response. |
| `GET /prompts/{prompt_id}` | Add optional `environment` query param. Scope latest-version resolution to environment. |
| `GET /prompts/{prompt_id}/versions` | Add optional `environment` query param to scope history. |
| `DELETE /prompts/{prompt_id}` | Add optional `environment` query param. Delete all versions in that environment only. |
| `PATCH /prompts/{prompt_id}` | Allow patching `environment`. Set `created_by` from auth. |

### Auth

`user_api_key_auth` already provides `UserAPIKeyAuth` with `user_id`. Pass to DB writes as `created_by`.

## UI Changes

### Prompt List Page (`prompt_table.tsx`)

- Environment filter dropdown at top (All / Development / Staging / Production). Default: All.
- New **Environment** column in the table.
- New **Created By** column in the table.
- Pass `environment` query param to list API when filtered.

### Prompt Editor (`prompt_editor_view/`)

- Environment dropdown (Development / Staging / Production) in the editor header area. Default: Development.
- Pre-fill with current environment when editing existing prompt.
- Include selected environment in create/update API request.

### Prompt Detail Page (`prompt_info.tsx`)

- Display Environment badge.
- Display Created By field (user ID or alias).

### Networking (`networking.tsx`)

- `getPromptsList` — accept optional `environment` param.
- `createPromptCall` / `updatePromptCall` — include `environment` in request body.
- Response types updated with `environment` and `created_by`.

## Backward Compatibility

- All new fields have defaults or are nullable — existing API calls without `environment` work (default to "development").
- `created_by` is never required — old prompts show no author.
- Existing prompts become "development" environment prompts.
