# Weekend 1 — Collection System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to
> implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `python -m scorecard.cli collect --all` runs end to end, writes one
snapshot JSON per company per ISO week, and runs green on a schedule in GitHub
Actions with ≥80% coverage on ATS, GitHub and pricing.

**Architecture:** Six independent collectors, each subclassing a `Collector` ABC
that owns retry, rate limiting, robots.txt and the error envelope. A collector
raising an exception is caught by the base class and recorded as
`status: "error"` in the snapshot — it never aborts the company, and a failing
company never aborts the run. Snapshots are raw values only; every derived
quantity is computed later by `ledger/`, which does not exist yet.

**Tech Stack:** Python 3.11+, `uv`, `ruff` (line length 88), `httpx`,
`selectolax`, `pydantic` v2, `typer`, `tenacity`, `pyyaml`, `pytest`.

**Scope:** Weekend 1 of `SPEC.md` plus the enterprise-readiness docs regex from
the design spec §9. EDGAR, LinkedIn/Apify and `customers.py` are Week 2 and are
NOT in this plan. `ledger/` is a separate plan.

**Reference:** `docs/superpowers/specs/2026-08-28-prediction-ledger-design.md`

**One disclosed divergence from `SPEC.md`:** the CLI is
`python -m scorecard.cli collect --all`, not `python -m scorecard.collect --all`.
`scorecard/collect.py` is the orchestration module and cannot also be the typer
entrypoint without an awkward split. All `SPEC.md` subcommands exist with the
same names and flags.

---

## File Structure

| File | Responsibility |
|---|---|
| `pyproject.toml` | deps, ruff config, pytest config |
| `scorecard/models.py` | every pydantic schema. Schemas live here and nowhere else. |
| `scorecard/storage.py` | ISO week, snapshot read/write, hash-gated raw archiving |
| `scorecard/universe.py` | load + validate `companies/universe.yaml` |
| `scorecard/http.py` | shared client: UA, per-domain rate limit, robots.txt |
| `scorecard/collectors/base.py` | `Collector` ABC, retry, error envelope |
| `scorecard/collectors/taxonomy.py` | job title → department + seniority |
| `scorecard/collectors/ats.py` | Greenhouse / Lever / Ashby |
| `scorecard/collectors/github.py` | repos, commits, releases |
| `scorecard/collectors/pricing.py` | hash, tiers, prices |
| `scorecard/collectors/changelog.py` | entries, docs sitemap, enterprise flags |
| `scorecard/validate.py` | coverage report |
| `scorecard/cli.py` | typer entrypoint |
| `.github/workflows/collect.yml` | Monday 09:00 UTC |

**Dependency direction:** `collectors/*` → `http`, `models`. `cli` → everything.
Nothing imports `cli`. No collector imports another collector.

---

## Task 1: Project scaffold

**Files:**
- Create: `pyproject.toml`, `.env.example`, `scorecard/__init__.py`,
  `scorecard/collectors/__init__.py`, `tests/__init__.py`

- [ ] **Step 1: Write `pyproject.toml`**

```toml
[project]
name = "momentum-scorecard"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "selectolax>=0.3.21",
    "pydantic>=2.7",
    "typer>=0.12",
    "pyyaml>=6.0",
    "tenacity>=8.3",
]

[dependency-groups]
dev = ["pytest>=8.2", "ruff>=0.5", "respx>=0.21"]

[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]

[tool.pytest.ini_options]
testpaths = ["tests"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["scorecard"]
```

- [ ] **Step 2: Create package directories**

```bash
mkdir -p scorecard/collectors tests/fixtures companies data
touch scorecard/__init__.py scorecard/collectors/__init__.py tests/__init__.py
```

- [ ] **Step 3: Write `.env.example`**

```bash
cat > .env.example <<'EOF'
# Required. Sent in User-Agent on every request. SEC EDGAR mandates this.
SCRAPER_CONTACT_EMAIL=blacksha@usc.edu
# Optional but strongly recommended: raises GitHub limit to 5000 req/hr.
GITHUB_TOKEN=
EOF
cp .env.example .env
```

- [ ] **Step 4: Install and verify**

Run: `uv sync && uv run python -c "import httpx, selectolax, pydantic; print('ok')"`
Expected: `ok`

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml .env.example scorecard tests uv.lock
git commit -m "chore: project scaffold"
```

---

## Task 2: Schemas

**Files:**
- Create: `scorecard/models.py`
- Test: `tests/test_models.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_models.py
import pytest
from pydantic import ValidationError

from scorecard.models import CollectorResult, Snapshot, Status


def test_ok_result_carries_data_and_no_error():
    r = CollectorResult(status=Status.OK, data={"open_roles": 3})
    assert r.status == "ok"
    assert r.error is None
    assert r.data == {"open_roles": 3}


def test_error_result_requires_a_message():
    with pytest.raises(ValidationError):
        CollectorResult(status=Status.ERROR, data=None)


def test_ok_result_rejects_null_data():
    with pytest.raises(ValidationError):
        CollectorResult(status=Status.OK, data=None)


def test_snapshot_always_contains_every_collector_key():
    s = Snapshot(slug="modal-labs", iso_week="2026-W36")
    assert set(s.collectors) == {
        "github", "ats", "pricing", "changelog", "edgar", "linkedin"
    }
    assert s.collectors["edgar"].status == "skipped"


def test_iso_week_format_is_enforced():
    with pytest.raises(ValidationError):
        Snapshot(slug="x", iso_week="2026-36")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_models.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.models'`

- [ ] **Step 3: Write `scorecard/models.py`**

```python
"""Pydantic schemas. Every schema in the project lives in this file."""

from __future__ import annotations

import re
from datetime import datetime, timezone
from enum import Enum
from typing import Any

from pydantic import BaseModel, Field, field_validator, model_validator

SCHEMA_VERSION = 1
ISO_WEEK_RE = re.compile(r"^\d{4}-W\d{2}$")

COLLECTOR_NAMES = ("github", "ats", "pricing", "changelog", "edgar", "linkedin")


class Status(str, Enum):
    OK = "ok"
    ERROR = "error"
    SKIPPED = "skipped"
    PARTIAL = "partial"


class CollectorResult(BaseModel):
    """The envelope every collector writes, success or failure.

    A collector key is never omitted from a snapshot. Distinguishing "we did
    not collect this" from "this does not exist" matters at scoring time.
    """

    status: Status
    error: str | None = None
    data: dict[str, Any] | None = None

    @model_validator(mode="after")
    def check_envelope(self) -> CollectorResult:
        if self.status in (Status.ERROR, Status.SKIPPED) and not self.error:
            raise ValueError(f"status={self.status.value} requires an error message")
        if self.status in (Status.OK, Status.PARTIAL) and self.data is None:
            raise ValueError(f"status={self.status.value} requires data")
        return self


def _default_collectors() -> dict[str, CollectorResult]:
    return {
        name: CollectorResult(status=Status.SKIPPED, error="not collected")
        for name in COLLECTOR_NAMES
    }


class Snapshot(BaseModel):
    """One company, one ISO week. Raw values only — never derived ones."""

    slug: str
    iso_week: str
    collected_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    schema_version: int = SCHEMA_VERSION
    collectors: dict[str, CollectorResult] = Field(
        default_factory=_default_collectors
    )

    @field_validator("iso_week")
    @classmethod
    def check_iso_week(cls, v: str) -> str:
        if not ISO_WEEK_RE.match(v):
            raise ValueError(f"iso_week must look like 2026-W36, got {v!r}")
        return v


class AtsConfig(BaseModel):
    provider: str | None = None
    token: str | None = None


class Sources(BaseModel):
    github_org: str | None = None
    ats: AtsConfig = Field(default_factory=AtsConfig)
    careers_url: str | None = None
    pricing_url: str | None = None
    changelog_url: str | None = None
    docs_url: str | None = None
    linkedin_url: str | None = None
    cik: str | None = None


class Company(BaseModel):
    slug: str
    name: str
    domain: str
    sector: str
    stage: str
    business_model: str = "b2b"
    added: str
    cohort: str
    sources: Sources = Field(default_factory=Sources)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_models.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/models.py tests/test_models.py
git commit -m "feat: snapshot and collector schemas"
```

---

## Task 3: Storage

**Files:**
- Create: `scorecard/storage.py`
- Test: `tests/test_storage.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_storage.py
from datetime import date

from scorecard.models import CollectorResult, Snapshot, Status
from scorecard.storage import (
    archive_raw_if_changed,
    current_iso_week,
    load_snapshot,
    snapshot_exists,
    write_snapshot,
)


def test_current_iso_week_formats_with_zero_padding():
    assert current_iso_week(date(2026, 1, 5)) == "2026-W02"
    assert current_iso_week(date(2026, 9, 7)) == "2026-W37"


def test_write_then_load_roundtrips(tmp_path):
    snap = Snapshot(slug="modal-labs", iso_week="2026-W36")
    snap.collectors["ats"] = CollectorResult(
        status=Status.OK, data={"open_roles": 23}
    )
    write_snapshot(snap, root=tmp_path)

    loaded = load_snapshot("modal-labs", "2026-W36", root=tmp_path)
    assert loaded.collectors["ats"].data["open_roles"] == 23


def test_rerunning_a_week_overwrites_never_duplicates(tmp_path):
    write_snapshot(Snapshot(slug="a", iso_week="2026-W36"), root=tmp_path)
    write_snapshot(Snapshot(slug="a", iso_week="2026-W36"), root=tmp_path)
    files = list((tmp_path / "snapshots" / "2026-W36").glob("*.json"))
    assert len(files) == 1


def test_writing_this_week_does_not_touch_prior_weeks(tmp_path):
    write_snapshot(Snapshot(slug="a", iso_week="2026-W35"), root=tmp_path)
    write_snapshot(Snapshot(slug="a", iso_week="2026-W36"), root=tmp_path)
    assert snapshot_exists("a", "2026-W35", root=tmp_path)
    assert snapshot_exists("a", "2026-W36", root=tmp_path)


def test_raw_archived_on_first_write(tmp_path):
    path = archive_raw_if_changed(
        "modal-labs", "pricing", "2026-W36", "<html>hi</html>",
        previous_hash=None, root=tmp_path,
    )
    assert path is not None
    assert path.endswith("2026-W36.html.gz")


def test_raw_not_rearchived_when_hash_unchanged(tmp_path):
    from scorecard.storage import content_hash

    html = "<html>hi</html>"
    same = content_hash(html)
    path = archive_raw_if_changed(
        "modal-labs", "pricing", "2026-W36", html,
        previous_hash=same, root=tmp_path,
    )
    assert path is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_storage.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.storage'`

- [ ] **Step 3: Write `scorecard/storage.py`**

