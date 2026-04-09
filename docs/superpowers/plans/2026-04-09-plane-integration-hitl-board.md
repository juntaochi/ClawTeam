# Plane Integration + HITL Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate Plane (self-hosted) as a bidirectional presentation and operation layer on top of ClawTeam's file-based task system, enabling humans and agents to create/approve/reject tasks through a Jira-like board with Human-in-the-Loop workflows.

**Architecture:** File system remains SOT. A `PlaneClient` wraps Plane's REST API. A `PlaneSyncEngine` handles bidirectional sync between file-based `TaskItem`s and Plane work items, triggered by ClawTeam event bus hooks (file→Plane) and a Plane webhook receiver (Plane→file). HITL flows use Plane states (e.g. "Awaiting Approval") and comments to drive approval workflows. The sync engine stores mapping metadata (`plane_issue_id`, `plane_state_id`) in `TaskItem.metadata`.

**Tech Stack:** Python 3.10+, `httpx` (async HTTP client for Plane API), ClawTeam event bus, Plane self-hosted (Docker Compose), existing `BaseTaskStore` / `FileTaskStore`.

---

## File Structure

```
clawteam/plane/
├── __init__.py           # Public API: PlaneClient, PlaneSyncEngine
├── client.py             # Low-level Plane REST API wrapper
├── models.py             # Pydantic models for Plane API objects
├── sync.py               # Bidirectional sync engine (file ↔ Plane)
├── webhook.py            # Webhook receiver (HTTP handler for Plane → file)
├── config.py             # Plane-specific config (url, api_key, workspace, project)
└── mapping.py            # State/priority/status mapping between ClawTeam ↔ Plane

clawteam/cli/commands.py  # Add `plane` command group (setup, sync, webhook)
clawteam/config.py        # Add PlaneConfig nested model
clawteam/store/base.py    # (no changes — metadata dict already supports plane fields)

tests/
├── test_plane_client.py    # Unit tests for API client (mocked HTTP)
├── test_plane_models.py    # Model serialization tests
├── test_plane_sync.py      # Sync engine tests
├── test_plane_webhook.py   # Webhook handler tests
├── test_plane_mapping.py   # Mapping logic tests
└── test_plane_config.py    # Config validation tests

scripts/
└── plane-docker-setup.sh  # One-command Plane self-hosted setup script
```

---

### Task 1: Plane Docker Setup Script

**Files:**
- Create: `scripts/plane-docker-setup.sh`

This task creates a script that deploys Plane locally for development.

- [ ] **Step 1: Write the setup script**

```bash
#!/usr/bin/env bash
# plane-docker-setup.sh — Deploy Plane self-hosted for ClawTeam development
set -euo pipefail

PLANE_DIR="${PLANE_DIR:-$HOME/plane-selfhost}"
PLANE_PORT="${PLANE_PORT:-8082}"

echo "==> Setting up Plane self-hosted in $PLANE_DIR (port $PLANE_PORT)"

if [ -d "$PLANE_DIR/plane-app" ]; then
    echo "Plane already installed at $PLANE_DIR/plane-app"
    echo "To restart: cd $PLANE_DIR/plane-app && docker compose up -d"
    exit 0
fi

mkdir -p "$PLANE_DIR"
cd "$PLANE_DIR"

curl -fsSL -o setup.sh https://github.com/makeplane/plane/releases/latest/download/setup.sh
chmod +x setup.sh

echo ""
echo "==> Running Plane installer..."
echo "    When prompted:"
echo "    - Choose option 1 (Install)"
echo "    - Use default domain (localhost)"
echo "    - Set HTTP port to $PLANE_PORT"
echo ""

./setup.sh

echo ""
echo "==> Plane setup complete!"
echo "    Access: http://localhost:$PLANE_PORT"
echo "    1. Create an account"
echo "    2. Create a workspace"
echo "    3. Go to Settings > API Tokens > Create API Token"
echo "    4. Run: clawteam plane setup --url http://localhost:$PLANE_PORT --api-key <token>"
```

- [ ] **Step 2: Make executable and verify syntax**

Run: `chmod +x scripts/plane-docker-setup.sh && bash -n scripts/plane-docker-setup.sh`
Expected: No syntax errors

- [ ] **Step 3: Commit**

```bash
git add scripts/plane-docker-setup.sh
git commit -m "feat(plane): add Docker self-hosted setup script"
```

---

### Task 2: Plane Configuration Model

**Files:**
- Create: `clawteam/plane/__init__.py`
- Create: `clawteam/plane/config.py`
- Modify: `clawteam/config.py:50-69` (add `plane` field to `ClawTeamConfig`)
- Test: `tests/test_plane_config.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_plane_config.py
from __future__ import annotations

from pathlib import Path

import pytest

from clawteam.plane.config import PlaneConfig, load_plane_config, save_plane_config


def test_plane_config_defaults():
    cfg = PlaneConfig()
    assert cfg.url == ""
    assert cfg.api_key == ""
    assert cfg.workspace_slug == ""
    assert cfg.project_id == ""
    assert cfg.sync_enabled is False
    assert cfg.webhook_secret == ""
    assert cfg.webhook_port == 9091


def test_plane_config_from_dict():
    cfg = PlaneConfig(
        url="http://localhost:8082",
        api_key="pl_test_key",
        workspace_slug="my-workspace",
        project_id="abc123",
        sync_enabled=True,
    )
    assert cfg.url == "http://localhost:8082"
    assert cfg.api_key == "pl_test_key"
    assert cfg.workspace_slug == "my-workspace"
    assert cfg.project_id == "abc123"
    assert cfg.sync_enabled is True


def test_plane_config_roundtrip(monkeypatch, tmp_path: Path):
    monkeypatch.setenv("CLAWTEAM_DATA_DIR", str(tmp_path))
    cfg = PlaneConfig(
        url="http://localhost:8082",
        api_key="pl_test_key",
        workspace_slug="ws",
        project_id="proj1",
    )
    save_plane_config(cfg)
    loaded = load_plane_config()
    assert loaded.url == cfg.url
    assert loaded.api_key == cfg.api_key
    assert loaded.workspace_slug == cfg.workspace_slug
    assert loaded.project_id == cfg.project_id


def test_plane_config_in_clawteam_config():
    from clawteam.config import ClawTeamConfig

    cfg = ClawTeamConfig()
    assert hasattr(cfg, "plane")
    assert isinstance(cfg.plane, PlaneConfig)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_config.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'clawteam.plane'`

- [ ] **Step 3: Create plane package init**

```python
# clawteam/plane/__init__.py
"""Plane integration for ClawTeam — bidirectional sync with Plane project management."""

from __future__ import annotations
```

- [ ] **Step 4: Write PlaneConfig model**

```python
# clawteam/plane/config.py
"""Plane integration configuration."""

from __future__ import annotations

import json
from pathlib import Path

from pydantic import BaseModel, Field

from clawteam.team.models import get_data_dir


class PlaneConfig(BaseModel):
    """Configuration for Plane integration."""

    url: str = ""  # e.g. "http://localhost:8082"
    api_key: str = ""
    workspace_slug: str = ""
    project_id: str = ""
    sync_enabled: bool = False
    webhook_secret: str = ""
    webhook_port: int = 9091
    state_mapping: dict[str, str] = Field(default_factory=dict)
    # Maps ClawTeam status -> Plane state name (customizable per project)
    # Defaults populated on first sync if empty


def _plane_config_path() -> Path:
    return get_data_dir() / "plane-config.json"


def load_plane_config() -> PlaneConfig:
    """Load Plane config from data dir. Returns defaults if not found."""
    p = _plane_config_path()
    if not p.exists():
        return PlaneConfig()
    try:
        data = json.loads(p.read_text(encoding="utf-8"))
        return PlaneConfig.model_validate(data)
    except Exception:
        return PlaneConfig()


def save_plane_config(cfg: PlaneConfig) -> None:
    """Persist Plane config to data dir."""
    from clawteam.fileutil import atomic_write_text

    p = _plane_config_path()
    p.parent.mkdir(parents=True, exist_ok=True)
    atomic_write_text(p, cfg.model_dump_json(indent=2))
```

