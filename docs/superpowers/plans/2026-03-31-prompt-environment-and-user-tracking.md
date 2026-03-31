# Prompt Environment & User Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `environment` (development/staging/production) and `created_by` (user tracking) fields to the prompt management system — schema, backend API, and UI.

**Architecture:** Two new columns on `LiteLLM_PromptTable`: `environment` (String, default "development") and `created_by` (String, nullable). The unique constraint changes from `[prompt_id, version]` to `[prompt_id, version, environment]`. Backend endpoints gain an optional `environment` query/body param. UI gets an environment filter dropdown on the list page and an environment selector in the editor.

**Tech Stack:** Prisma (PostgreSQL), Python/FastAPI, React/TypeScript (Tremor + Ant Design), pytest

**Spec:** `docs/superpowers/specs/2026-03-31-prompt-environment-and-user-tracking-design.md`

---

### Task 1: Update Prisma Schema (all copies)

**Files:**
- Modify: `litellm/proxy/schema.prisma:1001-1012`
- Modify: `schema.prisma:1001-1012` (root copy)
- Modify: `litellm-proxy-extras/litellm_proxy_extras/schema.prisma:1001-1012`

- [ ] **Step 1: Update `litellm/proxy/schema.prisma`**

Replace the `LiteLLM_PromptTable` model at line 1001:

```prisma
// Prompt table for storing prompt configurations
model LiteLLM_PromptTable {
  id String @id @default(uuid())
  prompt_id String
  version Int @default(1)
  environment String @default("development")
  created_by String?
  litellm_params Json
  prompt_info Json?
  created_at DateTime @default(now())
  updated_at DateTime @updatedAt

  @@unique([prompt_id, version, environment])
  @@index([prompt_id, environment])
  @@index([prompt_id])
}
```

- [ ] **Step 2: Apply the same change to root `schema.prisma`**

Same model replacement at line 1001 of `/media/talha/work/ltw/litellm/schema.prisma`.

- [ ] **Step 3: Apply the same change to `litellm-proxy-extras` schema**

Same model replacement at line 1001 of `/media/talha/work/ltw/litellm/litellm-proxy-extras/litellm_proxy_extras/schema.prisma`.

- [ ] **Step 4: Commit**

```bash
git add litellm/proxy/schema.prisma schema.prisma litellm-proxy-extras/litellm_proxy_extras/schema.prisma
git commit -m "feat(schema): add environment and created_by columns to LiteLLM_PromptTable"
```

---

### Task 2: Create Prisma Migration

**Files:**
- Create: `litellm-proxy-extras/litellm_proxy_extras/migrations/20260331000000_add_prompt_environment_and_created_by/migration.sql`

- [ ] **Step 1: Write the migration SQL file**

Create directory and file:

```sql
-- AlterTable
ALTER TABLE "LiteLLM_PromptTable" ADD COLUMN "environment" TEXT NOT NULL DEFAULT 'development';
ALTER TABLE "LiteLLM_PromptTable" ADD COLUMN "created_by" TEXT;

-- DropIndex (old unique constraint)
DROP INDEX IF EXISTS "LiteLLM_PromptTable_prompt_id_version_key";

-- CreateIndex (new unique constraint)
CREATE UNIQUE INDEX "LiteLLM_PromptTable_prompt_id_version_environment_key" ON "LiteLLM_PromptTable"("prompt_id", "version", "environment");

-- CreateIndex (new composite index)
CREATE INDEX "LiteLLM_PromptTable_prompt_id_environment_idx" ON "LiteLLM_PromptTable"("prompt_id", "environment");
```

- [ ] **Step 2: Commit**

```bash
git add litellm-proxy-extras/litellm_proxy_extras/migrations/20260331000000_add_prompt_environment_and_created_by/
git commit -m "feat(migration): add prompt environment and created_by migration"
```

---

### Task 3: Update Pydantic Models

**Files:**
- Modify: `litellm/types/prompts/init_prompts.py:18-76`
- Test: `tests/test_litellm/proxy/prompts/test_prompt_environment.py` (new)

- [ ] **Step 1: Write tests for updated models**

Create `tests/test_litellm/proxy/prompts/test_prompt_environment.py`:

```python
import pytest
from litellm.types.prompts.init_prompts import (
    PromptInfo,
    PromptSpec,
    PromptLiteLLMParams,
)


def test_prompt_info_default_environment():
    """PromptInfo should default environment to 'development'."""
    info = PromptInfo(prompt_type="db")
    assert info.environment == "development"


def test_prompt_info_custom_environment():
    """PromptInfo should accept a custom environment."""
    info = PromptInfo(prompt_type="db", environment="production")
    assert info.environment == "production"


def test_prompt_spec_includes_environment_and_created_by():
    """PromptSpec should carry environment and created_by fields."""
    spec = PromptSpec(
        prompt_id="test",
        litellm_params=PromptLiteLLMParams(
            prompt_id="test", prompt_integration="dotprompt"
        ),
        prompt_info=PromptInfo(prompt_type="db", environment="staging"),
        environment="staging",
        created_by="user-123",
    )
    assert spec.environment == "staging"
    assert spec.created_by == "user-123"


def test_prompt_spec_default_environment():
    """PromptSpec environment should default to 'development'."""
    spec = PromptSpec(
        prompt_id="test",
        litellm_params=PromptLiteLLMParams(
            prompt_id="test", prompt_integration="dotprompt"
        ),
        prompt_info=PromptInfo(prompt_type="db"),
    )
    assert spec.environment == "development"
    assert spec.created_by is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v`