```python
"""Reading and writing the datastore. The git repo is the database."""

from __future__ import annotations

import gzip
import hashlib
import json
from datetime import date
from pathlib import Path

from scorecard.models import Snapshot

DATA_ROOT = Path("data")


def current_iso_week(today: date | None = None) -> str:
    """Return the ISO week as e.g. 2026-W36."""
    d = today or date.today()
    iso_year, iso_week, _ = d.isocalendar()
    return f"{iso_year}-W{iso_week:02d}"


def content_hash(text: str) -> str:
    """Hash normalised text. Prefixed so the algorithm is visible in the data."""
    normalised = " ".join(text.split())
    digest = hashlib.sha256(normalised.encode("utf-8")).hexdigest()
    return f"sha256:{digest}"


def _snapshot_path(slug: str, iso_week: str, root: Path) -> Path:
    return root / "snapshots" / iso_week / f"{slug}.json"


def snapshot_exists(slug: str, iso_week: str, root: Path = DATA_ROOT) -> bool:
    return _snapshot_path(slug, iso_week, root).exists()


def write_snapshot(snapshot: Snapshot, root: Path = DATA_ROOT) -> Path:
    """Write one snapshot, overwriting this week's file if it already exists.

    Only ever touches the file for `snapshot.iso_week`. Prior weeks are
    immutable by construction.
    """
    path = _snapshot_path(snapshot.slug, snapshot.iso_week, root)
    path.parent.mkdir(parents=True, exist_ok=True)
    payload = snapshot.model_dump(mode="json")
    path.write_text(json.dumps(payload, indent=2, sort_keys=True) + "\n")
    return path


def load_snapshot(
    slug: str, iso_week: str, root: Path = DATA_ROOT
) -> Snapshot | None:
    path = _snapshot_path(slug, iso_week, root)
    if not path.exists():
        return None
    return Snapshot.model_validate_json(path.read_text())


def archive_raw_if_changed(
    slug: str,
    kind: str,
    iso_week: str,
    html: str,
    previous_hash: str | None,
    root: Path = DATA_ROOT,
) -> str | None:
    """Gzip and store raw HTML only when its content hash changed.

    Most weeks nothing changes. Skipping unchanged pages keeps the repo small
    while preserving the ability to re-parse history with a better parser.
    Returns the archived path, or None when nothing was written.
    """
    if previous_hash is not None and content_hash(html) == previous_hash:
        return None
    path = root / "raw" / slug / kind / f"{iso_week}.html.gz"
    path.parent.mkdir(parents=True, exist_ok=True)
    with gzip.open(path, "wt", encoding="utf-8") as fh:
        fh.write(html)
    return str(path)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_storage.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/storage.py tests/test_storage.py
git commit -m "feat: snapshot storage with hash-gated raw archiving"
```

---

## Task 4: Universe loading

**Files:**
- Create: `scorecard/universe.py`, `companies/universe.yaml`
- Test: `tests/test_universe.py`

- [ ] **Step 1: Write `companies/universe.yaml` with 10 hand-entered companies**

These 10 are the Weekend 1 test set. All are systematic-cohort AI companies with
public ATS boards. Verify each `token` by opening the board URL in a browser
before committing — a wrong token silently yields zero roles.

```yaml
- slug: modal-labs
  name: Modal Labs
  domain: modal.com
  sector: ai-infra
  stage: series-a
  business_model: b2b
  added: 2026-08-29
  cohort: manual-seed
  sources:
    github_org: modal-labs
    ats: {provider: ashby, token: modal}
    careers_url: https://modal.com/careers
    pricing_url: https://modal.com/pricing
    changelog_url: https://modal.com/docs/reference/changelog
    docs_url: https://modal.com/docs
    linkedin_url: https://linkedin.com/company/modal-labs
    cik: null
```

Repeat that block for nine more companies. Choose companies that have a public
ATS board AND a public pricing page, since those are the two collectors with an
80% coverage bar. Leave `github_org: null` where there is no public org — that
is a legitimate `skipped`, not a bug.

- [ ] **Step 2: Write the failing test**

```python
# tests/test_universe.py
import pytest

from scorecard.universe import load_universe


def test_loads_all_companies(tmp_path):
    (tmp_path / "universe.yaml").write_text(
        """
- slug: modal-labs
  name: Modal Labs
  domain: modal.com
  sector: ai-infra
  stage: series-a
  added: 2026-08-29
  cohort: manual-seed
  sources:
    github_org: modal-labs
    ats: {provider: ashby, token: modal}
"""
    )
    companies = load_universe(tmp_path / "universe.yaml")
    assert len(companies) == 1
    assert companies[0].sources.ats.provider == "ashby"


def test_duplicate_slugs_are_rejected(tmp_path):
    (tmp_path / "u.yaml").write_text(
        """
- {slug: a, name: A, domain: a.com, sector: ai-infra, stage: seed,
   added: 2026-08-29, cohort: manual-seed}
- {slug: a, name: B, domain: b.com, sector: ai-infra, stage: seed,
   added: 2026-08-29, cohort: manual-seed}
"""
    )
    with pytest.raises(ValueError, match="duplicate slug"):
        load_universe(tmp_path / "u.yaml")


def test_real_universe_file_is_valid():
    companies = load_universe()
    assert len(companies) >= 10
    assert len({c.slug for c in companies}) == len(companies)
```

- [ ] **Step 3: Run test to verify it fails**

Run: `uv run pytest tests/test_universe.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.universe'`

- [ ] **Step 4: Write `scorecard/universe.py`**

```python
"""Load and validate the hand-curated company universe."""

from __future__ import annotations

from pathlib import Path

import yaml

from scorecard.models import Company

UNIVERSE_PATH = Path("companies/universe.yaml")


def load_universe(path: Path | None = None) -> list[Company]:
    """Parse universe.yaml into validated Company models.

    Raises on duplicate slugs. A slug is a permanent identifier; two entries
    sharing one would silently overwrite each other's snapshots.
    """
    p = path or UNIVERSE_PATH
    raw = yaml.safe_load(p.read_text()) or []
    companies = [Company.model_validate(entry) for entry in raw]

    seen: set[str] = set()
    for c in companies:
        if c.slug in seen:
            raise ValueError(f"duplicate slug in universe: {c.slug}")
        seen.add(c.slug)
    return companies
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_universe.py -v`
Expected: 3 passed

- [ ] **Step 6: Commit**

```bash
git add scorecard/universe.py companies/universe.yaml tests/test_universe.py
git commit -m "feat: universe loading with 10 seed companies"
```

---

## Task 5: Polite HTTP client

**Files:**
- Create: `scorecard/http.py`
- Test: `tests/test_http.py`

Politeness is non-negotiable per `CLAUDE.md`. Centralising it here means no
collector can accidentally skip it.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_http.py
import httpx
import pytest
import respx

from scorecard.http import PoliteClient, user_agent


def test_user_agent_reads_email_from_env(monkeypatch):
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    assert "blacksha@usc.edu" in user_agent()


def test_missing_email_raises_rather_than_going_anonymous(monkeypatch):
    monkeypatch.delenv("SCRAPER_CONTACT_EMAIL", raising=False)
    with pytest.raises(RuntimeError, match="SCRAPER_CONTACT_EMAIL"):
        user_agent()


@respx.mock
def test_get_sends_the_user_agent(monkeypatch):
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    route = respx.get("https://example.com/x").mock(
        return_value=httpx.Response(200, text="hi")
    )
    client = PoliteClient(min_interval=0.0, check_robots=False)
    client.get("https://example.com/x")
    assert "blacksha@usc.edu" in route.calls[0].request.headers["user-agent"]


@respx.mock
def test_retries_then_succeeds(monkeypatch):
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    respx.get("https://example.com/y").mock(
        side_effect=[
            httpx.Response(500),
            httpx.Response(500),
            httpx.Response(200, text="ok"),
        ]
    )
    client = PoliteClient(min_interval=0.0, check_robots=False)
    assert client.get("https://example.com/y").text == "ok"


@respx.mock
def test_gives_up_after_three_attempts_on_5xx(monkeypatch):
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    route = respx.get("https://example.com/z").mock(
        return_value=httpx.Response(500)
    )
    client = PoliteClient(min_interval=0.0, check_robots=False)
    with pytest.raises(httpx.HTTPStatusError):
        client.get("https://example.com/z")
    assert route.call_count == 3


@respx.mock
def test_404_is_not_retried(monkeypatch):
    """A 404 is a definitive answer. Retrying it is slow and impolite."""
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    route = respx.get("https://example.com/gone").mock(
        return_value=httpx.Response(404)
    )
    client = PoliteClient(min_interval=0.0, check_robots=False)
    with pytest.raises(httpx.HTTPStatusError):
        client.get("https://example.com/gone")
    assert route.call_count == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_http.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.http'`

- [ ] **Step 3: Write `scorecard/http.py`**