- [ ] **Step 5: Add `plane` field to ClawTeamConfig**

In `clawteam/config.py`, add import and field:

```python
# At the top, add import:
from clawteam.plane.config import PlaneConfig

# In ClawTeamConfig class, add field after 'plugins':
    plane: PlaneConfig = Field(default_factory=PlaneConfig)
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_config.py -v`
Expected: All 4 tests PASS

- [ ] **Step 7: Commit**

```bash
git add clawteam/plane/__init__.py clawteam/plane/config.py clawteam/config.py tests/test_plane_config.py
git commit -m "feat(plane): add PlaneConfig model with persistence"
```

---

### Task 3: Plane API Data Models

**Files:**
- Create: `clawteam/plane/models.py`
- Test: `tests/test_plane_models.py`

These are Pydantic models that map Plane REST API responses so we have typed access.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_plane_models.py
from __future__ import annotations

from clawteam.plane.models import (
    PlaneProject,
    PlaneState,
    PlaneWorkItem,
    PlaneComment,
    PlaneWorkspace,
)


def test_work_item_from_api_response():
    """Plane API returns snake_case JSON — model should parse it."""
    data = {
        "id": "abc-123",
        "name": "Fix login bug",
        "description_html": "<p>Details here</p>",
        "state": "state-uuid-1",
        "priority": "high",
        "assignees": ["user-1"],
        "labels": ["label-1"],
        "created_at": "2026-04-09T10:00:00Z",
        "updated_at": "2026-04-09T11:00:00Z",
        "sequence_id": 42,
        "project": "proj-uuid",
    }
    item = PlaneWorkItem.model_validate(data)
    assert item.id == "abc-123"
    assert item.name == "Fix login bug"
    assert item.state == "state-uuid-1"
    assert item.priority == "high"
    assert item.assignees == ["user-1"]
    assert item.sequence_id == 42


def test_state_from_api_response():
    data = {
        "id": "state-uuid-1",
        "name": "In Progress",
        "group": "started",
        "color": "#f59e0b",
        "sequence": 2,
    }
    state = PlaneState.model_validate(data)
    assert state.id == "state-uuid-1"
    assert state.name == "In Progress"
    assert state.group == "started"


def test_comment_from_api_response():
    data = {
        "id": "comment-1",
        "comment_html": "<p>Approved</p>",
        "actor_detail": {"display_name": "Alice"},
        "created_at": "2026-04-09T12:00:00Z",
    }
    comment = PlaneComment.model_validate(data)
    assert comment.id == "comment-1"
    assert comment.actor_detail["display_name"] == "Alice"


def test_work_item_extra_fields_ignored():
    """API may return fields we don't model — should not fail."""
    data = {
        "id": "abc-123",
        "name": "task",
        "state": "s1",
        "priority": "none",
        "unknown_future_field": True,
    }
    item = PlaneWorkItem.model_validate(data)
    assert item.id == "abc-123"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_models.py -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Write the models**

```python
# clawteam/plane/models.py
"""Pydantic models for Plane REST API objects."""

from __future__ import annotations

from typing import Any

from pydantic import BaseModel, Field


class PlaneWorkspace(BaseModel):
    """Plane workspace (top-level org container)."""

    model_config = {"extra": "ignore"}

    id: str = ""
    name: str = ""
    slug: str = ""


class PlaneProject(BaseModel):
    """Plane project within a workspace."""

    model_config = {"extra": "ignore"}

    id: str = ""
    name: str = ""
    identifier: str = ""  # e.g. "CLAW"
    description: str = ""


class PlaneState(BaseModel):
    """Workflow state in a Plane project."""

    model_config = {"extra": "ignore"}

    id: str = ""
    name: str = ""
    group: str = ""  # "backlog" | "unstarted" | "started" | "completed" | "cancelled"
    color: str = ""
    sequence: float = 0


class PlaneWorkItem(BaseModel):
    """A work item (issue) in Plane."""

    model_config = {"extra": "ignore"}

    id: str = ""
    name: str = ""
    description_html: str = ""
    state: str = ""  # state UUID
    priority: str = "none"  # "urgent" | "high" | "medium" | "low" | "none"
    assignees: list[str] = Field(default_factory=list)
    labels: list[str] = Field(default_factory=list)
    created_at: str = ""
    updated_at: str = ""
    sequence_id: int = 0
    project: str = ""


class PlaneComment(BaseModel):
    """A comment on a Plane work item."""

    model_config = {"extra": "ignore"}

    id: str = ""
    comment_html: str = ""
    actor_detail: dict[str, Any] = Field(default_factory=dict)
    created_at: str = ""
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_models.py -v`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add clawteam/plane/models.py tests/test_plane_models.py
git commit -m "feat(plane): add Pydantic models for Plane API objects"
```

---

### Task 4: Plane REST API Client

**Files:**
- Create: `clawteam/plane/client.py`
- Modify: `pyproject.toml:21-28` (add `httpx` dependency)
- Test: `tests/test_plane_client.py`

- [ ] **Step 1: Add httpx dependency**

In `pyproject.toml`, add to `[project.optional-dependencies]`:

```toml
[project.optional-dependencies]
dev = [
    "pytest>=9.0.0,<10.0.0",
    "ruff>=0.1.0",
]
p2p = [
    "pyzmq>=25.0.0,<27.0.0",
]
plane = [
    "httpx>=0.27.0,<1.0.0",
]
```

- [ ] **Step 2: Install the plane extra**

Run: `cd /home/jac/repos/ClawTeam && pip install -e ".[plane,dev]"`

- [ ] **Step 3: Write the failing tests**

```python
# tests/test_plane_client.py
from __future__ import annotations

import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from clawteam.plane.client import PlaneClient
from clawteam.plane.models import PlaneState, PlaneWorkItem


@pytest.fixture
def client():
    return PlaneClient(
        base_url="http://localhost:8082",
        api_key="pl_test_key",
        workspace_slug="test-ws",
    )


def test_client_init(client: PlaneClient):
    assert client.base_url == "http://localhost:8082"
    assert client.workspace_slug == "test-ws"


def test_client_headers(client: PlaneClient):
    headers = client._headers()
    assert headers["X-API-Key"] == "pl_test_key"
    assert headers["Content-Type"] == "application/json"


def test_client_build_url(client: PlaneClient):
    url = client._url("projects", "proj1", "work-items")
    assert url == "http://localhost:8082/api/v1/workspaces/test-ws/projects/proj1/work-items/"


def test_work_item_to_plane_payload():
    """Verify our payload builder produces correct Plane API fields."""
    from clawteam.plane.client import _task_to_plane_payload
    from clawteam.team.models import TaskItem, TaskPriority, TaskStatus

    task = TaskItem(
        id="abc123",
        subject="Fix the bug",
        description="Detailed description",
        status=TaskStatus.in_progress,
        priority=TaskPriority.high,
        owner="worker1",
    )
    payload = _task_to_plane_payload(task, state_id="state-uuid-started")
    assert payload["name"] == "Fix the bug"
    assert payload["description_html"] == "<p>Detailed description</p>"
    assert payload["state"] == "state-uuid-started"
    assert payload["priority"] == "high"
```

- [ ] **Step 4: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_client.py -v`
Expected: FAIL — `ImportError`

- [ ] **Step 5: Write the PlaneClient**