Expected: FAIL — `PromptSpec` and `PromptInfo` don't have `environment`/`created_by` fields yet.

- [ ] **Step 3: Update `PromptInfo` model**

In `litellm/types/prompts/init_prompts.py`, change the `PromptInfo` class (line 18):

```python
class PromptInfo(BaseModel):
    prompt_type: Literal["config", "db"]
    environment: Optional[str] = "development"

    model_config = ConfigDict(extra="allow", protected_namespaces=())
```

- [ ] **Step 4: Update `PromptSpec` model**

In `litellm/types/prompts/init_prompts.py`, add fields to `PromptSpec` (line 44):

```python
class PromptSpec(BaseModel):
    prompt_id: str
    litellm_params: PromptLiteLLMParams
    prompt_info: PromptInfo
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
    version: Optional[int] = None
    environment: Optional[str] = "development"
    created_by: Optional[str] = None

    def __init__(self, **data):
        if "prompt_info" not in data:
            data["prompt_info"] = PromptInfo(prompt_type="config")
        elif "prompt_info" in data:
            if (
                isinstance(data["prompt_info"], dict)
                and data["prompt_info"].get("prompt_type") is None
            ):
                data["prompt_info"]["prompt_type"] = "config"
        super().__init__(**data)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v`
Expected: All 4 tests PASS.

- [ ] **Step 6: Run existing prompt tests to check no regressions**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/ -v`
Expected: All existing tests still PASS.

- [ ] **Step 7: Commit**

```bash
git add litellm/types/prompts/init_prompts.py tests/test_litellm/proxy/prompts/test_prompt_environment.py
git commit -m "feat(models): add environment and created_by to PromptInfo and PromptSpec"
```

---

### Task 4: Update Backend Endpoint Models and Helper Functions

**Files:**
- Modify: `litellm/proxy/prompts/prompt_endpoints.py:195-270` (helper functions and request models)
- Test: `tests/test_litellm/proxy/prompts/test_prompt_environment.py` (append)

- [ ] **Step 1: Write tests for updated helpers and request models**

Append to `tests/test_litellm/proxy/prompts/test_prompt_environment.py`:

```python
from litellm.proxy.prompts.prompt_endpoints import (
    Prompt,
    PatchPromptRequest,
    create_versioned_prompt_spec,
)
from unittest.mock import MagicMock
import json


def test_prompt_request_model_with_environment():
    """Prompt request model should accept environment in prompt_info."""
    prompt = Prompt(
        prompt_id="test",
        litellm_params=PromptLiteLLMParams(
            prompt_id="test", prompt_integration="dotprompt"
        ),
        prompt_info=PromptInfo(prompt_type="db", environment="staging"),
    )
    assert prompt.prompt_info.environment == "staging"


def test_patch_request_model_with_environment():
    """PatchPromptRequest should accept environment in prompt_info."""
    patch = PatchPromptRequest(
        prompt_info=PromptInfo(prompt_type="db", environment="production"),
    )
    assert patch.prompt_info.environment == "production"


def test_create_versioned_prompt_spec_includes_environment():
    """create_versioned_prompt_spec should populate environment and created_by from DB row."""
    mock_db_prompt = MagicMock()
    mock_db_prompt.model_dump.return_value = {
        "id": "uuid-123",
        "prompt_id": "test_prompt",
        "version": 2,
        "environment": "staging",
        "created_by": "user-456",
        "litellm_params": json.dumps({
            "prompt_id": "test_prompt",
            "prompt_integration": "dotprompt",
        }),
        "prompt_info": json.dumps({"prompt_type": "db", "environment": "staging"}),
        "created_at": None,
        "updated_at": None,
    }
    spec = create_versioned_prompt_spec(mock_db_prompt)
    assert spec.environment == "staging"
    assert spec.created_by == "user-456"
    assert spec.prompt_id == "test_prompt.v2"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v -k "test_prompt_request_model or test_patch_request or test_create_versioned"`
Expected: FAIL — `create_versioned_prompt_spec` doesn't pass `environment`/`created_by` to `PromptSpec` yet.

- [ ] **Step 3: Update `get_next_version_for_prompt` to accept environment**

In `litellm/proxy/prompts/prompt_endpoints.py`, update the function at line 195:

```python
async def get_next_version_for_prompt(
    prisma_client, prompt_id: str, environment: str = "development"
) -> int:
    """
    Get the next version number for a prompt in a specific environment.

    Args:
        prisma_client: Prisma database client
        prompt_id: Base prompt ID
        environment: Environment to scope version lookup

    Returns:
        Next version number (1 if no versions exist, max_version + 1 otherwise)
    """
    existing_prompts = await prisma_client.db.litellm_prompttable.find_many(
        where={"prompt_id": prompt_id, "environment": environment}
    )

    if existing_prompts:
        max_version = max(p.version for p in existing_prompts)
        return max_version + 1
    else:
        return 1