```python
"""Shared HTTP client. Every outbound request in the project goes through here.

Politeness rules from CLAUDE.md, enforced in one place so no collector can
skip them: identifying User-Agent, robots.txt, 1 req/sec per domain.
"""

from __future__ import annotations

import os
import time
import urllib.robotparser
from urllib.parse import urlparse

import httpx
from tenacity import (
    retry,
    retry_if_exception,
    stop_after_attempt,
    wait_exponential,
)

PROJECT_NAME = "Momentum Scorecard (research project)"
MIN_INTERVAL_SECONDS = 1.0
TIMEOUT_SECONDS = 20.0


def user_agent() -> str:
    """Build the User-Agent. Raises if no contact email is configured.

    Deliberately fails loudly: a silent fallback to an anonymous UA would make
    every request impolite without anyone noticing, and SEC EDGAR requires it.
    """
    email = os.environ.get("SCRAPER_CONTACT_EMAIL", "").strip()
    if not email:
        raise RuntimeError(
            "SCRAPER_CONTACT_EMAIL is not set. Every request must identify "
            "the project with a real email address. See .env.example."
        )
    return f"{PROJECT_NAME} <{email}>"


class RobotsDisallowed(Exception):
    """Raised when robots.txt forbids the path. Not a failure — a skip."""


def _is_retryable(exc: BaseException) -> bool:
    """Retry transport errors and 5xx only.

    4xx must NOT be retried. A 404 is a definitive answer, and retrying it
    three times with backoff is both slow and impolite — github.py requests
    releases/latest for ten repos per org and most legitimately 404.
    """
    if isinstance(exc, httpx.TransportError):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code >= 500
    return False


class PoliteClient:
    """httpx wrapper adding identity, rate limiting, robots and retries."""

    def __init__(
        self,
        min_interval: float = MIN_INTERVAL_SECONDS,
        check_robots: bool = True,
    ) -> None:
        self._min_interval = min_interval
        self._check_robots = check_robots
        self._last_request_at: dict[str, float] = {}
        self._robots: dict[str, urllib.robotparser.RobotFileParser | None] = {}
        self._client = httpx.Client(
            timeout=TIMEOUT_SECONDS, follow_redirects=True
        )

    def _throttle(self, domain: str) -> None:
        last = self._last_request_at.get(domain)
        if last is not None:
            elapsed = time.monotonic() - last
            if elapsed < self._min_interval:
                time.sleep(self._min_interval - elapsed)
        self._last_request_at[domain] = time.monotonic()

    def _robots_allows(self, url: str) -> bool:
        parsed = urlparse(url)
        domain = parsed.netloc
        if domain not in self._robots:
            parser = urllib.robotparser.RobotFileParser()
            parser.set_url(f"{parsed.scheme}://{domain}/robots.txt")
            try:
                parser.read()
            except Exception:
                # An unreachable robots.txt is not permission to ignore it,
                # but it is also not a reason to stop. Treat as allowed and
                # rely on rate limiting.
                self._robots[domain] = None
            else:
                self._robots[domain] = parser
        parser = self._robots[domain]
        if parser is None:
            return True
        return parser.can_fetch(user_agent(), url)

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception(_is_retryable),
        reraise=True,
    )
    def get(self, url: str, **kwargs: object) -> httpx.Response:
        if self._check_robots and not self._robots_allows(url):
            raise RobotsDisallowed(f"robots.txt disallows {url}")
        self._throttle(urlparse(url).netloc)
        headers = {"User-Agent": user_agent()}
        headers.update(kwargs.pop("headers", {}) or {})
        response = self._client.get(url, headers=headers, **kwargs)
        response.raise_for_status()
        return response

    def close(self) -> None:
        self._client.close()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_http.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/http.py tests/test_http.py
git commit -m "feat: polite http client with robots, rate limit, retries"
```

---

## Task 6: Collector base class

**Files:**
- Create: `scorecard/collectors/base.py`
- Test: `tests/test_base.py`

This is the class that makes the system degrade rather than crash. Everything it
does is in service of one rule: a collector raising must never abort anything.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_base.py
from scorecard.collectors.base import Collector, Skip
from scorecard.models import Company, Status


def _company(**kw) -> Company:
    base = dict(
        slug="a", name="A", domain="a.com", sector="ai-infra",
        stage="seed", added="2026-08-29", cohort="manual-seed",
    )
    base.update(kw)
    return Company.model_validate(base)


class Boom(Collector):
    name = "ats"

    def collect(self, company, client):
        raise ValueError("upstream exploded")


class Fine(Collector):
    name = "ats"

    def collect(self, company, client):
        return {"open_roles": 4}


class Unconfigured(Collector):
    name = "ats"

    def collect(self, company, client):
        raise Skip("no ats token")


def test_exception_becomes_an_error_result_not_a_crash():
    result = Boom().run(_company(), client=None)
    assert result.status == Status.ERROR
    assert "upstream exploded" in result.error
    assert result.data is None


def test_success_becomes_an_ok_result():
    result = Fine().run(_company(), client=None)
    assert result.status == Status.OK
    assert result.data == {"open_roles": 4}


def test_skip_is_distinct_from_error():
    result = Unconfigured().run(_company(), client=None)
    assert result.status == Status.SKIPPED
    assert result.error == "no ats token"


def test_error_message_is_truncated_not_unbounded():
    class Huge(Collector):
        name = "ats"

        def collect(self, company, client):
            raise ValueError("x" * 5000)

    result = Huge().run(_company(), client=None)
    assert len(result.error) <= 500
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_base.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.base'`

- [ ] **Step 3: Write `scorecard/collectors/base.py`**

```python
"""The Collector ABC. Owns the error envelope so collectors can be naive."""

from __future__ import annotations

import logging
from abc import ABC, abstractmethod
from typing import Any

from scorecard.http import PoliteClient, RobotsDisallowed
from scorecard.models import CollectorResult, Company, Status

logger = logging.getLogger(__name__)

MAX_ERROR_LENGTH = 500


class Skip(Exception):
    """Raised when a collector is not configured for this company.

    Distinct from an error: "this company has no GitHub org" is information,
    not a failure, and the two must stay distinguishable at scoring time.
    """


class Partial(Exception):
    """Raised when a collector got something, but not everything.

    Carries the partial data so it is still written to the snapshot.
    """

    def __init__(self, message: str, data: dict[str, Any]) -> None:
        super().__init__(message)
        self.data = data


class Collector(ABC):
    """Base class for all collectors.

    Subclasses implement `collect` and may raise freely. `run` converts any
    outcome into a CollectorResult, so one collector failing never aborts the
    company and one company failing never aborts the run.
    """

    name: str

    @abstractmethod
    def collect(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        """Fetch and parse. Return raw values only — never derived ones."""

    def run(
        self, company: Company, client: PoliteClient | None
    ) -> CollectorResult:
        try:
            data = self.collect(company, client)
        except Skip as exc:
            logger.info("%s %s skipped: %s", company.slug, self.name, exc)
            return CollectorResult(status=Status.SKIPPED, error=str(exc))
        except Partial as exc:
            logger.warning("%s %s partial: %s", company.slug, self.name, exc)
            return CollectorResult(
                status=Status.PARTIAL,
                error=self._truncate(str(exc)),
                data=exc.data,
            )
        except RobotsDisallowed as exc:
            logger.info("%s %s robots: %s", company.slug, self.name, exc)
            return CollectorResult(
                status=Status.SKIPPED, error=self._truncate(str(exc))
            )
        except Exception as exc:  # noqa: BLE001 — degrade, never crash
            logger.warning(
                "%s %s error: %s: %s",
                company.slug, self.name, type(exc).__name__, exc,
            )
            message = f"{type(exc).__name__}: {exc}"
            return CollectorResult(
                status=Status.ERROR, error=self._truncate(message)
            )
        logger.info("%s %s ok", company.slug, self.name)
        return CollectorResult(status=Status.OK, data=data)

    @staticmethod
    def _truncate(message: str) -> str:
        if len(message) <= MAX_ERROR_LENGTH:
            return message
        return message[: MAX_ERROR_LENGTH - 3] + "..."
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_base.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/collectors/base.py tests/test_base.py
git commit -m "feat: Collector ABC with degrade-never-crash error envelope"
```

---

## Task 7: Title taxonomy

**Files:**
- Create: `scorecard/collectors/taxonomy.py`
- Test: `tests/test_taxonomy.py`

`CLAUDE.md` names this as one of the two parsers most needing tests. It encodes
judgment, it feeds the highest-value signal, and it will be wrong in interesting
ways. Ambiguous titles are resolved by **most specific match wins**, so ordering
inside each keyword list is load-bearing.

Two calls are deliberately contentious and are pinned by tests so that changing
them is a visible decision rather than an accident:

1. **`Solutions Engineer` → `sales`, not `engineering`.** It is a pre-sales,
   customer-facing role. Classifying it as engineering would hide exactly the
   GTM inflection the project exists to detect.
2. **`Founding Engineer` → `exec`**, per the `SPEC.md` rule that `founding`
   implies exec. This is probably wrong — a founding engineer at a seed company
   is a senior IC — and it inflates exec counts at precisely the companies of
   interest. It is implemented as specified, pinned by a test, and recorded in
   `RULES.md` as a known limitation. `raw_titles` is stored in every snapshot
   so a corrected classifier can be applied retroactively to all history.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_taxonomy.py
import pytest

from scorecard.collectors.taxonomy import classify_department, classify_seniority


@pytest.mark.parametrize(
    "title,expected",
    [
        ("Senior Backend Engineer", "engineering"),
        ("Staff Software Engineer, Infra", "engineering"),
        ("Machine Learning Engineer", "engineering"),
        ("Product Manager", "product"),
        ("Group Product Manager", "product"),
        ("Product Designer", "design"),
        ("Enterprise Account Executive, West", "sales"),
        ("Account Executive", "sales"),
        ("Sales Development Representative", "sales"),
        ("Solutions Engineer", "sales"),
        ("Sales Engineer", "sales"),
        ("Growth Marketing Manager", "marketing"),
        ("Developer Advocate", "marketing"),
        ("Customer Success Manager", "customer-success"),
        ("Support Engineer", "customer-success"),
        ("Business Operations Analyst", "operations"),
        ("Controller", "finance"),
        ("Technical Recruiter", "people"),
        ("Chief of Staff", "other"),
    ],
)
def test_department_classification(title, expected):
    assert classify_department(title) == expected


@pytest.mark.parametrize(
    "title,expected",
    [
        ("Software Engineering Intern", "junior"),
        ("Junior Analyst", "junior"),
        ("Associate Product Manager", "junior"),
        ("Backend Engineer", "mid"),
        ("Account Executive", "mid"),
        ("Senior Backend Engineer", "senior"),
        ("Staff Engineer", "senior"),
        ("Principal Engineer", "senior"),
        ("Engineering Lead", "senior"),
        ("Head of Sales", "exec"),
        ("VP of Engineering", "exec"),
        ("Director of Marketing", "exec"),
        ("Chief Technology Officer", "exec"),
    ],
)
def test_seniority_classification(title, expected):
    assert classify_seniority(title) == expected


def test_solutions_engineer_is_gtm_not_engineering():
    """Pinned: a pre-sales role must not be hidden inside the eng count."""
    assert classify_department("Solutions Engineer") == "sales"


def test_founding_engineer_is_exec_known_limitation():
    """Pinned per SPEC.md. Probably wrong; see RULES.md.

    A founding engineer is a senior IC, not an executive. This inflates exec
    counts at seed companies. raw_titles is stored so this can be corrected
    retroactively across all history.
    """
    assert classify_seniority("Founding Engineer") == "exec"


def test_exec_beats_senior_when_both_match():
    assert classify_seniority("Senior Director, Sales") == "exec"


def test_unknown_title_falls_back_not_crashes():
    assert classify_department("Zookeeper") == "other"
    assert classify_seniority("Zookeeper") == "mid"


def test_classification_is_case_insensitive():
    assert classify_department("SENIOR BACKEND ENGINEER") == "engineering"
    assert classify_seniority("senior backend engineer") == "senior"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_taxonomy.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.taxonomy'`

- [ ] **Step 3: Write `scorecard/collectors/taxonomy.py`**