```python
# clawteam/plane/client.py
"""Low-level Plane REST API client using httpx."""

from __future__ import annotations

from typing import Any

import httpx

from clawteam.plane.models import (
    PlaneComment,
    PlaneProject,
    PlaneState,
    PlaneWorkItem,
    PlaneWorkspace,
)
from clawteam.team.models import TaskItem


# ── Priority mapping ─────────────────────────────────────────────────

_CLAWTEAM_TO_PLANE_PRIORITY = {
    "urgent": "urgent",
    "high": "high",
    "medium": "medium",
    "low": "low",
}

_PLANE_TO_CLAWTEAM_PRIORITY = {v: k for k, v in _CLAWTEAM_TO_PLANE_PRIORITY.items()}
_PLANE_TO_CLAWTEAM_PRIORITY["none"] = "medium"


def _task_to_plane_payload(task: TaskItem, state_id: str) -> dict[str, Any]:
    """Convert a ClawTeam TaskItem to a Plane work item creation/update payload."""
    desc_html = f"<p>{task.description}</p>" if task.description else ""
    return {
        "name": task.subject,
        "description_html": desc_html,
        "state": state_id,
        "priority": _CLAWTEAM_TO_PLANE_PRIORITY.get(task.priority.value, "medium"),
    }


class PlaneClient:
    """Synchronous HTTP client for the Plane REST API."""

    def __init__(self, base_url: str, api_key: str, workspace_slug: str):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.workspace_slug = workspace_slug
        self._http = httpx.Client(timeout=30.0)

    def _headers(self) -> dict[str, str]:
        return {
            "X-API-Key": self.api_key,
            "Content-Type": "application/json",
        }

    def _url(self, *parts: str) -> str:
        path = "/".join(parts)
        return f"{self.base_url}/api/v1/workspaces/{self.workspace_slug}/{path}/"

    def _get(self, *parts: str, params: dict | None = None) -> Any:
        resp = self._http.get(self._url(*parts), headers=self._headers(), params=params)
        resp.raise_for_status()
        return resp.json()

    def _post(self, *parts: str, json_data: dict | None = None) -> Any:
        resp = self._http.post(self._url(*parts), headers=self._headers(), json=json_data)
        resp.raise_for_status()
        return resp.json()

    def _patch(self, *parts: str, json_data: dict | None = None) -> Any:
        resp = self._http.patch(self._url(*parts), headers=self._headers(), json=json_data)
        resp.raise_for_status()
        return resp.json()

    def close(self) -> None:
        self._http.close()

    # ── Projects ─────────────────────────────────────────────────────

    def list_projects(self) -> list[PlaneProject]:
        data = self._get("projects")
        results = data.get("results", data) if isinstance(data, dict) else data
        return [PlaneProject.model_validate(p) for p in results]

    def get_project(self, project_id: str) -> PlaneProject:
        data = self._get("projects", project_id)
        return PlaneProject.model_validate(data)

    # ── States ───────────────────────────────────────────────────────

    def list_states(self, project_id: str) -> list[PlaneState]:
        data = self._get("projects", project_id, "states")
        results = data.get("results", data) if isinstance(data, dict) else data
        return [PlaneState.model_validate(s) for s in results]

    def create_state(self, project_id: str, name: str, group: str, color: str = "#6b7280") -> PlaneState:
        data = self._post("projects", project_id, "states", json_data={
            "name": name,
            "group": group,
            "color": color,
        })
        return PlaneState.model_validate(data)

    # ── Work Items ───────────────────────────────────────────────────

    def list_work_items(self, project_id: str) -> list[PlaneWorkItem]:
        data = self._get("projects", project_id, "work-items")
        results = data.get("results", data) if isinstance(data, dict) else data
        return [PlaneWorkItem.model_validate(i) for i in results]

    def get_work_item(self, project_id: str, item_id: str) -> PlaneWorkItem:
        data = self._get("projects", project_id, "work-items", item_id)
        return PlaneWorkItem.model_validate(data)

    def create_work_item(self, project_id: str, payload: dict) -> PlaneWorkItem:
        data = self._post("projects", project_id, "work-items", json_data=payload)
        return PlaneWorkItem.model_validate(data)

    def update_work_item(self, project_id: str, item_id: str, payload: dict) -> PlaneWorkItem:
        data = self._patch("projects", project_id, "work-items", item_id, json_data=payload)
        return PlaneWorkItem.model_validate(data)

    # ── Comments ─────────────────────────────────────────────────────

    def list_comments(self, project_id: str, item_id: str) -> list[PlaneComment]:
        data = self._get("projects", project_id, "work-items", item_id, "comments")
        results = data.get("results", data) if isinstance(data, dict) else data
        return [PlaneComment.model_validate(c) for c in results]

    def create_comment(self, project_id: str, item_id: str, html: str) -> PlaneComment:
        data = self._post(
            "projects", project_id, "work-items", item_id, "comments",
            json_data={"comment_html": html},
        )
        return PlaneComment.model_validate(data)
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_client.py -v`
Expected: All 4 tests PASS

- [ ] **Step 7: Commit**

```bash
git add clawteam/plane/client.py tests/test_plane_client.py pyproject.toml
git commit -m "feat(plane): add PlaneClient REST API wrapper"
```

---

### Task 5: Status/State Mapping

**Files:**
- Create: `clawteam/plane/mapping.py`
- Test: `tests/test_plane_mapping.py`

This maps ClawTeam's 4 task statuses to Plane's state group system, and ensures the Plane project has the required states.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_plane_mapping.py
from __future__ import annotations

from clawteam.plane.mapping import (
    clawteam_status_to_plane_group,
    plane_group_to_clawteam_status,
    DEFAULT_STATE_NAMES,
    resolve_state_id,
)
from clawteam.plane.models import PlaneState
from clawteam.team.models import TaskStatus


def test_clawteam_to_plane_group():
    assert clawteam_status_to_plane_group(TaskStatus.pending) == "unstarted"
    assert clawteam_status_to_plane_group(TaskStatus.in_progress) == "started"
    assert clawteam_status_to_plane_group(TaskStatus.completed) == "completed"
    assert clawteam_status_to_plane_group(TaskStatus.blocked) == "backlog"


def test_plane_group_to_clawteam():
    assert plane_group_to_clawteam_status("unstarted") == TaskStatus.pending
    assert plane_group_to_clawteam_status("started") == TaskStatus.in_progress
    assert plane_group_to_clawteam_status("completed") == TaskStatus.completed
    assert plane_group_to_clawteam_status("backlog") == TaskStatus.blocked
    assert plane_group_to_clawteam_status("cancelled") == TaskStatus.completed


def test_default_state_names():
    assert "pending" in DEFAULT_STATE_NAMES
    assert "in_progress" in DEFAULT_STATE_NAMES
    assert "completed" in DEFAULT_STATE_NAMES
    assert "blocked" in DEFAULT_STATE_NAMES
    assert "awaiting_approval" in DEFAULT_STATE_NAMES


def test_resolve_state_id():
    states = [
        PlaneState(id="s1", name="Pending", group="unstarted"),
        PlaneState(id="s2", name="In Progress", group="started"),
        PlaneState(id="s3", name="Done", group="completed"),
        PlaneState(id="s4", name="Blocked", group="backlog"),
    ]
    assert resolve_state_id(states, TaskStatus.pending) == "s1"
    assert resolve_state_id(states, TaskStatus.in_progress) == "s2"
    assert resolve_state_id(states, TaskStatus.completed) == "s3"
    assert resolve_state_id(states, TaskStatus.blocked) == "s4"


def test_resolve_state_id_fallback_by_group():
    """When no name match, fall back to group match."""
    states = [
        PlaneState(id="s1", name="Todo", group="unstarted"),
        PlaneState(id="s2", name="Working", group="started"),
    ]
    assert resolve_state_id(states, TaskStatus.pending) == "s1"
    assert resolve_state_id(states, TaskStatus.in_progress) == "s2"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_mapping.py -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Write the mapping module**