```

- [ ] **Step 4: Update `create_versioned_prompt_spec` to include new fields**

In `litellm/proxy/prompts/prompt_endpoints.py`, update the function at line 217:

```python
def create_versioned_prompt_spec(db_prompt) -> PromptSpec:
    """
    Helper function to create a PromptSpec with versioned prompt_id from a DB prompt entry.
    """
    import json

    from litellm.types.prompts.init_prompts import PromptLiteLLMParams

    prompt_dict = db_prompt.model_dump()
    base_prompt_id = prompt_dict["prompt_id"]
    version = prompt_dict.get("version", 1)
    environment = prompt_dict.get("environment", "development")
    created_by = prompt_dict.get("created_by")

    # Parse litellm_params
    litellm_params_data = prompt_dict.get("litellm_params")
    if isinstance(litellm_params_data, str):
        litellm_params_data = json.loads(litellm_params_data)
    litellm_params = PromptLiteLLMParams(**litellm_params_data)

    # Parse prompt_info
    prompt_info_data = prompt_dict.get("prompt_info")
    if prompt_info_data:
        if isinstance(prompt_info_data, str):
            prompt_info_data = json.loads(prompt_info_data)
        prompt_info = PromptInfo(**prompt_info_data)
    else:
        prompt_info = PromptInfo(prompt_type="db")

    # Create versioned prompt_id
    versioned_prompt_id = f"{base_prompt_id}.v{version}"

    return PromptSpec(
        prompt_id=versioned_prompt_id,
        litellm_params=litellm_params,
        prompt_info=prompt_info,
        created_at=prompt_dict.get("created_at"),
        updated_at=prompt_dict.get("updated_at"),
        environment=environment,
        created_by=created_by,
    )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v`
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add litellm/proxy/prompts/prompt_endpoints.py tests/test_litellm/proxy/prompts/test_prompt_environment.py
git commit -m "feat(endpoints): update helpers to support environment and created_by"
```

---

### Task 5: Update Create Prompt Endpoint

**Files:**
- Modify: `litellm/proxy/prompts/prompt_endpoints.py:591-678`
- Test: `tests/test_litellm/proxy/prompts/test_prompt_environment.py` (append)

- [ ] **Step 1: Write test for create with environment and created_by**

Append to `tests/test_litellm/proxy/prompts/test_prompt_environment.py`:

```python
import pytest
from unittest.mock import AsyncMock, patch
from litellm.proxy._types import UserAPIKeyAuth, LitellmUserRoles


@pytest.mark.asyncio
async def test_create_prompt_stores_environment_and_created_by():
    """
    create_prompt should pass environment from prompt_info and
    created_by from the authenticated user to the DB.
    """
    from litellm.proxy.prompts.prompt_endpoints import create_prompt, Prompt

    mock_user_auth = UserAPIKeyAuth(
        api_key="sk-1234",
        user_role=LitellmUserRoles.PROXY_ADMIN,
        user_id="user-789",
    )

    mock_prisma_client = MagicMock()

    # Mock the DB create to return a fake db entry
    mock_db_entry = MagicMock()
    mock_db_entry.model_dump.return_value = {
        "id": "uuid-1",
        "prompt_id": "my_prompt",
        "version": 1,
        "environment": "staging",
        "created_by": "user-789",
        "litellm_params": json.dumps({
            "prompt_id": "my_prompt",
            "prompt_integration": "dotprompt",
        }),
        "prompt_info": json.dumps({"prompt_type": "db", "environment": "staging"}),
        "created_at": None,
        "updated_at": None,
    }
    mock_prisma_client.db.litellm_prompttable.create = AsyncMock(
        return_value=mock_db_entry
    )
    mock_prisma_client.db.litellm_prompttable.find_many = AsyncMock(
        return_value=[]
    )

    request = Prompt(
        prompt_id="my_prompt",
        litellm_params=PromptLiteLLMParams(
            prompt_id="my_prompt", prompt_integration="dotprompt"
        ),
        prompt_info=PromptInfo(prompt_type="db", environment="staging"),
    )

    with patch("litellm.proxy.proxy_server.prisma_client", mock_prisma_client):
        with patch(
            "litellm.proxy.prompts.prompt_registry.IN_MEMORY_PROMPT_REGISTRY"
        ) as mock_registry:
            mock_registry.initialize_prompt.return_value = PromptSpec(
                prompt_id="my_prompt.v1",
                litellm_params=request.litellm_params,
                prompt_info=request.prompt_info,
                environment="staging",
                created_by="user-789",
            )

            result = await create_prompt(
                request=request, user_api_key_dict=mock_user_auth
            )

            # Verify DB create was called with environment and created_by
            create_call = mock_prisma_client.db.litellm_prompttable.create.call_args
            data = create_call.kwargs["data"]
            assert data["environment"] == "staging"
            assert data["created_by"] == "user-789"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py::test_create_prompt_stores_environment_and_created_by -v`
Expected: FAIL — `create_prompt` doesn't pass `environment`/`created_by` to DB yet.

- [ ] **Step 3: Update `create_prompt` endpoint**

In `litellm/proxy/prompts/prompt_endpoints.py`, update the `create_prompt` function (line 596). Change the DB create call (around line 650-661):