```python
"""Job title -> department and seniority.

ATS providers disagree about departments, so titles are normalised here rather
than trusting provider metadata. This file encodes judgment and will be wrong
in interesting ways; `raw_titles` is stored in every snapshot so a better
classifier can be applied retroactively to all collected history.

Matching is "most specific wins": the first pattern in each ordered list that
appears in the lowercased title decides.
"""

from __future__ import annotations

DEPARTMENTS = (
    "engineering", "product", "design", "sales", "marketing",
    "customer-success", "operations", "finance", "people", "other",
)

SENIORITIES = ("junior", "mid", "senior", "exec")

# Ordered most-specific first. A title matching several rules takes the first.
_DEPARTMENT_RULES: tuple[tuple[str, str], ...] = (
    # Customer-facing technical roles are GTM, not engineering. Placed above
    # the engineering rules so "solutions engineer" is not swallowed by them.
    ("solutions engineer", "sales"),
    ("solutions architect", "sales"),
    ("sales engineer", "sales"),
    ("forward deployed", "sales"),
    # Support is customer-success even though it contains "engineer".
    ("support engineer", "customer-success"),
    ("customer success", "customer-success"),
    ("customer support", "customer-success"),
    ("technical account manager", "customer-success"),
    # Developer relations is a marketing function.
    ("developer advocate", "marketing"),
    ("developer relations", "marketing"),
    ("devrel", "marketing"),
    # People before operations: "recruiting operations" is people.
    ("recruit", "people"),
    ("talent", "people"),
    ("people ops", "people"),
    ("human resources", "people"),
    # Finance before operations for the same reason.
    ("controller", "finance"),
    ("accountant", "finance"),
    ("accounting", "finance"),
    ("finance", "finance"),
    ("fp&a", "finance"),
    # Design.
    ("designer", "design"),
    ("design", "design"),
    ("user research", "design"),
    # Product. "product marketing" must resolve to marketing, so it is listed
    # ahead of the bare "product" rule.
    ("product marketing", "marketing"),
    ("product manager", "product"),
    ("product management", "product"),
    ("product owner", "product"),
    # Sales.
    ("account executive", "sales"),
    ("sales development", "sales"),
    ("business development", "sales"),
    ("sales", "sales"),
    ("revenue", "sales"),
    # Marketing.
    ("marketing", "marketing"),
    ("growth", "marketing"),
    ("content", "marketing"),
    ("brand", "marketing"),
    # Engineering.
    ("engineer", "engineering"),
    ("engineering", "engineering"),
    ("developer", "engineering"),
    ("software", "engineering"),
    ("infrastructure", "engineering"),
    ("data scientist", "engineering"),
    ("machine learning", "engineering"),
    ("research scientist", "engineering"),
    ("security", "engineering"),
    # Operations last: it is the most generic.
    ("operations", "operations"),
    ("program manager", "operations"),
    ("project manager", "operations"),
    ("legal", "operations"),
    ("counsel", "operations"),
)

# Checked exec -> senior -> junior. Exec wins over senior so that
# "Senior Director" resolves to exec.
_EXEC_MARKERS = (
    "head of", "vp ", "vp,", "vp of", "vice president", "director",
    "chief", "cto", "ceo", "cfo", "coo", "cro", "founding", "founder",
)
_SENIOR_MARKERS = ("senior", "sr.", "sr ", "staff", "principal", "lead")
_JUNIOR_MARKERS = ("intern", "junior", "jr.", "jr ", "associate", "entry level")


def classify_department(title: str) -> str:
    """Map a raw job title to one of DEPARTMENTS. Never raises."""
    t = title.lower()
    for pattern, department in _DEPARTMENT_RULES:
        if pattern in t:
            return department
    return "other"


def classify_seniority(title: str) -> str:
    """Map a raw job title to one of SENIORITIES. Never raises.

    NOTE: "founding" maps to exec per SPEC.md. This is a known limitation —
    a founding engineer is a senior IC, not an executive. See RULES.md.
    """
    t = title.lower()
    if any(marker in t for marker in _EXEC_MARKERS):
        return "exec"
    if any(marker in t for marker in _SENIOR_MARKERS):
        return "senior"
    if any(marker in t for marker in _JUNIOR_MARKERS):
        return "junior"
    return "mid"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_taxonomy.py -v`
Expected: 40 passed

If a parametrised case fails, fix the **ordering** in `_DEPARTMENT_RULES`, not
the test. The tests encode the intended judgment; the rules implement it.

- [ ] **Step 5: Commit**

```bash
git add scorecard/collectors/taxonomy.py tests/test_taxonomy.py
git commit -m "feat: job title department and seniority taxonomy"
```

---

## Task 8: ATS collector

**Files:**
- Create: `scorecard/collectors/ats.py`
- Test: `tests/test_ats.py`
- Create: `tests/fixtures/greenhouse_jobs.json`, `tests/fixtures/lever_postings.json`,
  `tests/fixtures/ashby_board.json`

The highest-value collector. Build it before GitHub.

- [ ] **Step 1: Record the three fixtures**

Fetch each once by hand and save the response body verbatim. Tests must never
hit live endpoints.

```bash
mkdir -p tests/fixtures
curl -s "https://boards-api.greenhouse.io/v1/boards/vanta/jobs?content=true" \
  > tests/fixtures/greenhouse_jobs.json
curl -s "https://api.lever.co/v0/postings/mux?mode=json" \
  > tests/fixtures/lever_postings.json
curl -s "https://api.ashbyhq.com/posting-api/job-board/modal" \
  > tests/fixtures/ashby_board.json
```

Confirm each file is non-empty and valid JSON before continuing:

```bash
for f in tests/fixtures/*.json; do python3 -c "import json,sys; json.load(open('$f')); print('$f ok')"; done
```

- [ ] **Step 2: Write the failing test**

```python
# tests/test_ats.py
import json
from pathlib import Path

import pytest

from scorecard.collectors.ats import (
    AtsCollector,
    parse_ashby,
    parse_greenhouse,
    parse_lever,
    summarise,
)
from scorecard.collectors.base import Skip
from scorecard.models import Company

FIXTURES = Path(__file__).parent / "fixtures"


def _company(provider=None, token=None) -> Company:
    return Company.model_validate({
        "slug": "a", "name": "A", "domain": "a.com", "sector": "ai-infra",
        "stage": "seed", "added": "2026-08-29", "cohort": "manual-seed",
        "sources": {"ats": {"provider": provider, "token": token}},
    })


def test_greenhouse_parses_titles_and_locations():
    payload = json.loads((FIXTURES / "greenhouse_jobs.json").read_text())
    jobs = parse_greenhouse(payload)
    assert len(jobs) > 0
    assert all(isinstance(j.title, str) and j.title for j in jobs)


def test_lever_parses_titles_and_locations():
    payload = json.loads((FIXTURES / "lever_postings.json").read_text())
    jobs = parse_lever(payload)
    assert len(jobs) > 0
    assert all(isinstance(j.title, str) and j.title for j in jobs)


def test_ashby_parses_titles_and_locations():
    payload = json.loads((FIXTURES / "ashby_board.json").read_text())
    jobs = parse_ashby(payload)
    assert len(jobs) > 0
    assert all(isinstance(j.title, str) and j.title for j in jobs)


def test_summarise_counts_by_department_and_seniority():
    from scorecard.collectors.ats import Job

    jobs = [
        Job(title="Senior Backend Engineer", location="San Francisco"),
        Job(title="Enterprise Account Executive", location="Remote"),
        Job(title="Head of Sales", location="New York"),
    ]
    data = summarise(jobs, provider="ashby", previous=None)
    assert data["open_roles"] == 3
    assert data["by_department"]["engineering"] == 1
    assert data["by_department"]["sales"] == 2
    assert data["by_seniority"]["exec"] == 1
    assert data["raw_titles"] == [
        "Senior Backend Engineer",
        "Enterprise Account Executive",
        "Head of Sales",
    ]


def test_summarise_computes_week_over_week_diffs():
    from scorecard.collectors.ats import Job

    previous = {"raw_titles": ["Backend Engineer", "Recruiter"]}
    jobs = [Job(title="Backend Engineer", location="Remote"),
            Job(title="Account Executive", location="Remote")]
    data = summarise(jobs, provider="ashby", previous=previous)
    assert data["new_since_last_week"] == ["Account Executive"]
    assert data["closed_since_last_week"] == ["Recruiter"]


def test_summarise_diffs_are_null_with_no_prior_week():
    from scorecard.collectors.ats import Job

    data = summarise([Job(title="X", location="Y")], "ashby", previous=None)
    assert data["new_since_last_week"] is None
    assert data["closed_since_last_week"] is None


def test_no_ats_configured_raises_skip():
    with pytest.raises(Skip, match="no ats"):
        AtsCollector().collect(_company(), client=None)


def test_locations_are_slugified():
    from scorecard.collectors.ats import Job

    data = summarise(
        [Job(title="X", location="San Francisco, CA")], "ashby", previous=None
    )
    assert "san-francisco-ca" in data["by_location"]
```

- [ ] **Step 3: Run test to verify it fails**

Run: `uv run pytest tests/test_ats.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.ats'`

- [ ] **Step 4: Write `scorecard/collectors/ats.py`**