```python
# clawteam/plane/mapping.py
"""Bidirectional mapping between ClawTeam task statuses and Plane workflow states."""

from __future__ import annotations

from clawteam.plane.models import PlaneState
from clawteam.team.models import TaskStatus


# ── ClawTeam status ↔ Plane state group ──────────────────────────────

_STATUS_TO_GROUP: dict[TaskStatus, str] = {
    TaskStatus.pending: "unstarted",
    TaskStatus.in_progress: "started",
    TaskStatus.completed: "completed",
    TaskStatus.blocked: "backlog",
}

_GROUP_TO_STATUS: dict[str, TaskStatus] = {
    "unstarted": TaskStatus.pending,
    "started": TaskStatus.in_progress,
    "completed": TaskStatus.completed,
    "backlog": TaskStatus.blocked,
    "cancelled": TaskStatus.completed,
}


def clawteam_status_to_plane_group(status: TaskStatus) -> str:
    return _STATUS_TO_GROUP[status]


def plane_group_to_clawteam_status(group: str) -> TaskStatus:
    return _GROUP_TO_STATUS.get(group, TaskStatus.pending)


# ── Default state names we create in Plane if missing ────────────────

DEFAULT_STATE_NAMES: dict[str, tuple[str, str]] = {
    # clawteam_key: (plane_display_name, plane_group)
    "pending": ("Pending", "unstarted"),
    "in_progress": ("In Progress", "started"),
    "completed": ("Done", "completed"),
    "blocked": ("Blocked", "backlog"),
    "awaiting_approval": ("Awaiting Approval", "unstarted"),
}


# ── Preferred name match, then group fallback ────────────────────────

_STATUS_TO_PREFERRED_NAME: dict[TaskStatus, list[str]] = {
    TaskStatus.pending: ["Pending", "Todo", "To Do"],
    TaskStatus.in_progress: ["In Progress", "Working", "Active"],
    TaskStatus.completed: ["Done", "Completed", "Closed"],
    TaskStatus.blocked: ["Blocked", "On Hold"],
}


def resolve_state_id(states: list[PlaneState], status: TaskStatus) -> str:
    """Find the best Plane state ID for a ClawTeam status.

    1. Try matching by preferred display name (case-insensitive).
    2. Fall back to first state in the matching group.
    3. Return empty string if nothing matches.
    """
    preferred = _STATUS_TO_PREFERRED_NAME.get(status, [])
    for name in preferred:
        for s in states:
            if s.name.lower() == name.lower():
                return s.id

    target_group = _STATUS_TO_GROUP.get(status, "")
    for s in states:
        if s.group == target_group:
            return s.id

    return ""
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_mapping.py -v`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add clawteam/plane/mapping.py tests/test_plane_mapping.py
git commit -m "feat(plane): add ClawTeam ↔ Plane status/state mapping"
```

---

### Task 6: Bidirectional Sync Engine (File → Plane)

**Files:**
- Create: `clawteam/plane/sync.py`
- Test: `tests/test_plane_sync.py`

The sync engine pushes file-based task changes to Plane and pulls Plane changes back. This task covers the file→Plane direction. Task metadata stores `plane_issue_id` and `plane_updated_at` to track sync state.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_plane_sync.py
from __future__ import annotations

from pathlib import Path
from unittest.mock import MagicMock, patch

import pytest

from clawteam.plane.config import PlaneConfig
from clawteam.plane.models import PlaneState, PlaneWorkItem
from clawteam.plane.sync import PlaneSyncEngine
from clawteam.team.manager import TeamManager
from clawteam.team.models import TaskItem, TaskPriority, TaskStatus


@pytest.fixture
def setup_team(monkeypatch, tmp_path: Path):
    monkeypatch.setenv("CLAWTEAM_DATA_DIR", str(tmp_path))
    TeamManager.create_team(
        name="demo", leader_name="leader", leader_id="leader001",
    )
    return "demo"


@pytest.fixture
def plane_config():
    return PlaneConfig(
        url="http://localhost:8082",
        api_key="test-key",
        workspace_slug="test-ws",
        project_id="proj-1",
        sync_enabled=True,
    )


@pytest.fixture
def mock_client():
    client = MagicMock()
    client.list_states.return_value = [
        PlaneState(id="s-pending", name="Pending", group="unstarted"),
        PlaneState(id="s-progress", name="In Progress", group="started"),
        PlaneState(id="s-done", name="Done", group="completed"),
        PlaneState(id="s-blocked", name="Blocked", group="backlog"),
    ]
    return client


def test_push_new_task_creates_plane_work_item(setup_team, plane_config, mock_client):
    mock_client.create_work_item.return_value = PlaneWorkItem(
        id="plane-issue-1", name="Build feature", state="s-pending",
    )

    engine = PlaneSyncEngine(plane_config, client=mock_client)
    from clawteam.store.file import FileTaskStore
    store = FileTaskStore(setup_team)
    task = store.create(subject="Build feature", description="Details")

    engine.push_task(setup_team, task)

    mock_client.create_work_item.assert_called_once()
    # Task metadata should now have plane_issue_id
    updated = store.get(task.id)
    assert updated is not None
    assert updated.metadata["plane_issue_id"] == "plane-issue-1"


def test_push_existing_task_updates_plane_work_item(setup_team, plane_config, mock_client):
    mock_client.update_work_item.return_value = PlaneWorkItem(
        id="plane-issue-1", name="Build feature v2", state="s-progress",
    )

    engine = PlaneSyncEngine(plane_config, client=mock_client)
    from clawteam.store.file import FileTaskStore
    store = FileTaskStore(setup_team)
    task = store.create(
        subject="Build feature",
        metadata={"plane_issue_id": "plane-issue-1"},
    )
    store.update(task.id, subject="Build feature v2", status=TaskStatus.in_progress, caller="test")

    updated_task = store.get(task.id)
    engine.push_task(setup_team, updated_task)

    mock_client.update_work_item.assert_called_once()
    call_args = mock_client.update_work_item.call_args
    assert call_args[0][0] == "proj-1"  # project_id
    assert call_args[0][1] == "plane-issue-1"  # item_id


def test_push_skips_when_sync_disabled(setup_team, mock_client):
    config = PlaneConfig(sync_enabled=False)
    engine = PlaneSyncEngine(config, client=mock_client)
    task = TaskItem(subject="test")

    engine.push_task(setup_team, task)

    mock_client.create_work_item.assert_not_called()
    mock_client.update_work_item.assert_not_called()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_sync.py -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Write the sync engine**

```python
# clawteam/plane/sync.py
"""Bidirectional sync engine between ClawTeam file store and Plane."""

from __future__ import annotations

import logging
from typing import TYPE_CHECKING

from clawteam.plane.client import PlaneClient, _task_to_plane_payload
from clawteam.plane.config import PlaneConfig
from clawteam.plane.mapping import (
    plane_group_to_clawteam_status,
    resolve_state_id,
)
from clawteam.plane.models import PlaneState, PlaneWorkItem
from clawteam.team.models import TaskItem, TaskPriority, TaskStatus

if TYPE_CHECKING:
    from clawteam.store.base import BaseTaskStore

log = logging.getLogger(__name__)