```python
async def create_prompt(
    request: Prompt,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
    """
    Create a new prompt

    👉 [Prompt docs](https://docs.litellm.ai/docs/proxy/prompt_management)

    Example Request:
    ```bash
    curl -X POST "http://localhost:4000/prompts" \\
        -H "Authorization: Bearer <your_api_key>" \\
        -H "Content-Type: application/json" \\
        -d '{
            "prompt_id": "my_prompt",
            "litellm_params": {
                "prompt_id": "json_prompt",
                "prompt_integration": "dotprompt",
                ### EITHER prompt_directory OR prompt_data MUST BE PROVIDED
                "prompt_directory": "/path/to/dotprompt/folder",
                "prompt_data": {"json_prompt": {"content": "This is a prompt", "metadata": {"model": "gpt-4"}}}
            },
            "prompt_info": {
                "prompt_type": "config"
            }
        }'
    ```
    """

    from litellm.proxy.prompts.prompt_registry import IN_MEMORY_PROMPT_REGISTRY
    from litellm.proxy.proxy_server import prisma_client

    # Only allow proxy admins to create prompts
    if user_api_key_dict.user_role is None or (
        user_api_key_dict.user_role != LitellmUserRoles.PROXY_ADMIN
        and user_api_key_dict.user_role != LitellmUserRoles.PROXY_ADMIN.value
    ):
        raise HTTPException(
            status_code=403, detail="Only proxy admins can create prompts"
        )

    if prisma_client is None:
        raise HTTPException(
            status_code=500, detail=CommonProxyErrors.db_not_connected_error.value
        )

    try:
        # Resolve environment from prompt_info
        environment = (
            request.prompt_info.environment
            if request.prompt_info and request.prompt_info.environment
            else "development"
        )

        # Get next version number scoped to environment
        new_version = await get_next_version_for_prompt(
            prisma_client=prisma_client,
            prompt_id=request.prompt_id,
            environment=environment,
        )

        # Store prompt in db with version, environment, and created_by
        prompt_db_entry = await prisma_client.db.litellm_prompttable.create(
            data={
                "prompt_id": request.prompt_id,
                "version": new_version,
                "environment": environment,
                "created_by": user_api_key_dict.user_id,
                "litellm_params": request.litellm_params.model_dump_json(),
                "prompt_info": (
                    request.prompt_info.model_dump_json()
                    if request.prompt_info
                    else PromptInfo(prompt_type="db").model_dump_json()
                ),
            }
        )

        # Create versioned prompt spec
        prompt_spec = create_versioned_prompt_spec(db_prompt=prompt_db_entry)

        # Initialize the prompt
        initialized_prompt = IN_MEMORY_PROMPT_REGISTRY.initialize_prompt(
            prompt=prompt_spec, config_file_path=None
        )

        if initialized_prompt is None:
            raise HTTPException(status_code=500, detail="Failed to initialize prompt")

        return initialized_prompt

    except Exception as e:
        verbose_proxy_logger.exception(f"Error creating prompt: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add litellm/proxy/prompts/prompt_endpoints.py tests/test_litellm/proxy/prompts/test_prompt_environment.py
git commit -m "feat(create): store environment and created_by when creating prompts"
```

---

### Task 6: Update Update/Delete/Patch/List/Get Endpoints

**Files:**
- Modify: `litellm/proxy/prompts/prompt_endpoints.py:273-998`
- Test: `tests/test_litellm/proxy/prompts/test_prompt_environment.py` (append)

- [ ] **Step 1: Write test for update endpoint with environment**

Append to `tests/test_litellm/proxy/prompts/test_prompt_environment.py`:

```python
@pytest.mark.asyncio
async def test_update_prompt_stores_environment_and_created_by():
    """
    update_prompt should pass environment and created_by to the new version row.
    """
    from litellm.proxy.prompts.prompt_endpoints import update_prompt, Prompt

    mock_user_auth = UserAPIKeyAuth(
        api_key="sk-1234",
        user_role=LitellmUserRoles.PROXY_ADMIN,
        user_id="user-update",
    )

    mock_prisma_client = MagicMock()

    # Existing prompt in DB
    mock_existing = MagicMock()
    mock_existing.version = 1
    mock_prisma_client.db.litellm_prompttable.find_many = AsyncMock(
        return_value=[mock_existing]
    )

    # Mock the DB create for new version
    mock_db_entry = MagicMock()
    mock_db_entry.model_dump.return_value = {
        "id": "uuid-2",
        "prompt_id": "my_prompt",
        "version": 2,
        "environment": "production",
        "created_by": "user-update",
        "litellm_params": json.dumps({
            "prompt_id": "my_prompt",
            "prompt_integration": "dotprompt",
        }),
        "prompt_info": json.dumps({"prompt_type": "db", "environment": "production"}),
        "created_at": None,
        "updated_at": None,
    }
    mock_prisma_client.db.litellm_prompttable.create = AsyncMock(
        return_value=mock_db_entry
    )

    request = Prompt(
        prompt_id="my_prompt",
        litellm_params=PromptLiteLLMParams(
            prompt_id="my_prompt", prompt_integration="dotprompt"
        ),
        prompt_info=PromptInfo(prompt_type="db", environment="production"),
    )

    with patch("litellm.proxy.proxy_server.prisma_client", mock_prisma_client):
        with patch(
            "litellm.proxy.prompts.prompt_registry.IN_MEMORY_PROMPT_REGISTRY"
        ) as mock_registry:
            mock_registry.get_prompt_by_id.return_value = PromptSpec(
                prompt_id="my_prompt.v1",
                litellm_params=request.litellm_params,
                prompt_info=PromptInfo(prompt_type="db"),
            )
            mock_registry.initialize_prompt.return_value = PromptSpec(
                prompt_id="my_prompt.v2",
                litellm_params=request.litellm_params,
                prompt_info=request.prompt_info,
                environment="production",
                created_by="user-update",
            )

            result = await update_prompt(
                prompt_id="my_prompt",
                request=request,
                user_api_key_dict=mock_user_auth,
            )

            # Verify DB create was called with environment and created_by
            create_call = mock_prisma_client.db.litellm_prompttable.create.call_args
            data = create_call.kwargs["data"]
            assert data["environment"] == "production"
            assert data["created_by"] == "user-update"


@pytest.mark.asyncio
async def test_delete_prompt_scoped_to_environment():
    """
    delete_prompt with environment param should only delete versions in that environment.
    """
    from litellm.proxy.prompts.prompt_endpoints import delete_prompt

    mock_user_auth = UserAPIKeyAuth(
        api_key="sk-1234",
        user_role=LitellmUserRoles.PROXY_ADMIN,
    )

    mock_prisma_client = MagicMock()
    mock_prisma_client.db.litellm_prompttable.delete_many = AsyncMock(return_value=None)

    with patch(
        "litellm.proxy.prompts.prompt_registry.IN_MEMORY_PROMPT_REGISTRY"
    ) as mock_registry:
        prompt_spec = PromptSpec(
            prompt_id="test_prompt.v1",
            litellm_params=PromptLiteLLMParams(
                prompt_id="test_prompt", prompt_integration="dotprompt"
            ),
            prompt_info=PromptInfo(prompt_type="db"),
            environment="staging",
        )
        mock_registry.get_prompt_by_id.return_value = prompt_spec

        with patch("litellm.proxy.proxy_server.prisma_client", mock_prisma_client):
            response = await delete_prompt(
                prompt_id="test_prompt",
                user_api_key_dict=mock_user_auth,
                environment="staging",
            )

            # Verify DB deletion includes environment filter
            mock_prisma_client.db.litellm_prompttable.delete_many.assert_called_once_with(
                where={"prompt_id": "test_prompt", "environment": "staging"}
            )
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/test_prompt_environment.py -v -k "test_update_prompt_stores or test_delete_prompt_scoped"`
Expected: FAIL — endpoints don't accept `environment` param yet.

- [ ] **Step 3: Update `list_prompts` endpoint**

In `litellm/proxy/prompts/prompt_endpoints.py`, update the function signature and body at line 279. Add `environment` query param:

```python
async def list_prompts(
    environment: Optional[str] = None,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
```

In the proxy admin branch (around line 344-358), filter by environment if provided:

```python
    if user_api_key_dict.user_role is not None and (
        user_api_key_dict.user_role == LitellmUserRoles.PROXY_ADMIN
        or user_api_key_dict.user_role == LitellmUserRoles.PROXY_ADMIN.value
    ):
        # Get all prompts and filter to show only the latest version of each
        all_prompts = list(IN_MEMORY_PROMPT_REGISTRY.IN_MEMORY_PROMPTS.values())
        # Filter by environment if specified
        if environment:
            all_prompts = [
                p for p in all_prompts if p.environment == environment
            ]
        latest_prompts = get_latest_prompt_versions(prompts=all_prompts)
        # Create copies with base prompt_id for display
        prompts_for_display = []
        for original_prompt in latest_prompts:
            prompt_copy = PromptSpec(
                prompt_id=get_base_prompt_id(prompt_id=original_prompt.prompt_id),
                litellm_params=original_prompt.litellm_params,
                prompt_info=original_prompt.prompt_info,
                created_at=original_prompt.created_at,
                updated_at=original_prompt.updated_at,
                environment=original_prompt.environment,
                created_by=original_prompt.created_by,
            )
            prompts_for_display.append(prompt_copy)
        return ListPromptsResponse(prompts=prompts_for_display)
```

Also update the key metadata branch (around line 319-338) to include environment/created_by in the copy:

```python
            prompt_copy = PromptSpec(
                prompt_id=get_base_prompt_id(prompt_id=original_prompt.prompt_id),
                litellm_params=original_prompt.litellm_params,
                prompt_info=original_prompt.prompt_info,
                created_at=original_prompt.created_at,
                updated_at=original_prompt.updated_at,
                environment=original_prompt.environment,
                created_by=original_prompt.created_by,
            )
```

- [ ] **Step 4: Update `get_prompt_versions` endpoint**

At line 369, add `environment` query param:

```python
async def get_prompt_versions(
    prompt_id: str,
    environment: Optional[str] = None,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
```

In the filtering logic (around line 421-426), add environment filter:

```python
    prompt_versions = [
        prompt
        for prompt in all_prompts
        if get_base_prompt_id(prompt_id=prompt.prompt_id) == base_prompt_id
        and (environment is None or prompt.environment == environment)
    ]
```

And in the versioned_prompt creation (around line 444-451), include new fields:

```python
        versioned_prompt = PromptSpec(
            prompt_id=base_prompt_id,
            litellm_params=prompt.litellm_params,
            prompt_info=prompt.prompt_info,
            created_at=prompt.created_at,
            updated_at=prompt.updated_at,
            version=version_number,
            environment=prompt.environment,
            created_by=prompt.created_by,
        )
```

- [ ] **Step 5: Update `get_prompt_info` endpoint**

At line 472, add `environment` query param:

```python
async def get_prompt_info(
    prompt_id: str,
    environment: Optional[str] = None,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
```

In the prompt_spec_response creation (around line 545-552), include new fields:

```python
    prompt_spec_response = PromptSpec(
        prompt_id=get_base_prompt_id(prompt_id=prompt_spec.prompt_id),
        litellm_params=prompt_spec.litellm_params,
        prompt_info=prompt_spec.prompt_info,
        created_at=prompt_spec.created_at,
        updated_at=prompt_spec.updated_at,
        version=version_number,
        environment=prompt_spec.environment,
        created_by=prompt_spec.created_by,
    )
```

- [ ] **Step 6: Update `update_prompt` endpoint**

At line 686, the function body. After resolving `base_prompt_id` (line 734), extract environment and scope the version lookup and DB create:

```python
        # Resolve environment
        environment = (
            request.prompt_info.environment
            if request.prompt_info and request.prompt_info.environment
            else "development"
        )

        # Check if any version exists in this environment
        existing_prompts = await prisma_client.db.litellm_prompttable.find_many(
            where={"prompt_id": base_prompt_id, "environment": environment}
        )

        if not existing_prompts:
            raise HTTPException(
                status_code=404,
                detail=f"Prompt with ID {base_prompt_id} not found in environment {environment}",
            )

        # ... (config check stays the same)

        # Get next version number scoped to environment
        new_version = await get_next_version_for_prompt(
            prisma_client=prisma_client,
            prompt_id=base_prompt_id,
            environment=environment,
        )

        # Store new version in db
        prompt_db_entry = await prisma_client.db.litellm_prompttable.create(
            data={
                "prompt_id": base_prompt_id,
                "version": new_version,
                "environment": environment,
                "created_by": user_api_key_dict.user_id,
                "litellm_params": request.litellm_params.model_dump_json(),
                "prompt_info": (
                    request.prompt_info.model_dump_json()
                    if request.prompt_info
                    else PromptInfo(prompt_type="db").model_dump_json()
                ),
            }
        )
```

- [ ] **Step 7: Update `delete_prompt` endpoint**

At line 801, add `environment` query param:

```python
async def delete_prompt(
    prompt_id: str,
    environment: Optional[str] = None,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
```

Update the DB delete call (around line 871) to include environment filter when provided:

```python
        # Delete versions from the database (scoped to environment if provided)
        delete_where = {"prompt_id": base_prompt_id}
        if environment:
            delete_where["environment"] = environment
        await prisma_client.db.litellm_prompttable.delete_many(
            where=delete_where
        )
```

- [ ] **Step 8: Update `patch_prompt` endpoint**

At line 892, set `created_by` from auth user in the DB update. Update the data dict (around line 970-975) to include `created_by`:

```python
        update_data = {
            "litellm_params": updated_litellm_params.model_dump_json(),
            "prompt_info": updated_prompt_info.model_dump_json(),
        }
        if user_api_key_dict.user_id:
            update_data["created_by"] = user_api_key_dict.user_id

        updated_prompt_db_entry = await prisma_client.db.litellm_prompttable.update(
            where={"prompt_id": prompt_id},
            data=update_data,
        )
```

- [ ] **Step 9: Run all prompt tests**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/ -v`
Expected: All tests PASS (existing tests may need minor updates if they assert on exact call signatures — update stubs to accept `environment` kwarg).

- [ ] **Step 10: Commit**

```bash
git add litellm/proxy/prompts/prompt_endpoints.py tests/test_litellm/proxy/prompts/test_prompt_environment.py
git commit -m "feat(endpoints): add environment and created_by to all prompt CRUD endpoints"
```

---

### Task 7: Update UI TypeScript Types and Networking

**Files:**
- Modify: `ui/litellm-dashboard/src/components/networking.tsx:203-214,6038-6200`

- [ ] **Step 1: Update `PromptInfo` interface**

In `ui/litellm-dashboard/src/components/networking.tsx`, update the `PromptInfo` interface at line 203:

```typescript
interface PromptInfo {
  prompt_type: string;
  environment?: string;
}
```

- [ ] **Step 2: Update `PromptSpec` interface**

At line 207:

```typescript
export interface PromptSpec {
  prompt_id: string;
  litellm_params: object;
  prompt_info: PromptInfo;
  created_at?: string;
  updated_at?: string;
  version?: number;
  environment?: string;
  created_by?: string;
}
```

- [ ] **Step 3: Update `getPromptsList` to accept environment filter**

At line 6038:

```typescript
export const getPromptsList = async (
  accessToken: string,
  environment?: string,
): Promise<ListPromptsResponse> => {
  try {
    let url = proxyBaseUrl ? `${proxyBaseUrl}/prompts/list` : `/prompts/list`;
    if (environment) {
      url += `?environment=${encodeURIComponent(environment)}`;
    }
    const response = await fetch(url, {
      method: "GET",
      headers: {
        [globalLitellmHeaderName]: `Bearer ${accessToken}`,
        "Content-Type": "application/json",
      },
    });

    if (!response.ok) {
      const errorData = await response.json();
      const errorMessage = deriveErrorMessage(errorData);
      handleError(errorMessage);
      throw new Error(errorMessage);
    }

    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Failed to get prompts list:", error);
    throw error;
  }
};
```

- [ ] **Step 4: Commit**

```bash
git add ui/litellm-dashboard/src/components/networking.tsx
git commit -m "feat(ui): update TypeScript types and networking for environment support"
```

---

### Task 8: Add Environment Filter to Prompt List Page

**Files:**
- Modify: `ui/litellm-dashboard/src/components/prompts.tsx:1-183`
- Modify: `ui/litellm-dashboard/src/components/prompts/prompt_table.tsx:1-312`

- [ ] **Step 1: Add environment state and filter dropdown to `PromptsPanel`**

In `ui/litellm-dashboard/src/components/prompts.tsx`, add state and pass to `fetchPrompts`:

Add import for `Select` from antd at the top:

```typescript
import { Modal, Select } from "antd";
```

Add state after line 21 (`const [isLoading, setIsLoading] = useState(false);`):

```typescript
const [selectedEnvironment, setSelectedEnvironment] = useState<string | undefined>(undefined);
```

Update `fetchPrompts` (line 30) to use environment:

```typescript
const fetchPrompts = async () => {
    if (!accessToken) {
      return;
    }

    setIsLoading(true);
    try {
      const response: ListPromptsResponse = await getPromptsList(accessToken, selectedEnvironment);
      console.log(`prompts: ${JSON.stringify(response)}`);
      setPromptsList(response.prompts);
    } catch (error) {
      console.error("Error fetching prompts:", error);
    } finally {
      setIsLoading(false);
    }
  };