```python
"""Greenhouse / Lever / Ashby job boards.

The highest-signal collector. Hiring mix — the ratio of engineering to
go-to-market roles and how it shifts — is the best public proxy for a company
finding product-market fit and starting to commercialise.

Stores raw_titles alongside the counts so a better classifier can be applied
retroactively.
"""

from __future__ import annotations

import re
from collections import Counter
from dataclasses import dataclass
from typing import Any

from scorecard.collectors.base import Collector, Partial, Skip
from scorecard.collectors.taxonomy import (
    DEPARTMENTS,
    SENIORITIES,
    classify_department,
    classify_seniority,
)
from scorecard.http import PoliteClient
from scorecard.models import Company

GREENHOUSE_URL = "https://boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true"
LEVER_URL = "https://api.lever.co/v0/postings/{token}?mode=json"
ASHBY_URL = "https://api.ashbyhq.com/posting-api/job-board/{token}"

MIN_CAREERS_TEXT = 200


@dataclass(frozen=True)
class Job:
    title: str
    location: str


def _slugify(value: str) -> str:
    slug = re.sub(r"[^a-z0-9]+", "-", value.lower()).strip("-")
    return slug or "unknown"


def parse_greenhouse(payload: dict[str, Any]) -> list[Job]:
    return [
        Job(
            title=(j.get("title") or "").strip(),
            location=((j.get("location") or {}).get("name") or "").strip(),
        )
        for j in payload.get("jobs", [])
        if (j.get("title") or "").strip()
    ]


def parse_lever(payload: list[dict[str, Any]]) -> list[Job]:
    jobs = []
    for j in payload:
        title = (j.get("text") or "").strip()
        if not title:
            continue
        categories = j.get("categories") or {}
        location = (categories.get("location") or "").strip()
        jobs.append(Job(title=title, location=location))
    return jobs


def parse_ashby(payload: dict[str, Any]) -> list[Job]:
    return [
        Job(
            title=(j.get("title") or "").strip(),
            location=(j.get("location") or "").strip(),
        )
        for j in payload.get("jobs", [])
        if (j.get("title") or "").strip()
    ]


def summarise(
    jobs: list[Job], provider: str, previous: dict[str, Any] | None
) -> dict[str, Any]:
    """Build the ATS snapshot payload. Raw counts only — no rates or deltas.

    new/closed_since_last_week are the SPEC.md-sanctioned exception: they
    require comparing to the previous snapshot at collection time.
    """
    titles = [j.title for j in jobs]

    departments = Counter(classify_department(t) for t in titles)
    seniorities = Counter(classify_seniority(t) for t in titles)
    locations = Counter(_slugify(j.location) for j in jobs if j.location)

    if previous is None:
        new_titles: list[str] | None = None
        closed_titles: list[str] | None = None
    else:
        prev = set(previous.get("raw_titles") or [])
        current = set(titles)
        new_titles = sorted(current - prev)
        closed_titles = sorted(prev - current)

    return {
        "provider": provider,
        "open_roles": len(jobs),
        "by_department": {d: departments.get(d, 0) for d in DEPARTMENTS},
        "by_seniority": {s: seniorities.get(s, 0) for s in SENIORITIES},
        "by_location": dict(sorted(locations.items())),
        "new_since_last_week": new_titles,
        "closed_since_last_week": closed_titles,
        "raw_titles": titles,
    }


class AtsCollector(Collector):
    name = "ats"

    def __init__(self, previous: dict[str, Any] | None = None) -> None:
        self._previous = previous

    def collect(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        provider = company.sources.ats.provider
        token = company.sources.ats.token

        if not provider or not token:
            return self._careers_fallback(company, client)

        if provider == "greenhouse":
            payload = client.get(GREENHOUSE_URL.format(token=token)).json()
            jobs = parse_greenhouse(payload)
        elif provider == "lever":
            payload = client.get(LEVER_URL.format(token=token)).json()
            jobs = parse_lever(payload)
        elif provider == "ashby":
            payload = client.get(ASHBY_URL.format(token=token)).json()
            jobs = parse_ashby(payload)
        else:
            raise ValueError(f"unknown ats provider: {provider}")

        return summarise(jobs, provider, self._previous)

    def _careers_fallback(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        """No ATS: hash the careers page and count job-like links."""
        from selectolax.parser import HTMLParser

        from scorecard.storage import content_hash

        if not company.sources.careers_url:
            raise Skip("no ats provider or token, and no careers_url")

        html = client.get(company.sources.careers_url).text
        tree = HTMLParser(html)
        links = [
            a.text(strip=True)
            for a in tree.css("a")
            if a.attributes.get("href")
            and re.search(
                r"job|career|opening|position|role",
                a.attributes.get("href", ""),
                re.I,
            )
        ]
        titles = [t for t in links if t]
        raise Partial(
            "no ats provider; counted job-like links on careers page",
            {
                "provider": None,
                "open_roles": len(titles),
                "by_department": {d: 0 for d in DEPARTMENTS},
                "by_seniority": {s: 0 for s in SENIORITIES},
                "by_location": {},
                "new_since_last_week": None,
                "closed_since_last_week": None,
                "raw_titles": titles,
                "careers_page_hash": content_hash(html),
            },
        )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_ats.py -v`
Expected: 8 passed

- [ ] **Step 6: Verify against all three providers live**

```bash
uv run python -c "
from scorecard.collectors.ats import AtsCollector
from scorecard.http import PoliteClient
from scorecard.universe import load_universe
c = PoliteClient()
for co in load_universe():
    r = AtsCollector().run(co, c)
    n = (r.data or {}).get('open_roles')
    print(f'{co.slug:20} {r.status.value:8} roles={n} {r.error or \"\"}')
"
```

Expected: at least 8 of 10 report `ok`. Investigate any `error` before moving on
— a wrong board token is the most likely cause and is silent otherwise.

- [ ] **Step 7: Commit**

```bash
git add scorecard/collectors/ats.py tests/test_ats.py tests/fixtures
git commit -m "feat: ATS collector for greenhouse, lever and ashby"
```

---

## Task 9: GitHub collector

**Files:**
- Create: `scorecard/collectors/github.py`
- Test: `tests/test_github.py`

Capped at the 10 most recently pushed repos per org to stay inside rate limits.
This is a real methodological limitation and is disclosed in `RULES.md`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_github.py
import httpx
import pytest
import respx

from scorecard.collectors.base import Skip
from scorecard.collectors.github import MAX_REPOS, GithubCollector
from scorecard.http import PoliteClient
from scorecard.models import Company


def _company(org=None) -> Company:
    return Company.model_validate({
        "slug": "a", "name": "A", "domain": "a.com", "sector": "ai-infra",
        "stage": "seed", "added": "2026-08-29", "cohort": "manual-seed",
        "sources": {"github_org": org},
    })


def test_no_org_raises_skip():
    with pytest.raises(Skip, match="no github_org"):
        GithubCollector().collect(_company(), client=None)


@respx.mock
def test_forks_are_excluded_and_repos_capped(monkeypatch):
    monkeypatch.setenv("SCRAPER_CONTACT_EMAIL", "blacksha@usc.edu")
    repos = [
        {"name": f"r{i}", "fork": False, "stargazers_count": 10,
         "pushed_at": "2026-09-01T00:00:00Z"}
        for i in range(15)
    ]
    repos.append({"name": "forked", "fork": True, "stargazers_count": 999,
                  "pushed_at": "2026-09-02T00:00:00Z"})
    respx.get(url__regex=r".*/orgs/acme/repos.*").mock(
        return_value=httpx.Response(200, json=repos)
    )
    respx.get(url__regex=r".*/commits.*").mock(
        return_value=httpx.Response(200, json=[
            {"sha": "x", "author": {"login": "alice"}},
        ])
    )
    respx.get(url__regex=r".*/releases/latest").mock(
        return_value=httpx.Response(404)
    )

    data = GithubCollector().collect(
        _company("acme"), PoliteClient(min_interval=0.0, check_robots=False)
    )
    assert data["public_repos"] == 15  # forks excluded from the count
    assert data["distinct_repos_touched_28d"] <= MAX_REPOS
    assert data["total_stars"] == 150  # fork's 999 stars not counted
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_github.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.github'`

- [ ] **Step 3: Write `scorecard/collectors/github.py`**

```python
"""Product velocity from the public GitHub REST API.

Many companies have no public GitHub. That is a legitimate skip, and it means
this signal is sparse for AI-application and fintech companies specifically.
Roughly 50% coverage is expected and is not a bug.
"""

from __future__ import annotations

import os
from collections import Counter
from datetime import datetime, timedelta, timezone
from typing import Any

import httpx

from scorecard.collectors.base import Collector, Skip
from scorecard.http import PoliteClient
from scorecard.models import Company

API = "https://api.github.com"
MAX_REPOS = 10
TRAILING_DAYS = 28


def _auth_headers() -> dict[str, str]:
    headers = {"Accept": "application/vnd.github+json"}
    token = os.environ.get("GITHUB_TOKEN", "").strip()
    if token:
        headers["Authorization"] = f"Bearer {token}"
    return headers


class GithubCollector(Collector):
    name = "github"

    def collect(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        org = company.sources.github_org
        if not org:
            raise Skip("no github_org")

        headers = _auth_headers()
        since = (
            datetime.now(timezone.utc) - timedelta(days=TRAILING_DAYS)
        ).isoformat().replace("+00:00", "Z")

        repos = client.get(
            f"{API}/orgs/{org}/repos?per_page=100&type=public",
            headers=headers,
        ).json()
        non_forks = [r for r in repos if not r.get("fork")]

        total_stars = sum(r.get("stargazers_count", 0) for r in non_forks)
        recent = sorted(
            non_forks, key=lambda r: r.get("pushed_at") or "", reverse=True
        )[:MAX_REPOS]

        commits = 0
        contributors: Counter[str] = Counter()
        touched = 0
        for repo in recent:
            name = repo["name"]
            page = client.get(
                f"{API}/repos/{org}/{name}/commits"
                f"?since={since}&per_page=100",
                headers=headers,
            ).json()
            if not isinstance(page, list) or not page:
                continue
            touched += 1
            commits += len(page)
            for commit in page:
                author = (commit.get("author") or {}).get("login")
                if author:
                    contributors[author] += 1

        tag, released_at = self._latest_release(client, org, recent, headers)

        return {
            "public_repos": len(non_forks),
            "total_stars": total_stars,
            "commits_trailing_28d": commits,
            "contributors_trailing_28d": len(contributors),
            "distinct_repos_touched_28d": touched,
            "latest_release_tag": tag,
            "latest_release_at": released_at,
            "repos_scanned": [r["name"] for r in recent],
            "repo_cap_applied": len(non_forks) > MAX_REPOS,
        }

    def _latest_release(
        self,
        client: PoliteClient,
        org: str,
        repos: list[dict[str, Any]],
        headers: dict[str, str],
    ) -> tuple[str | None, str | None]:
        """Most recent release across scanned repos. Absent is normal."""
        best_tag: str | None = None
        best_at: str | None = None
        for repo in repos:
            try:
                release = client.get(
                    f"{API}/repos/{org}/{repo['name']}/releases/latest",
                    headers=headers,
                ).json()
            except httpx.HTTPStatusError:
                continue  # 404 means no releases; common and not an error
            published = release.get("published_at")
            if published and (best_at is None or published > best_at):
                best_at = published
                best_tag = release.get("tag_name")
        return best_tag, best_at
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_github.py -v`
Expected: 2 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/collectors/github.py tests/test_github.py
git commit -m "feat: github collector with 10-repo cap"
```

---

## Task 10: Pricing collector

**Files:**
- Create: `scorecard/collectors/pricing.py`
- Test: `tests/test_pricing.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_pricing.py
import pytest

from scorecard.collectors.base import Skip
from scorecard.collectors.pricing import PricingCollector, parse_pricing


def test_extracts_tiers_prices_and_enterprise_flag():
    html = """
    <html><body>
      <h2>Starter</h2><p>$30 per month</p>
      <h2>Team</h2><p>$99 per month</p>
      <h2>Enterprise</h2><p>Contact sales for pricing</p>
    </body></html>
    """
    data = parse_pricing(html)
    assert data["has_enterprise_tier"] is True
    assert data["has_public_prices"] is True
    assert data["lowest_listed_price_usd"] == 30
    assert data["mentions_contact_sales"] is True
    assert "Starter" in data["tier_names"]
    assert data["tier_count"] == 3