class PlaneSyncEngine:
    """Syncs ClawTeam tasks with Plane work items."""

    def __init__(self, config: PlaneConfig, client: PlaneClient | None = None):
        self.config = config
        self._client = client or (
            PlaneClient(config.url, config.api_key, config.workspace_slug)
            if config.url and config.api_key
            else None
        )
        self._states: list[PlaneState] | None = None

    def _get_states(self) -> list[PlaneState]:
        if self._states is None:
            self._states = self._client.list_states(self.config.project_id)
        return self._states

    def _resolve_state(self, status: TaskStatus) -> str:
        return resolve_state_id(self._get_states(), status)

    # ── File → Plane ─────────────────────────────────────────────────

    def push_task(self, team_name: str, task: TaskItem) -> None:
        """Push a single task to Plane. Creates or updates based on metadata."""
        if not self.config.sync_enabled or not self._client:
            return

        state_id = self._resolve_state(task.status)
        payload = _task_to_plane_payload(task, state_id)
        plane_id = task.metadata.get("plane_issue_id", "")

        if plane_id:
            self._client.update_work_item(self.config.project_id, plane_id, payload)
            log.info("Updated Plane work item %s for task %s", plane_id, task.id)
        else:
            item = self._client.create_work_item(self.config.project_id, payload)
            # Write plane_issue_id back to file store
            from clawteam.store.file import FileTaskStore

            store = FileTaskStore(team_name)
            store.update(task.id, metadata={"plane_issue_id": item.id})
            log.info("Created Plane work item %s for task %s", item.id, task.id)

    def push_all(self, team_name: str) -> int:
        """Push all tasks in a team to Plane. Returns count of items synced."""
        if not self.config.sync_enabled or not self._client:
            return 0

        from clawteam.store.file import FileTaskStore

        store = FileTaskStore(team_name)
        tasks = store.list_tasks()
        count = 0
        for task in tasks:
            try:
                self.push_task(team_name, task)
                count += 1
            except Exception as exc:
                log.warning("Failed to push task %s: %s", task.id, exc)
        return count

    # ── Plane → File ─────────────────────────────────────────────────

    def pull_all(self, team_name: str) -> int:
        """Pull work items from Plane and sync to file store.

        Creates new tasks for unlinked Plane items.
        Updates existing tasks when Plane state differs.
        Returns count of items synced.
        """
        if not self.config.sync_enabled or not self._client:
            return 0

        from clawteam.store.file import FileTaskStore

        store = FileTaskStore(team_name)
        existing = store.list_tasks()
        # Build reverse index: plane_issue_id -> task
        plane_to_task: dict[str, TaskItem] = {}
        for task in existing:
            pid = task.metadata.get("plane_issue_id", "")
            if pid:
                plane_to_task[pid] = task

        items = self._client.list_work_items(self.config.project_id)
        states = self._get_states()
        state_map = {s.id: s for s in states}
        count = 0

        for item in items:
            try:
                state = state_map.get(item.state)
                clawteam_status = (
                    plane_group_to_clawteam_status(state.group)
                    if state
                    else TaskStatus.pending
                )

                if item.id in plane_to_task:
                    # Update existing task if status changed
                    task = plane_to_task[item.id]
                    if task.status != clawteam_status or task.subject != item.name:
                        store.update(
                            task.id,
                            status=clawteam_status,
                            subject=item.name,
                            force=True,
                        )
                        count += 1
                else:
                    # Create new task from Plane item
                    priority_str = item.priority if item.priority != "none" else "medium"
                    try:
                        priority = TaskPriority(priority_str)
                    except ValueError:
                        priority = TaskPriority.medium
                    store.create(
                        subject=item.name,
                        description=item.description_html,
                        metadata={"plane_issue_id": item.id},
                        priority=priority,
                    )
                    count += 1
            except Exception as exc:
                log.warning("Failed to pull item %s: %s", item.id, exc)

        return count
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_sync.py -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git add clawteam/plane/sync.py tests/test_plane_sync.py
git commit -m "feat(plane): add bidirectional PlaneSyncEngine"
```

---

### Task 7: Plane Webhook Receiver (Plane → File)

**Files:**
- Create: `clawteam/plane/webhook.py`
- Test: `tests/test_plane_webhook.py`

Plane sends webhook POST requests when work items change. This handler receives them and updates the file store. It also implements HITL: when a Plane state changes to "Awaiting Approval" or a comment contains "APPROVED"/"REJECTED", it sends the corresponding message through ClawTeam's mailbox.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_plane_webhook.py
from __future__ import annotations

import hashlib
import hmac
import json
from pathlib import Path
from unittest.mock import MagicMock

import pytest

from clawteam.plane.config import PlaneConfig
from clawteam.plane.webhook import (
    PlaneWebhookHandler,
    _verify_signature,
    _handle_work_item_event,
)
from clawteam.team.manager import TeamManager
from clawteam.team.models import TaskStatus


@pytest.fixture
def setup_team(monkeypatch, tmp_path: Path):
    monkeypatch.setenv("CLAWTEAM_DATA_DIR", str(tmp_path))
    TeamManager.create_team(
        name="demo", leader_name="leader", leader_id="leader001",
    )
    return "demo"


def test_verify_signature_valid():
    secret = "webhook-secret-123"
    body = b'{"event": "issue.updated"}'
    sig = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    assert _verify_signature(body, sig, secret) is True


def test_verify_signature_invalid():
    assert _verify_signature(b"body", "bad-sig", "secret") is False


def test_handle_work_item_created_creates_task(setup_team):
    from clawteam.store.file import FileTaskStore

    config = PlaneConfig(
        url="http://localhost:8082",
        api_key="key",
        workspace_slug="ws",
        project_id="proj-1",
        sync_enabled=True,
    )
    payload = {
        "event": "issue",
        "action": "created",
        "data": {
            "id": "plane-new-1",
            "name": "Human-created task",
            "description_html": "<p>Do this</p>",
            "state": "state-1",
            "priority": "high",
        },
    }

    mock_states = {
        "state-1": MagicMock(group="unstarted"),
    }

    result = _handle_work_item_event(payload, config, setup_team, mock_states)

    assert result["action"] == "created"
    store = FileTaskStore(setup_team)
    tasks = store.list_tasks()
    assert len(tasks) == 1
    assert tasks[0].subject == "Human-created task"
    assert tasks[0].metadata["plane_issue_id"] == "plane-new-1"


def test_handle_work_item_updated_changes_status(setup_team):
    from clawteam.store.file import FileTaskStore

    store = FileTaskStore(setup_team)
    task = store.create(
        subject="Existing task",
        metadata={"plane_issue_id": "plane-exist-1"},
    )

    config = PlaneConfig(
        url="http://localhost:8082",
        api_key="key",
        workspace_slug="ws",
        project_id="proj-1",
        sync_enabled=True,
    )
    payload = {
        "event": "issue",
        "action": "updated",
        "data": {
            "id": "plane-exist-1",
            "name": "Existing task updated",
            "state": "state-started",
            "priority": "medium",
        },
    }
    mock_states = {
        "state-started": MagicMock(group="started"),
    }

    result = _handle_work_item_event(payload, config, setup_team, mock_states)

    assert result["action"] == "updated"
    updated = store.get(task.id)
    assert updated.status == TaskStatus.in_progress
    assert updated.subject == "Existing task updated"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_webhook.py -v`
Expected: FAIL — `ImportError`

- [ ] **Step 3: Write the webhook handler**