```

Update `useEffect` (line 47) to depend on `selectedEnvironment`:

```typescript
useEffect(() => {
    fetchPrompts();
  }, [accessToken, selectedEnvironment]);
```

Add the environment dropdown in the JSX (around line 136-144), next to existing buttons:

```tsx
<div className="flex justify-between items-center mb-4">
  <div className="flex gap-2">
    <Button onClick={handleAddPrompt} disabled={!accessToken}>
      + Add New Prompt
    </Button>
    <Button onClick={handleAddPromptFromFile} disabled={!accessToken} variant="secondary">
      Upload .prompt File
    </Button>
  </div>
  <Select
    placeholder="All Environments"
    allowClear
    value={selectedEnvironment}
    onChange={(value) => setSelectedEnvironment(value)}
    style={{ width: 180 }}
    options={[
      { label: "Development", value: "development" },
      { label: "Staging", value: "staging" },
      { label: "Production", value: "production" },
    ]}
  />
</div>
```

- [ ] **Step 2: Add Environment and Created By columns to `PromptTable`**

In `ui/litellm-dashboard/src/components/prompts/prompt_table.tsx`, add two new column definitions after the "Updated At" column (after line 185, before the "Type" column):

```typescript
    {
      header: "Environment",
      accessorKey: "environment",
      cell: ({ row }) => {
        const prompt = row.original;
        const env = prompt.environment || "development";
        const colorMap: Record<string, string> = {
          production: "text-red-600 bg-red-50",
          staging: "text-yellow-600 bg-yellow-50",
          development: "text-green-600 bg-green-50",
        };
        return (
          <span className={`text-xs px-2 py-0.5 rounded ${colorMap[env] || "text-gray-600 bg-gray-50"}`}>
            {env}
          </span>
        );
      },
    },
    {
      header: "Created By",
      accessorKey: "created_by",
      cell: ({ row }) => {
        const prompt = row.original;
        return (
          <span className="text-xs text-gray-600">
            {prompt.created_by || "-"}
          </span>
        );
      },
    },