def test_no_prices_shown():
    html = "<html><body><h2>Custom</h2><p>Contact sales</p></body></html>"
    data = parse_pricing(html)
    assert data["has_public_prices"] is False
    assert data["lowest_listed_price_usd"] is None


def test_prices_are_whole_usd_integers_never_floats():
    html = "<html><body><h2>Pro</h2><p>$19.99/mo</p></body></html>"
    data = parse_pricing(html)
    assert data["lowest_listed_price_usd"] == 19
    assert isinstance(data["lowest_listed_price_usd"], int)


def test_no_pricing_url_raises_skip():
    from scorecard.models import Company

    company = Company.model_validate({
        "slug": "a", "name": "A", "domain": "a.com", "sector": "ai-infra",
        "stage": "seed", "added": "2026-08-29", "cohort": "manual-seed",
    })
    with pytest.raises(Skip, match="no pricing_url"):
        PricingCollector().collect(company, client=None)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_pricing.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.pricing'`

- [ ] **Step 3: Write `scorecard/collectors/pricing.py`**

```python
"""Commercialisation signals from the public pricing page.

Pricing pages are heavily JavaScript-rendered. Failure is accepted gracefully:
under MIN_TEXT characters of extracted text yields a partial result carrying the
hash only. No headless browser.
"""

from __future__ import annotations

import re
from typing import Any

from selectolax.parser import HTMLParser

from scorecard.collectors.base import Collector, Partial, Skip
from scorecard.http import PoliteClient
from scorecard.models import Company
from scorecard.storage import content_hash

MIN_TEXT = 200
PRICE_RE = re.compile(r"\$\s?([0-9][0-9,]*)(?:\.[0-9]{2})?")
ENTERPRISE_RE = re.compile(r"\b(enterprise|custom)\b", re.I)
CONTACT_RE = re.compile(r"contact\s+(sales|us)|talk\s+to\s+sales", re.I)
TIER_SELECTOR = "h1, h2, h3, h4"


def parse_pricing(html: str) -> dict[str, Any]:
    """Extract raw pricing facts. Stores values, never derived comparisons."""
    tree = HTMLParser(html)
    for tag in tree.css("script, style"):
        tag.decompose()
    text = " ".join((tree.body.text() if tree.body else "").split())

    headings = [h.text(strip=True) for h in tree.css(TIER_SELECTOR)]
    tier_names = [
        h for h in headings if h and 2 <= len(h) <= 30 and "?" not in h
    ]

    prices = [
        int(m.group(1).replace(",", "")) for m in PRICE_RE.finditer(text)
    ]
    positive = [p for p in prices if p > 0]

    return {
        "content_hash": content_hash(text),
        "tier_names": tier_names,
        "tier_count": len(tier_names),
        "has_enterprise_tier": bool(
            ENTERPRISE_RE.search(" ".join(tier_names))
        ),
        "has_public_prices": bool(positive),
        "lowest_listed_price_usd": min(positive) if positive else None,
        "mentions_contact_sales": bool(CONTACT_RE.search(text)),
        "extracted_text_length": len(text),
    }


class PricingCollector(Collector):
    name = "pricing"

    def __init__(
        self,
        previous: dict[str, Any] | None = None,
        iso_week: str | None = None,
    ) -> None:
        self._previous = previous
        self._iso_week = iso_week

    def collect(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        url = company.sources.pricing_url
        if not url:
            raise Skip("no pricing_url")

        html = client.get(url).text
        data = parse_pricing(html)

        previous_hash = (self._previous or {}).get("content_hash")
        data["changed_since_last_week"] = (
            None if previous_hash is None
            else data["content_hash"] != previous_hash
        )

        if self._iso_week:
            from scorecard.storage import archive_raw_if_changed

            data["raw_archived_path"] = archive_raw_if_changed(
                company.slug, "pricing", self._iso_week, html, previous_hash
            )
        else:
            data["raw_archived_path"] = None

        if data["extracted_text_length"] < MIN_TEXT:
            raise Partial(
                f"only {data['extracted_text_length']} chars extracted; "
                "page is likely JavaScript-rendered",
                data,
            )
        return data
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_pricing.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/collectors/pricing.py tests/test_pricing.py
git commit -m "feat: pricing collector with hash-gated archiving"
```

---

## Task 11: Changelog collector, with enterprise-readiness flags

**Files:**
- Create: `scorecard/collectors/changelog.py`
- Test: `tests/test_changelog.py`

This task carries the addition from design spec §9. The enterprise-readiness
regex runs over the sitemap that is **already being fetched** for
`docs_page_count`, so it costs zero additional requests. Nobody builds SAML for
fun; its documentation appearing is the clearest public signal that a company
has landed or is chasing enterprise customers.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_changelog.py
from scorecard.collectors.changelog import (
    count_entries_trailing_28d,
    enterprise_flags,
    parse_sitemap,
)


def test_sitemap_urls_are_parsed():
    xml = """<?xml version="1.0"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      <url><loc>https://x.com/docs/intro</loc></url>
      <url><loc>https://x.com/docs/auth/saml</loc></url>
      <url><loc>https://x.com/blog/hello</loc></url>
    </urlset>"""
    urls = parse_sitemap(xml)
    assert len(urls) == 3
    assert "https://x.com/docs/auth/saml" in urls


def test_docs_pages_counted_under_docs_path():
    xml = """<?xml version="1.0"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      <url><loc>https://x.com/docs/a</loc></url>
      <url><loc>https://x.com/docs/b</loc></url>
      <url><loc>https://x.com/blog/c</loc></url>
    </urlset>"""
    urls = parse_sitemap(xml)
    assert len([u for u in urls if "/docs/" in u]) == 2


def test_enterprise_flags_detected_from_urls():
    urls = [
        "https://x.com/docs/auth/saml",
        "https://x.com/docs/security/audit-logs",
        "https://x.com/docs/intro",
    ]
    flags = enterprise_flags(urls)
    assert flags["has_saml_docs"] is True
    assert flags["has_audit_log_docs"] is True
    assert flags["has_scim_docs"] is False
    assert flags["enterprise_signal_count"] == 2


def test_enterprise_flags_all_false_when_absent():
    flags = enterprise_flags(["https://x.com/docs/intro"])
    assert flags["enterprise_signal_count"] == 0
    assert flags["has_sso_docs"] is False


def test_sso_matches_word_not_substring():
    """'sso' must not fire on words like 'lesson' or 'assonance'."""
    flags = enterprise_flags(["https://x.com/docs/lesson-one"])
    assert flags["has_sso_docs"] is False


def test_entries_within_28_days_counted():
    from datetime import datetime, timedelta, timezone

    now = datetime(2026, 9, 7, tzinfo=timezone.utc)
    dates = [
        now - timedelta(days=3),
        now - timedelta(days=20),
        now - timedelta(days=40),
    ]
    assert count_entries_trailing_28d(dates, now=now) == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_changelog.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collectors.changelog'`

- [ ] **Step 3: Write `scorecard/collectors/changelog.py`**

```python
"""Shipping cadence, docs surface area, and enterprise-readiness signals.

Entry dates are tried in order: RSS/Atom feed, then HTML date patterns near
headings. docs_page_count comes from sitemap.xml, which costs one request and
is a surprisingly good proxy for product surface area growth.

The enterprise-readiness flags reuse that same sitemap fetch, so they are free.
"""

from __future__ import annotations

import re
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta, timezone
from typing import Any

import httpx
from selectolax.parser import HTMLParser

from scorecard.collectors.base import Collector, Skip
from scorecard.http import PoliteClient
from scorecard.models import Company

TRAILING_DAYS = 28
SITEMAP_NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"
DATE_RE = re.compile(r"(\d{4})-(\d{2})-(\d{2})")

# Word-boundary matched against the URL path so "sso" does not fire on
# "lesson". Each entry is (flag name, pattern).
_ENTERPRISE_PATTERNS: tuple[tuple[str, str], ...] = (
    ("has_sso_docs", r"\bsso\b|\bsingle-sign-on\b"),
    ("has_saml_docs", r"\bsaml\b"),
    ("has_scim_docs", r"\bscim\b|\bdirectory-sync\b"),
    ("has_audit_log_docs", r"\baudit-logs?\b|\baudit-trail\b"),
    ("has_compliance_docs", r"\bsoc-?2\b|\bhipaa\b|\bgdpr\b|\bcompliance\b"),
    ("has_rbac_docs", r"\brbac\b|\brole-based\b|\bpermissions\b"),
)


def parse_sitemap(xml: str) -> list[str]:
    """Return every <loc> URL. Malformed XML yields an empty list."""
    try:
        root = ET.fromstring(xml)
    except ET.ParseError:
        return []
    return [
        el.text.strip()
        for el in root.iter(f"{SITEMAP_NS}loc")
        if el.text and el.text.strip()
    ]


def enterprise_flags(urls: list[str]) -> dict[str, Any]:
    """Detect enterprise-readiness documentation. Zero extra requests.

    Raw booleans only. Whether these constitute "enterprise motion" is a
    scoring-time judgment, computed later by ledger/.
    """
    haystack = " ".join(u.lower() for u in urls)
    flags: dict[str, Any] = {}
    count = 0
    for name, pattern in _ENTERPRISE_PATTERNS:
        hit = bool(re.search(pattern, haystack))
        flags[name] = hit
        count += int(hit)
    flags["enterprise_signal_count"] = count
    return flags


def count_entries_trailing_28d(
    dates: list[datetime], now: datetime | None = None
) -> int:
    reference = now or datetime.now(timezone.utc)
    cutoff = reference - timedelta(days=TRAILING_DAYS)
    return sum(1 for d in dates if d >= cutoff)


def _extract_dates(text: str) -> list[datetime]:
    dates = []
    for match in DATE_RE.finditer(text):
        try:
            dates.append(
                datetime(
                    int(match.group(1)), int(match.group(2)),
                    int(match.group(3)), tzinfo=timezone.utc,
                )
            )
        except ValueError:
            continue
    return dates


class ChangelogCollector(Collector):
    name = "changelog"

    def collect(
        self, company: Company, client: PoliteClient
    ) -> dict[str, Any]:
        if not company.sources.changelog_url and not company.sources.docs_url:
            raise Skip("no changelog_url or docs_url")

        entries = 0
        latest: str | None = None
        if company.sources.changelog_url:
            entries, latest = self._changelog(
                client, company.sources.changelog_url
            )

        docs_count = 0
        flags = enterprise_flags([])
        sitemap_url = f"https://{company.domain}/sitemap.xml"
        try:
            urls = parse_sitemap(client.get(sitemap_url).text)
        except httpx.HTTPError:
            urls = []
        if urls:
            docs_count = len([u for u in urls if "/docs" in u])
            flags = enterprise_flags(urls)

        return {
            "entries_trailing_28d": entries,
            "latest_entry_at": latest,
            "docs_page_count": docs_count,
            "sitemap_url_count": len(urls),
            **flags,
        }

    def _changelog(
        self, client: PoliteClient, url: str
    ) -> tuple[int, str | None]:
        body = client.get(url).text
        dates = _extract_dates(body)
        if not dates:
            tree = HTMLParser(body)
            dates = _extract_dates(tree.body.text() if tree.body else "")
        if not dates:
            return 0, None
        latest = max(dates)
        return (
            count_entries_trailing_28d(dates),
            latest.isoformat().replace("+00:00", "Z"),
        )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_changelog.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add scorecard/collectors/changelog.py tests/test_changelog.py
git commit -m "feat: changelog collector with enterprise-readiness flags"
```