```python
# clawteam/plane/webhook.py
"""Webhook receiver for Plane → ClawTeam sync and HITL triggers."""

from __future__ import annotations

import hashlib
import hmac
import json
import logging
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from typing import Any

from clawteam.plane.config import PlaneConfig
from clawteam.plane.mapping import plane_group_to_clawteam_status
from clawteam.team.models import MessageType, TaskPriority, TaskStatus

log = logging.getLogger(__name__)


def _verify_signature(body: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 webhook signature."""
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)


def _handle_work_item_event(
    payload: dict,
    config: PlaneConfig,
    team_name: str,
    state_lookup: dict[str, Any],
) -> dict[str, str]:
    """Process a work item webhook event. Returns summary dict."""
    from clawteam.store.file import FileTaskStore

    action = payload.get("action", "")
    data = payload.get("data", {})
    plane_id = data.get("id", "")
    name = data.get("name", "")
    state_uuid = data.get("state", "")
    priority_str = data.get("priority", "medium")

    state_obj = state_lookup.get(state_uuid)
    clawteam_status = (
        plane_group_to_clawteam_status(state_obj.group)
        if state_obj
        else TaskStatus.pending
    )

    store = FileTaskStore(team_name)

    if action == "created":
        # Check if we already have this plane item
        for task in store.list_tasks():
            if task.metadata.get("plane_issue_id") == plane_id:
                return {"action": "skipped", "reason": "already exists"}

        try:
            priority = TaskPriority(priority_str)
        except ValueError:
            priority = TaskPriority.medium

        store.create(
            subject=name,
            description=data.get("description_html", ""),
            priority=priority,
            metadata={"plane_issue_id": plane_id},
        )
        return {"action": "created", "plane_id": plane_id}

    elif action == "updated":
        for task in store.list_tasks():
            if task.metadata.get("plane_issue_id") == plane_id:
                store.update(
                    task.id,
                    subject=name,
                    status=clawteam_status,
                    force=True,
                )
                # HITL: check if state indicates approval needed
                if state_obj and state_obj.group == "unstarted" and "approv" in (getattr(state_obj, "name", "") or "").lower():
                    _send_approval_request(team_name, task.id, name)
                return {"action": "updated", "task_id": task.id}

        return {"action": "skipped", "reason": "no matching task"}

    return {"action": "ignored", "event_action": action}


def _handle_comment_event(
    payload: dict,
    config: PlaneConfig,
    team_name: str,
) -> dict[str, str]:
    """Process a comment webhook for HITL approve/reject."""
    from clawteam.store.file import FileTaskStore

    data = payload.get("data", {})
    comment_html = data.get("comment_html", "")
    issue_id = data.get("issue", "") or data.get("work_item", "")
    actor = data.get("actor_detail", {}).get("display_name", "human")

    comment_lower = comment_html.lower()
    is_approve = "approved" in comment_lower or "approve" in comment_lower or "lgtm" in comment_lower
    is_reject = "rejected" in comment_lower or "reject" in comment_lower

    if not is_approve and not is_reject:
        return {"action": "ignored", "reason": "not an approval comment"}

    store = FileTaskStore(team_name)
    for task in store.list_tasks():
        if task.metadata.get("plane_issue_id") == issue_id:
            if is_approve:
                store.update(task.id, status=TaskStatus.in_progress, force=True)
                _send_hitl_message(
                    team_name, task, "plan_approved", actor, comment_html,
                )
                return {"action": "approved", "task_id": task.id}
            elif is_reject:
                store.update(task.id, status=TaskStatus.blocked, force=True)
                _send_hitl_message(
                    team_name, task, "plan_rejected", actor, comment_html,
                )
                return {"action": "rejected", "task_id": task.id}

    return {"action": "skipped", "reason": "no matching task"}


def _send_approval_request(team_name: str, task_id: str, subject: str) -> None:
    """Send a plan_approval_request message to the team leader."""
    try:
        from clawteam.team.mailbox import MailboxManager
        from clawteam.team.manager import TeamManager

        leader_inbox = TeamManager.get_leader_inbox(team_name)
        if leader_inbox:
            mailbox = MailboxManager(team_name)
            mailbox.send(
                from_agent="plane-webhook",
                to=leader_inbox,
                msg_type=MessageType.plan_approval_request,
                content=f"Task '{subject}' (id={task_id}) requires approval in Plane.",
                summary=subject,
            )
    except Exception as exc:
        log.warning("Failed to send approval request: %s", exc)


def _send_hitl_message(
    team_name: str,
    task: Any,
    msg_type_str: str,
    actor: str,
    comment: str,
) -> None:
    """Send HITL approval/rejection message to the task owner's inbox."""
    try:
        from clawteam.team.mailbox import MailboxManager
        from clawteam.team.manager import TeamManager

        msg_type = MessageType(msg_type_str)
        target = task.owner or TeamManager.get_leader_inbox(team_name) or ""
        if not target:
            return
        mailbox = MailboxManager(team_name)
        mailbox.send(
            from_agent="plane-webhook",
            to=target,
            msg_type=msg_type,
            content=f"{actor}: {comment}",
            summary=task.subject,
            feedback=comment,
        )
    except Exception as exc:
        log.warning("Failed to send HITL message: %s", exc)


class PlaneWebhookHandler(BaseHTTPRequestHandler):
    """HTTP handler for receiving Plane webhooks."""

    config: PlaneConfig
    team_name: str
    state_lookup: dict[str, Any]

    def do_POST(self):
        content_length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_length)

        # Verify signature if secret is configured
        if self.config.webhook_secret:
            sig = self.headers.get("X-Plane-Signature", "")
            if not _verify_signature(body, sig, self.config.webhook_secret):
                self.send_error(401, "Invalid signature")
                return

        try:
            payload = json.loads(body)
        except json.JSONDecodeError:
            self.send_error(400, "Invalid JSON")
            return

        event = payload.get("event", "")
        if event == "issue":
            result = _handle_work_item_event(
                payload, self.config, self.team_name, self.state_lookup,
            )
        elif event == "issue_comment":
            result = _handle_comment_event(payload, self.config, self.team_name)
        else:
            result = {"action": "ignored", "event": event}

        response = json.dumps(result).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(response)))
        self.end_headers()
        self.wfile.write(response)

    def log_message(self, format, *args):
        log.debug(format, *args)


def serve_webhook(
    config: PlaneConfig,
    team_name: str,
    state_lookup: dict[str, Any] | None = None,
    host: str = "0.0.0.0",
) -> None:
    """Start the Plane webhook receiver server."""
    PlaneWebhookHandler.config = config
    PlaneWebhookHandler.team_name = team_name
    PlaneWebhookHandler.state_lookup = state_lookup or {}

    server = ThreadingHTTPServer((host, config.webhook_port), PlaneWebhookHandler)
    log.info("Plane webhook receiver listening on %s:%d", host, config.webhook_port)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_webhook.py -v`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add clawteam/plane/webhook.py tests/test_plane_webhook.py
git commit -m "feat(plane): add webhook receiver with HITL approve/reject flow"
```

---

### Task 8: Event Bus Integration (Auto-sync on Task Changes)

**Files:**
- Modify: `clawteam/plane/__init__.py`
- Test: add to `tests/test_plane_sync.py`

Hook into ClawTeam's event bus so that task creates/updates automatically push to Plane. This makes the file→Plane direction automatic.

- [ ] **Step 1: Write the failing test**

Add to `tests/test_plane_sync.py`:

```python
def test_event_hook_pushes_on_task_update(setup_team, plane_config, mock_client):
    """AfterTaskUpdate event should trigger a push to Plane."""
    mock_client.create_work_item.return_value = PlaneWorkItem(
        id="plane-auto-1", name="Auto task", state="s-pending",
    )
    mock_client.update_work_item.return_value = PlaneWorkItem(
        id="plane-auto-1", name="Auto task", state="s-progress",
    )

    from clawteam.plane import register_sync_hooks
    from clawteam.events.bus import EventBus
    from clawteam.events.types import AfterTaskUpdate

    bus = EventBus()
    engine = PlaneSyncEngine(plane_config, client=mock_client)
    register_sync_hooks(bus, engine, setup_team)

    # Simulate task create + update cycle
    from clawteam.store.file import FileTaskStore
    store = FileTaskStore(setup_team)
    task = store.create(subject="Auto task")
    engine.push_task(setup_team, task)  # initial push

    # Now simulate status update event
    store.update(task.id, status=TaskStatus.in_progress, caller="worker1")
    bus.emit(AfterTaskUpdate(
        team_name=setup_team,
        task_id=task.id,
        old_status="pending",
        new_status="in_progress",
        owner="worker1",
    ))

    # Should have called update_work_item
    assert mock_client.update_work_item.called
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_sync.py::test_event_hook_pushes_on_task_update -v`
Expected: FAIL — `ImportError: cannot import name 'register_sync_hooks'`

- [ ] **Step 3: Implement register_sync_hooks**

Update `clawteam/plane/__init__.py`:

```python
# clawteam/plane/__init__.py
"""Plane integration for ClawTeam — bidirectional sync with Plane project management."""