```

- [ ] **Step 3: Commit**

```bash
git add ui/litellm-dashboard/src/components/prompts.tsx ui/litellm-dashboard/src/components/prompts/prompt_table.tsx
git commit -m "feat(ui): add environment filter dropdown and columns to prompt list"
```

---

### Task 9: Add Environment Selector to Prompt Editor

**Files:**
- Modify: `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/types.ts:12-23`
- Modify: `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/index.tsx:17-229`
- Modify: `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/PromptEditorHeader.tsx:1-80`

- [ ] **Step 1: Add `environment` to `PromptType` interface**

In `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/types.ts`, update `PromptType`:

```typescript
export interface PromptType {
  name: string;
  model: string;
  config: {
    temperature?: number;
    max_tokens?: number;
    top_p?: number;
  };
  tools: Tool[];
  developerMessage: string;
  messages: Message[];
  environment: string;
}
```

- [ ] **Step 2: Update `PromptEditorHeader` to include environment dropdown**

In `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/PromptEditorHeader.tsx`:

Add `Select` import from antd:

```typescript
import { Input, Select } from "antd";
```

Add `environment` and `onEnvironmentChange` to the props interface:

```typescript
interface PromptEditorHeaderProps {
  promptName: string;
  onNameChange: (name: string) => void;
  onBack: () => void;
  onSave: () => void;
  isSaving: boolean;
  editMode?: boolean;
  onShowHistory?: () => void;
  version?: string | null;
  promptModel?: string;
  promptVariables?: Record<string, string>;
  accessToken: string | null;
  proxySettings?: {
    PROXY_BASE_URL?: string;
    LITELLM_UI_API_DOC_BASE_URL?: string | null;
  };
  environment: string;
  onEnvironmentChange: (env: string) => void;
}
```

Add to destructured props:

```typescript
const PromptEditorHeader: React.FC<PromptEditorHeaderProps> = ({
  promptName,
  onNameChange,
  onBack,
  onSave,
  isSaving,
  editMode = false,
  onShowHistory,
  version,
  promptModel = "gpt-4o",
  promptVariables = {},
  accessToken,
  proxySettings,
  environment,
  onEnvironmentChange,
}) => {
```

Add the environment dropdown in the header JSX, after the version badge and before the "Draft" span:

```tsx
        <Select
          value={environment}
          onChange={onEnvironmentChange}
          style={{ width: 140 }}
          size="small"
          options={[
            { label: "Development", value: "development" },
            { label: "Staging", value: "staging" },
            { label: "Production", value: "production" },
          ]}
        />
```

- [ ] **Step 3: Update `PromptEditorView` to use environment**

In `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/index.tsx`:

Update `getInitialPrompt` (line 18) to include environment:

```typescript
  const getInitialPrompt = (): PromptType => {
    if (initialPromptData) {
      try {
        return parseExistingPrompt(initialPromptData);
      } catch (error) {
        console.error("Error parsing existing prompt:", error);
        NotificationsManager.fromBackend("Failed to parse prompt data");
      }
    }
    return {
      name: "New prompt",
      model: "gpt-4o",
      config: {
        temperature: 1,
        max_tokens: 1000,
      },
      tools: [],
      developerMessage: "",
      messages: [
        {
          role: "user",
          content: "Enter task specifics. Use {{template_variables}} for dynamic inputs",
        },
      ],
      environment: "development",
    };
  };
```

Update `handleSave` (around line 202-212) to include environment in promptData:

```typescript
      const promptData = {
        prompt_id: promptId,
        litellm_params: {
          prompt_integration: "dotprompt",
          prompt_id: promptId,
          dotprompt_content: dotpromptContent,
        },
        prompt_info: {
          prompt_type: "db",
          environment: prompt.environment,
        },
      };
```

Pass `environment` and `onEnvironmentChange` to `PromptEditorHeader` in the JSX (find where `<PromptEditorHeader` is rendered):

```tsx
        <PromptEditorHeader
          promptName={prompt.name}
          onNameChange={(name) => setPrompt({ ...prompt, name })}
          onBack={onClose}
          onSave={handleSaveClick}
          isSaving={isSaving}
          editMode={editMode}
          onShowHistory={() => setShowHistoryModal(true)}
          version={editMode ? `v${initialPromptData?.prompt_spec?.version || 1}` : null}
          promptModel={prompt.model}
          promptVariables={extractVariables()}
          accessToken={accessToken}
          environment={prompt.environment}
          onEnvironmentChange={(env) => setPrompt({ ...prompt, environment: env })}
        />
```

- [ ] **Step 4: Update `parseExistingPrompt` utility to include environment**

In `ui/litellm-dashboard/src/components/prompts/prompt_editor_view/utils.ts` (or wherever `parseExistingPrompt` is defined), ensure it reads the `environment` from `prompt_spec.environment` or `prompt_spec.prompt_info.environment` and includes it in the returned `PromptType`:

```typescript
// Add to the returned object:
environment: data.prompt_spec?.environment || data.prompt_spec?.prompt_info?.environment || "development",
```

- [ ] **Step 5: Commit**

```bash
git add ui/litellm-dashboard/src/components/prompts/prompt_editor_view/
git commit -m "feat(ui): add environment selector to prompt editor"
```

---

### Task 10: Add Environment and Created By to Prompt Detail Page

**Files:**
- Modify: `ui/litellm-dashboard/src/components/prompts/prompt_info.tsx:186-234`

- [ ] **Step 1: Add Environment and Created By cards to the Overview tab**

In `ui/litellm-dashboard/src/components/prompts/prompt_info.tsx`, in the Overview TabPanel's Grid (around line 188-223), add two new Cards after the "Prompt Type" card:

```tsx
              <Card>
                <Text>Environment</Text>
                <div className="mt-2">
                  <Title>{promptData.environment || "development"}</Title>
                  <Badge
                    color={
                      promptData.environment === "production"
                        ? "red"
                        : promptData.environment === "staging"
                        ? "yellow"
                        : "green"
                    }
                    className="mt-1"
                  >
                    {promptData.environment || "development"}
                  </Badge>
                </div>
              </Card>

              <Card>
                <Text>Created By</Text>
                <div className="mt-2">
                  <Title className="text-sm">{promptData.created_by || "-"}</Title>
                </div>
              </Card>
```

- [ ] **Step 2: Commit**

```bash
git add ui/litellm-dashboard/src/components/prompts/prompt_info.tsx
git commit -m "feat(ui): show environment and created_by on prompt detail page"
```

---

### Task 11: Final Integration Test and Verification

**Files:**
- All modified files

- [ ] **Step 1: Run all prompt unit tests**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/proxy/prompts/ -v`
Expected: All tests PASS.

- [ ] **Step 2: Run broader unit test suite to check for regressions**

Run: `cd /media/talha/work/ltw/litellm && python -m pytest tests/test_litellm/ -x --timeout=60 -q 2>&1 | tail -20`
Expected: No new failures.

- [ ] **Step 3: Verify UI builds without TypeScript errors**

Run: `cd /media/talha/work/ltw/litellm/ui/litellm-dashboard && npm run build 2>&1 | tail -20`
Expected: Build succeeds.

- [ ] **Step 4: Final commit (if any fixes needed)**

```bash
git add -A
git commit -m "fix: address integration issues from environment and user tracking feature"
```