---

## Task 12: Orchestration and CLI

**Files:**
- Create: `scorecard/collect.py`, `scorecard/validate.py`, `scorecard/cli.py`
- Test: `tests/test_collect.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_collect.py
from scorecard.collect import collect_company
from scorecard.models import Company, Status


def _company() -> Company:
    return Company.model_validate({
        "slug": "a", "name": "A", "domain": "a.com", "sector": "ai-infra",
        "stage": "seed", "added": "2026-08-29", "cohort": "manual-seed",
    })


class _Exploding:
    name = "ats"

    def run(self, company, client):
        raise RuntimeError("this must never escape")


def test_snapshot_always_has_all_six_collector_keys():
    snap = collect_company(_company(), "2026-W36", client=None, only=None)
    assert set(snap.collectors) == {
        "github", "ats", "pricing", "changelog", "edgar", "linkedin"
    }


def test_uncollected_collectors_are_skipped_not_missing():
    snap = collect_company(_company(), "2026-W36", client=None, only=["ats"])
    assert snap.collectors["edgar"].status == Status.SKIPPED
    assert snap.collectors["edgar"].error is not None


def test_a_collector_raising_outside_run_still_does_not_abort(monkeypatch):
    import scorecard.collect as mod

    monkeypatch.setattr(
        mod, "build_collectors", lambda *a, **k: {"ats": _Exploding()}
    )
    snap = collect_company(_company(), "2026-W36", client=None, only=["ats"])
    assert snap.collectors["ats"].status == Status.ERROR
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_collect.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.collect'`

- [ ] **Step 3: Write `scorecard/collect.py`**

```python
"""Orchestration. Owns the guarantee that nothing aborts anything else."""

from __future__ import annotations

import logging
from typing import Any

from scorecard.collectors.ats import AtsCollector
from scorecard.collectors.base import Collector
from scorecard.collectors.changelog import ChangelogCollector
from scorecard.collectors.github import GithubCollector
from scorecard.collectors.pricing import PricingCollector
from scorecard.http import PoliteClient
from scorecard.models import COLLECTOR_NAMES, CollectorResult, Company, Snapshot, Status
from scorecard.storage import load_snapshot, write_snapshot

logger = logging.getLogger(__name__)


def build_collectors(
    previous: Snapshot | None,
    iso_week: str,
) -> dict[str, Collector]:
    """Instantiate the collectors implemented so far.

    edgar and linkedin are Week 2. They are absent here and therefore recorded
    as skipped, which is exactly the intended distinction: not collected is not
    the same as does not exist.
    """

    def prior(name: str) -> dict[str, Any] | None:
        if previous is None:
            return None
        return previous.collectors.get(name, CollectorResult(
            status=Status.SKIPPED, error="absent"
        )).data

    return {
        "ats": AtsCollector(previous=prior("ats")),
        "github": GithubCollector(),
        "pricing": PricingCollector(
            previous=prior("pricing"), iso_week=iso_week
        ),
        "changelog": ChangelogCollector(),
    }


def collect_company(
    company: Company,
    iso_week: str,
    client: PoliteClient | None,
    only: list[str] | None,
    previous: Snapshot | None = None,
) -> Snapshot:
    """Run every configured collector for one company.

    Never raises. A collector that fails outside its own error envelope is
    still caught here, so one bad collector cannot lose the other five.
    """
    snapshot = Snapshot(slug=company.slug, iso_week=iso_week)
    collectors = build_collectors(previous, iso_week)

    for name in COLLECTOR_NAMES:
        if name not in collectors:
            snapshot.collectors[name] = CollectorResult(
                status=Status.SKIPPED, error="collector not implemented yet"
            )
            continue
        if only and name not in only:
            snapshot.collectors[name] = CollectorResult(
                status=Status.SKIPPED, error="not selected by --only"
            )
            continue
        try:
            snapshot.collectors[name] = collectors[name].run(company, client)
        except Exception as exc:  # noqa: BLE001 — last line of defence
            logger.error("%s %s escaped its envelope: %s", company.slug, name, exc)
            snapshot.collectors[name] = CollectorResult(
                status=Status.ERROR, error=f"{type(exc).__name__}: {exc}"[:500]
            )
    return snapshot


def collect_all(
    companies: list[Company],
    iso_week: str,
    only: list[str] | None,
    force: bool,
    dry_run: bool,
) -> int:
    """Run every company. Returns the number of snapshots written.

    Resumable: a company already written for this week is skipped unless
    --force, so a crash at company 90 does not mean redoing 89.
    """
    from scorecard.storage import snapshot_exists

    client = PoliteClient()
    written = 0
    try:
        for company in companies:
            if not force and snapshot_exists(company.slug, iso_week):
                logger.info("%s already collected for %s", company.slug, iso_week)
                continue
            previous = _previous_snapshot(company.slug, iso_week)
            snapshot = collect_company(
                company, iso_week, client, only, previous
            )
            for name, result in snapshot.collectors.items():
                print(
                    f"{iso_week} {company.slug} {name} {result.status.value} "
                    f"{result.error or ''}".rstrip()
                )
            if not dry_run:
                write_snapshot(snapshot)
                written += 1
    finally:
        client.close()
    return written


def _previous_snapshot(slug: str, iso_week: str) -> Snapshot | None:
    """Load the immediately preceding ISO week, if it exists.

    Uses real date arithmetic rather than decrementing the week number, so the
    year boundary works: the week before 2027-W01 is 2026-W52 or 2026-W53
    depending on the year, and naive subtraction silently returns None there —
    losing every week-over-week diff for one week each January.
    """
    from datetime import date, timedelta

    year, week = int(iso_week[:4]), int(iso_week[6:])
    monday = date.fromisocalendar(year, week, 1)
    prior = monday - timedelta(days=7)
    prior_year, prior_week, _ = prior.isocalendar()
    return load_snapshot(slug, f"{prior_year}-W{prior_week:02d}")
```

- [ ] **Step 4: Write `scorecard/validate.py`**

```python
"""Coverage report. Answers: how many companies got a good pull from each."""

from __future__ import annotations

import json
from collections import Counter
from pathlib import Path

from scorecard.models import COLLECTOR_NAMES
from scorecard.storage import DATA_ROOT

COVERAGE_BAR = {"ats": 0.80, "github": 0.50, "pricing": 0.80}


def coverage_report(iso_week: str, root: Path = DATA_ROOT) -> dict:
    """Count ok/partial/error/skipped per collector for one week."""
    directory = root / "snapshots" / iso_week
    files = sorted(directory.glob("*.json")) if directory.exists() else []

    per_collector: dict[str, Counter[str]] = {
        name: Counter() for name in COLLECTOR_NAMES
    }
    for path in files:
        payload = json.loads(path.read_text())
        for name in COLLECTOR_NAMES:
            status = payload["collectors"][name]["status"]
            per_collector[name][status] += 1

    total = len(files)
    return {
        "iso_week": iso_week,
        "companies": total,
        "collectors": {
            name: {
                "ok": counts["ok"],
                "partial": counts["partial"],
                "error": counts["error"],
                "skipped": counts["skipped"],
                "coverage": (counts["ok"] / total) if total else 0.0,
            }
            for name, counts in per_collector.items()
        },
    }


def write_coverage(report: dict, root: Path = DATA_ROOT) -> Path:
    path = root / "coverage" / f"{report['iso_week']}.json"
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(report, indent=2, sort_keys=True) + "\n")
    return path
```

- [ ] **Step 5: Write `scorecard/cli.py`**

```python
"""Typer entrypoint. One parseable line per company per collector to stdout."""

from __future__ import annotations

import logging

import typer

from scorecard.collect import collect_all
from scorecard.storage import current_iso_week
from scorecard.universe import load_universe
from scorecard.validate import COVERAGE_BAR, coverage_report, write_coverage

app = typer.Typer(add_completion=False)


@app.command()
def collect(
    all_: bool = typer.Option(False, "--all", help="every company"),
    company: str | None = typer.Option(None, help="single slug, for debugging"),
    only: str | None = typer.Option(None, help="comma-separated collectors"),
    force: bool = typer.Option(False, help="re-collect companies already done"),
    dry_run: bool = typer.Option(False, "--dry-run", help="fetch, print, write nothing"),
) -> None:
    logging.basicConfig(level=logging.INFO, format="%(message)s")
    companies = load_universe()
    if company:
        companies = [c for c in companies if c.slug == company]
        if not companies:
            typer.echo(f"no company with slug {company!r}", err=True)
            raise typer.Exit(1)
    elif not all_:
        typer.echo("pass --all or --company SLUG", err=True)
        raise typer.Exit(1)

    week = current_iso_week()
    selected = [s.strip() for s in only.split(",")] if only else None
    written = collect_all(companies, week, selected, force, dry_run)
    typer.echo(f"wrote {written} snapshots for {week}")


@app.command()
def validate() -> None:
    """Coverage report for the current week. Exits non-zero if it collapsed."""
    week = current_iso_week()
    report = coverage_report(week)
    write_coverage(report)

    typer.echo(f"{week}: {report['companies']} companies")
    for name, stats in report["collectors"].items():
        typer.echo(
            f"  {name:10} ok={stats['ok']:3} partial={stats['partial']:3} "
            f"error={stats['error']:3} skipped={stats['skipped']:3} "
            f"coverage={stats['coverage']:.0%}"
        )

    if report["companies"] == 0:
        typer.echo("FAIL: zero companies collected", err=True)
        raise typer.Exit(1)
    for name, floor in COVERAGE_BAR.items():
        actual = report["collectors"][name]["coverage"]
        if actual < floor / 2:
            typer.echo(
                f"FAIL: {name} coverage {actual:.0%} is less than half its "
                f"{floor:.0%} target — something structural broke",
                err=True,
            )
            raise typer.Exit(1)


if __name__ == "__main__":
    app()
```

- [ ] **Step 6: Run tests and the full suite**

Run: `uv run pytest -v`
Expected: all pass