from __future__ import annotations

import logging
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from clawteam.events.bus import EventBus
    from clawteam.plane.sync import PlaneSyncEngine

log = logging.getLogger(__name__)


def register_sync_hooks(bus: EventBus, engine: PlaneSyncEngine, team_name: str) -> None:
    """Subscribe to task events and auto-push changes to Plane."""
    from clawteam.events.types import AfterTaskUpdate, BeforeTaskCreate
    from clawteam.store.file import FileTaskStore

    def _on_task_update(event: AfterTaskUpdate) -> None:
        if event.team_name != team_name:
            return
        try:
            store = FileTaskStore(event.team_name)
            task = store.get(event.task_id)
            if task:
                engine.push_task(event.team_name, task)
        except Exception as exc:
            log.warning("Plane sync failed for task %s: %s", event.task_id, exc)

    bus.subscribe(AfterTaskUpdate, _on_task_update)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_sync.py -v`
Expected: All 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add clawteam/plane/__init__.py tests/test_plane_sync.py
git commit -m "feat(plane): auto-sync task changes via event bus hooks"
```

---

### Task 9: CLI Commands (`clawteam plane ...`)

**Files:**
- Modify: `clawteam/cli/commands.py` (add `plane` subcommand group)

This adds CLI commands for: `plane setup`, `plane sync`, `plane webhook`, `plane status`.

- [ ] **Step 1: Write the CLI commands**

Add to `clawteam/cli/commands.py` — find where other subcommand groups are defined and add a new `plane_app` group. Below is the code to add:

```python
# ── Plane integration ────────────────────────────────────────────────

plane_app = typer.Typer(help="Plane integration for HITL board.")
app.add_typer(plane_app, name="plane")


@plane_app.command("setup")
def plane_setup(
    url: str = typer.Option("", help="Plane instance URL (e.g. http://localhost:8082)"),
    api_key: str = typer.Option("", help="Plane API key"),
    workspace: str = typer.Option("", help="Plane workspace slug"),
    project: str = typer.Option("", help="Plane project ID"),
):
    """Configure Plane integration settings."""
    from clawteam.plane.config import load_plane_config, save_plane_config

    cfg = load_plane_config()
    if url:
        cfg.url = url
    if api_key:
        cfg.api_key = api_key
    if workspace:
        cfg.workspace_slug = workspace
    if project:
        cfg.project_id = project

    # If all required fields present, enable sync
    if cfg.url and cfg.api_key and cfg.workspace_slug and cfg.project_id:
        cfg.sync_enabled = True

    save_plane_config(cfg)

    from rich.console import Console
    console = Console()
    console.print(f"[green]Plane config saved.[/green]")
    console.print(f"  URL:       {cfg.url}")
    console.print(f"  Workspace: {cfg.workspace_slug}")
    console.print(f"  Project:   {cfg.project_id}")
    console.print(f"  Sync:      {'enabled' if cfg.sync_enabled else 'disabled'}")


@plane_app.command("status")
def plane_status():
    """Show Plane integration status and test connectivity."""
    from clawteam.plane.config import load_plane_config
    from rich.console import Console

    console = Console()
    cfg = load_plane_config()

    if not cfg.url:
        console.print("[yellow]Plane not configured. Run: clawteam plane setup[/yellow]")
        return

    console.print(f"URL:       {cfg.url}")
    console.print(f"Workspace: {cfg.workspace_slug}")
    console.print(f"Project:   {cfg.project_id}")
    console.print(f"Sync:      {'enabled' if cfg.sync_enabled else 'disabled'}")

    if cfg.sync_enabled:
        try:
            from clawteam.plane.client import PlaneClient

            client = PlaneClient(cfg.url, cfg.api_key, cfg.workspace_slug)
            projects = client.list_projects()
            console.print(f"[green]Connected! {len(projects)} project(s) found.[/green]")
            client.close()
        except Exception as exc:
            console.print(f"[red]Connection failed: {exc}[/red]")


@plane_app.command("sync")
def plane_sync(
    team: str = typer.Argument(..., help="Team name to sync"),
    direction: str = typer.Option("both", help="Sync direction: push, pull, or both"),
):
    """Run bidirectional sync between ClawTeam and Plane."""
    from clawteam.plane.config import load_plane_config
    from clawteam.plane.sync import PlaneSyncEngine
    from rich.console import Console

    console = Console()
    cfg = load_plane_config()

    if not cfg.sync_enabled:
        console.print("[yellow]Plane sync not enabled. Run: clawteam plane setup[/yellow]")
        return

    engine = PlaneSyncEngine(cfg)

    if direction in ("push", "both"):
        pushed = engine.push_all(team)
        console.print(f"[green]Pushed {pushed} task(s) to Plane.[/green]")

    if direction in ("pull", "both"):
        pulled = engine.pull_all(team)
        console.print(f"[green]Pulled {pulled} item(s) from Plane.[/green]")


@plane_app.command("webhook")
def plane_webhook(
    team: str = typer.Argument(..., help="Team name to receive webhooks for"),
    port: int = typer.Option(9091, help="Port for webhook receiver"),
):
    """Start Plane webhook receiver for real-time HITL sync."""
    from clawteam.plane.config import load_plane_config
    from clawteam.plane.webhook import serve_webhook
    from rich.console import Console

    console = Console()
    cfg = load_plane_config()
    cfg.webhook_port = port

    if not cfg.sync_enabled:
        console.print("[yellow]Plane sync not enabled. Run: clawteam plane setup[/yellow]")
        return

    # Pre-fetch states for the webhook handler
    state_lookup = {}
    try:
        from clawteam.plane.client import PlaneClient

        client = PlaneClient(cfg.url, cfg.api_key, cfg.workspace_slug)
        states = client.list_states(cfg.project_id)
        state_lookup = {s.id: s for s in states}
        client.close()
    except Exception as exc:
        console.print(f"[yellow]Warning: could not fetch states: {exc}[/yellow]")

    console.print(f"[green]Starting Plane webhook receiver on port {port}...[/green]")
    console.print(f"Configure Plane webhook URL: http://<your-ip>:{port}/")
    serve_webhook(cfg, team, state_lookup)
```

- [ ] **Step 2: Verify the CLI loads without errors**

Run: `cd /home/jac/repos/ClawTeam && python -m clawteam.cli.commands plane --help`
Expected: Shows `plane` subcommand help with setup/status/sync/webhook commands

- [ ] **Step 3: Commit**

```bash
git add clawteam/cli/commands.py
git commit -m "feat(plane): add CLI commands (setup, status, sync, webhook)"
```

---

### Task 10: Update Package Init and Exports

**Files:**
- Modify: `clawteam/plane/__init__.py` (add public exports)
- Modify: `clawteam/store/__init__.py` (document plane metadata convention)

- [ ] **Step 1: Update plane package init with exports**

```python
# clawteam/plane/__init__.py
"""Plane integration for ClawTeam — bidirectional sync with Plane project management."""

from __future__ import annotations

import logging
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from clawteam.events.bus import EventBus
    from clawteam.plane.sync import PlaneSyncEngine

log = logging.getLogger(__name__)


def register_sync_hooks(bus: EventBus, engine: PlaneSyncEngine, team_name: str) -> None:
    """Subscribe to task events and auto-push changes to Plane."""
    from clawteam.events.types import AfterTaskUpdate
    from clawteam.store.file import FileTaskStore

    def _on_task_update(event: AfterTaskUpdate) -> None:
        if event.team_name != team_name:
            return
        try:
            store = FileTaskStore(event.team_name)
            task = store.get(event.task_id)
            if task:
                engine.push_task(event.team_name, task)
        except Exception as exc:
            log.warning("Plane sync failed for task %s: %s", event.task_id, exc)

    bus.subscribe(AfterTaskUpdate, _on_task_update)


__all__ = ["register_sync_hooks"]
```

- [ ] **Step 2: Run full test suite**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/ -v --tb=short`
Expected: All tests PASS (existing + new plane tests)

- [ ] **Step 3: Commit**

```bash
git add clawteam/plane/__init__.py
git commit -m "feat(plane): finalize package exports and public API"
```

---

### Task 11: Integration Test — Full Round-Trip

**Files:**
- Create: `tests/test_plane_integration.py`

This test validates the full cycle: create task in file store → push to Plane (mocked) → simulate Plane webhook → verify file store updated.

- [ ] **Step 1: Write the integration test**

```python
# tests/test_plane_integration.py
"""End-to-end integration test for the full Plane sync + HITL cycle."""

from __future__ import annotations

from pathlib import Path
from unittest.mock import MagicMock

import pytest

from clawteam.plane.config import PlaneConfig
from clawteam.plane.models import PlaneState, PlaneWorkItem
from clawteam.plane.sync import PlaneSyncEngine
from clawteam.plane.webhook import _handle_work_item_event, _handle_comment_event
from clawteam.store.file import FileTaskStore
from clawteam.team.manager import TeamManager
from clawteam.team.models import TaskStatus


@pytest.fixture
def env(monkeypatch, tmp_path: Path):
    monkeypatch.setenv("CLAWTEAM_DATA_DIR", str(tmp_path))
    TeamManager.create_team(
        name="integration", leader_name="leader", leader_id="leader001",
    )
    TeamManager.add_member("integration", "worker1", "worker001")
    return "integration"


@pytest.fixture
def config():
    return PlaneConfig(
        url="http://localhost:8082",
        api_key="test-key",
        workspace_slug="test-ws",
        project_id="proj-1",
        sync_enabled=True,
    )


@pytest.fixture
def states():
    return [
        PlaneState(id="s-pending", name="Pending", group="unstarted"),
        PlaneState(id="s-progress", name="In Progress", group="started"),
        PlaneState(id="s-done", name="Done", group="completed"),
        PlaneState(id="s-blocked", name="Blocked", group="backlog"),
        PlaneState(id="s-approval", name="Awaiting Approval", group="unstarted"),
    ]


def test_full_round_trip(env, config, states):
    """
    1. Agent creates task in file store
    2. Push to Plane (mocked)
    3. Human changes state in Plane → webhook fires
    4. File store updates
    """
    team = env
    store = FileTaskStore(team)

    # Step 1: Agent creates task
    task = store.create(subject="Implement login", owner="worker1")
    assert task.status == TaskStatus.pending

    # Step 2: Push to Plane
    mock_client = MagicMock()
    mock_client.list_states.return_value = states
    mock_client.create_work_item.return_value = PlaneWorkItem(
        id="plane-rt-1", name="Implement login", state="s-pending",
    )
    engine = PlaneSyncEngine(config, client=mock_client)
    engine.push_task(team, task)

    updated_task = store.get(task.id)
    assert updated_task.metadata["plane_issue_id"] == "plane-rt-1"

    # Step 3: Human moves to "In Progress" in Plane → webhook
    state_lookup = {s.id: s for s in states}
    result = _handle_work_item_event(
        {
            "event": "issue",
            "action": "updated",
            "data": {
                "id": "plane-rt-1",
                "name": "Implement login",
                "state": "s-progress",
                "priority": "medium",
            },
        },
        config, team, state_lookup,
    )
    assert result["action"] == "updated"

    # Step 4: Verify file store updated
    final = store.get(task.id)
    assert final.status == TaskStatus.in_progress


def test_hitl_approval_via_comment(env, config, states):
    """
    1. Task exists with plane_issue_id
    2. Human comments "APPROVED" on Plane issue
    3. Webhook triggers → task moves to in_progress + HITL message sent
    """
    team = env
    store = FileTaskStore(team)
    task = store.create(
        subject="Deploy to staging",
        owner="worker1",
        metadata={"plane_issue_id": "plane-hitl-1"},
    )

    result = _handle_comment_event(
        {
            "event": "issue_comment",
            "action": "created",
            "data": {
                "comment_html": "<p>APPROVED - looks good!</p>",
                "issue": "plane-hitl-1",
                "actor_detail": {"display_name": "Alice"},
                "created_at": "2026-04-09T12:00:00Z",
            },
        },
        config, team,
    )

    assert result["action"] == "approved"
    updated = store.get(task.id)
    assert updated.status == TaskStatus.in_progress


def test_human_creates_task_in_plane_syncs_to_file(env, config, states):
    """Human creates a task directly in Plane → webhook → appears in file store."""
    team = env
    state_lookup = {s.id: s for s in states}

    result = _handle_work_item_event(
        {
            "event": "issue",
            "action": "created",
            "data": {
                "id": "plane-human-1",
                "name": "Review architecture doc",
                "description_html": "<p>Please review</p>",
                "state": "s-pending",
                "priority": "high",
            },
        },
        config, team, state_lookup,
    )

    assert result["action"] == "created"
    store = FileTaskStore(team)
    tasks = store.list_tasks()
    human_task = next(t for t in tasks if t.metadata.get("plane_issue_id") == "plane-human-1")
    assert human_task.subject == "Review architecture doc"
    assert human_task.priority.value == "high"
```

- [ ] **Step 2: Run the integration test**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/test_plane_integration.py -v`
Expected: All 3 tests PASS

- [ ] **Step 3: Run the full test suite to verify no regressions**

Run: `cd /home/jac/repos/ClawTeam && python -m pytest tests/ -v --tb=short`
Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git add tests/test_plane_integration.py
git commit -m "test(plane): add full round-trip integration tests for sync + HITL"
```

---

## Summary

| Task | Component | Description |
|------|-----------|-------------|
| 1 | Setup | Docker deployment script for self-hosted Plane |
| 2 | Config | `PlaneConfig` model with persistence |
| 3 | Models | Pydantic models for Plane API objects |
| 4 | Client | `PlaneClient` REST API wrapper (httpx) |
| 5 | Mapping | ClawTeam ↔ Plane status/state/priority mapping |
| 6 | Sync | `PlaneSyncEngine` — bidirectional file↔Plane sync |
| 7 | Webhook | Plane webhook receiver with HITL approve/reject |
| 8 | Events | Event bus hooks for auto-sync on task changes |
| 9 | CLI | `clawteam plane setup/status/sync/webhook` commands |
| 10 | Package | Exports and public API |
| 11 | Tests | Full round-trip integration tests |

**Data flow:**
```
Agent creates task → FileTaskStore → EventBus(AfterTaskUpdate)
                                          ↓
                                    PlaneSyncEngine.push_task()
                                          ↓
                                    Plane REST API (create/update work item)
                                          ↓
Human acts in Plane UI (approve/reject/create)
                                          ↓
                                    Plane Webhook POST
                                          ↓
                                    PlaneWebhookHandler
                                          ↓
                                    FileTaskStore.update() / .create()
                                          ↓
                                    MailboxManager.send() (HITL message to agent)
```