- [ ] **Step 7: Run it for real, twice, to prove idempotence**

```bash
uv run python -m scorecard.cli collect --all
uv run python -m scorecard.cli collect --all      # second run skips everything
ls data/snapshots/*/ | wc -l                      # 10 files, not 20
uv run python -m scorecard.cli validate
```

Expected: second run writes 0 snapshots. Exactly 10 JSON files.

- [ ] **Step 8: Commit**

```bash
git add scorecard/collect.py scorecard/validate.py scorecard/cli.py tests/test_collect.py
git commit -m "feat: collection orchestration, coverage report and CLI"
```

---

## Task 13: Scheduled workflow

**Files:**
- Create: `.github/workflows/collect.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: collect

on:
  schedule:
    - cron: "0 9 * * 1"   # Mondays 09:00 UTC
  workflow_dispatch:

permissions:
  contents: write

jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Collect
        env:
          SCRAPER_CONTACT_EMAIL: ${{ secrets.SCRAPER_CONTACT_EMAIL }}
          GITHUB_TOKEN: ${{ secrets.GH_READ_TOKEN }}
        run: uv run python -m scorecard.cli collect --all

      - name: Coverage
        env:
          SCRAPER_CONTACT_EMAIL: ${{ secrets.SCRAPER_CONTACT_EMAIL }}
        run: uv run python -m scorecard.cli validate

      - name: Commit
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/
          if git diff --staged --quiet; then
            echo "no changes"
          else
            git commit -m "data: snapshot $(date -u +%G-W%V)"
            git push
          fi
```

Note: individual collector failures do not fail this job — they are recorded in
the snapshot. Only `validate` fails it, and only on structural collapse.

- [ ] **Step 2: Add the repository secrets**

```bash
gh secret set SCRAPER_CONTACT_EMAIL --body "blacksha@usc.edu"
gh secret set GH_READ_TOKEN --body "<a classic PAT with public_repo scope>"
```

The secret is named `GH_READ_TOKEN` because GitHub reserves the `GITHUB_`
prefix for its own secrets and will reject `GITHUB_TOKEN`.

- [ ] **Step 3: Commit, push, and trigger a run**

```bash
git add .github/workflows/collect.yml
git commit -m "ci: weekly collection workflow"
git push
gh workflow run collect
sleep 60 && gh run list --workflow=collect --limit 1
```

Expected: the run completes green and a `data: snapshot 2026-W35` commit
appears. If it fails, read the logs with `gh run view --log-failed`.

- [ ] **Step 4: Verify the commit landed**

```bash
git pull && ls data/snapshots/
```

Expected: a directory for the current ISO week containing 10 JSON files.

---

## Task 14: RULES.md

**Files:**
- Create: `RULES.md`

`SPEC.md` requires this before collecting anything in earnest. It is also the
document a VC will actually read.

- [ ] **Step 1: Write `RULES.md`**

It must state, at minimum:

1. Every data source used, and what each can and cannot show.
2. **The GitHub 10-repo cap**, and that it undercounts orgs with many active
   repos.
3. **Form D shows offering amount and security type — not valuation, not the
   round label, and not a clean priced-vs-SAFE distinction.** Downstream logic
   must not pretend otherwise. (Applies from Week 2.)
4. The universe is frozen, additions are dated, nothing is ever deleted.
5. Headcount comes from a third-party scrape and may lag reality. (Week 2.)
6. **No prediction or score ever uses confidential or non-public information
   from any affiliation or employer.** Not optional. A public timestamped record
   is permanent, and that cuts both ways.
7. The repo is currently unlisted, and when it is published, everything
   publishes — including the early bad output.
8. **Known classifier limitation:** `Founding Engineer` is classified `exec`
   per `SPEC.md`, which is probably wrong and inflates exec counts at seed
   companies. `raw_titles` is stored so this can be corrected retroactively.
9. **Revenue is a named blind spot.** No public revenue data exists for private
   seed-stage companies, and no estimated figure is used.
10. Politeness: identifying User-Agent with a real email on every request,
    robots.txt respected, 1 req/sec per domain, only public unauthenticated
    pages, never authenticating to scrape.

- [ ] **Step 2: Commit**

```bash
git add RULES.md
git commit -m "docs: public methodology and honesty commitments"
git push
```

---

## Task 15: Frozen-field CI check

**Files:**
- Create: `scorecard/freeze_check.py`, `.github/workflows/universe.yml`
- Test: `tests/test_freeze_check.py`

`SPEC.md` requires a CI check that fails if an existing entry's `slug`, `added`
or `cohort` is modified. This is the mechanism that prevents survivorship bias:
without it a company can be quietly back-dated after it starts doing well, and
the cohort discipline becomes decorative.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_freeze_check.py
import pytest

from scorecard.freeze_check import FrozenFieldViolation, check_frozen_fields

BEFORE = [
    {"slug": "a", "name": "A", "added": "2026-08-29", "cohort": "yc-w26"},
    {"slug": "b", "name": "B", "added": "2026-08-29", "cohort": "manual-seed"},
]


def test_adding_a_new_entry_is_allowed():
    after = BEFORE + [
        {"slug": "c", "name": "C", "added": "2026-09-05", "cohort": "yc-s26"}
    ]
    check_frozen_fields(BEFORE, after)


def test_editing_a_non_frozen_field_is_allowed():
    after = [dict(BEFORE[0], name="A Inc"), BEFORE[1]]
    check_frozen_fields(BEFORE, after)


def test_changing_added_is_rejected():
    after = [dict(BEFORE[0], added="2026-07-01"), BEFORE[1]]
    with pytest.raises(FrozenFieldViolation, match="added"):
        check_frozen_fields(BEFORE, after)


def test_changing_cohort_is_rejected():
    after = [dict(BEFORE[0], cohort="form-d-rolling"), BEFORE[1]]
    with pytest.raises(FrozenFieldViolation, match="cohort"):
        check_frozen_fields(BEFORE, after)


def test_deleting_an_entry_is_rejected():
    with pytest.raises(FrozenFieldViolation, match="removed"):
        check_frozen_fields(BEFORE, [BEFORE[0]])
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_freeze_check.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'scorecard.freeze_check'`

- [ ] **Step 3: Write `scorecard/freeze_check.py`**

```python
"""Enforce that universe entries are append-only in their identifying fields.

slug, added and cohort are permanent. Allowing them to change would permit
back-dating a company after it starts doing well, which is exactly the
survivorship bias the cohort discipline exists to prevent.
"""

from __future__ import annotations

import subprocess
import sys
from typing import Any

import yaml

FROZEN_FIELDS = ("slug", "added", "cohort")
UNIVERSE_PATH = "companies/universe.yaml"


class FrozenFieldViolation(Exception):
    """Raised when a permanent field changed or an entry disappeared."""


def check_frozen_fields(
    before: list[dict[str, Any]], after: list[dict[str, Any]]
) -> None:
    """Raise if any existing entry lost or changed a frozen field."""
    old = {e["slug"]: e for e in before}
    new = {e["slug"]: e for e in after}

    removed = set(old) - set(new)
    if removed:
        raise FrozenFieldViolation(
            f"entries removed from the universe: {sorted(removed)}. "
            "Nothing is ever deleted."
        )

    for slug, old_entry in old.items():
        new_entry = new[slug]
        for field in FROZEN_FIELDS:
            if old_entry.get(field) != new_entry.get(field):
                raise FrozenFieldViolation(
                    f"{slug}: {field} changed from "
                    f"{old_entry.get(field)!r} to {new_entry.get(field)!r}. "
                    "This field is permanent."
                )


def _load_from_git(ref: str) -> list[dict[str, Any]]:
    result = subprocess.run(
        ["git", "show", f"{ref}:{UNIVERSE_PATH}"],
        capture_output=True,
        text=True,
    )
    if result.returncode != 0:
        return []  # file did not exist at that ref
    return yaml.safe_load(result.stdout) or []


def main() -> int:
    base = sys.argv[1] if len(sys.argv) > 1 else "origin/main"
    before = _load_from_git(base)
    after = yaml.safe_load(open(UNIVERSE_PATH).read()) or []
    try:
        check_frozen_fields(before, after)
    except FrozenFieldViolation as exc:
        print(f"FROZEN FIELD VIOLATION: {exc}", file=sys.stderr)
        return 1
    print(f"universe ok: {len(after)} entries, no frozen fields changed")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_freeze_check.py -v`
Expected: 5 passed

- [ ] **Step 5: Write `.github/workflows/universe.yml`**

```yaml
name: universe

on:
  pull_request:
    paths: ["companies/universe.yaml"]
  push:
    branches: [main]
    paths: ["companies/universe.yaml"]

jobs:
  freeze-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: astral-sh/setup-uv@v3

      - name: Frozen field check
        run: uv run python -m scorecard.freeze_check origin/main
```

- [ ] **Step 6: Prove the check actually fails on a violation**

A guard that has never fired is a guard you do not know works.

```bash
cp companies/universe.yaml /tmp/universe.bak
python3 -c "
import yaml
u = yaml.safe_load(open('companies/universe.yaml'))
u[0]['added'] = '2020-01-01'
yaml.safe_dump(u, open('companies/universe.yaml','w'), sort_keys=False)
"
uv run python -m scorecard.freeze_check HEAD; echo "exit=$?"
cp /tmp/universe.bak companies/universe.yaml
git diff --stat companies/
```

Expected: prints `FROZEN FIELD VIOLATION: ... added changed ...` and `exit=1`.
The final `git diff --stat` must show no changes — confirm the file was restored.

- [ ] **Step 7: Commit**

```bash
git add scorecard/freeze_check.py tests/test_freeze_check.py .github/workflows/universe.yml
git commit -m "ci: enforce frozen slug, added and cohort fields"
git push
```

---

## Done when

- [ ] `uv run pytest` passes
- [ ] `uv run ruff check . && uv run ruff format --check .` passes
- [ ] `uv run python -m scorecard.cli collect --all` writes 10 snapshots
- [ ] Running it a second time writes 0 — idempotent
- [ ] `uv run python -m scorecard.cli validate` reports ≥80% ok for ats and
      pricing across the 10 test companies
- [ ] The `collect` workflow has completed green at least once and committed
- [ ] Every snapshot contains all six collector keys, including `edgar` and
      `linkedin` as `skipped`
- [ ] `RULES.md` exists and covers all ten points
- [ ] `scorecard.freeze_check` exits 1 on a deliberately corrupted universe entry

## Not in this plan

EDGAR, LinkedIn/Apify, `customers.py`, universe expansion to 150, `stamp.yml`
and OpenTimestamps — all Week 2 (Sep 1–7). The entire `ledger/` package gets its
own plan and is built Sep 8–14.
