# Local Odysseus Fixes

These patches are maintained locally until equivalent fixes land in upstream
`dev`. After each upstream update, check whether each issue is fixed before
reapplying it.

Last checked against upstream `dev` commit `b51d83b` on June 18, 2026.

Retired stopgaps:

- Moonshot/Kimi `reasoning_content` preservation was merged in PR #3152
  (`2e6fff2`). Do not reapply or resubmit it.
- ChatGPT Subscription `max_output_tokens` removal was merged in PR #3656
  (`9e74a32`). Do not reapply or resubmit it.
- Moonshot/Kimi `temperature` requirement was merged in PR #3960
  (`75268e7`). Do not reapply or resubmit it.
- Memory tidy request timeout was merged in PR #3886
  (`ebd2332`). Do not reapply or resubmit it.
- Complete memory listing was merged in PR #3885
  (`ebd2332`). Do not reapply or resubmit it.
- Memory tidy OAuth credentials was merged in PR #4313
  (`2b519bf`). Do not reapply or resubmit it.

The one application fix below remains absent from upstream as a local patch. The Dockge updater and compose overrides are local deployment tooling,
not upstream application patches.

Upstream PR branches last rebased locally on June 16, 2026:

- `codex/dynamic-tool-awareness-compact` — local patch, fixes issue #3934
- `fix/caldav-unique-calendar-id-migration` — local patch (commit `9cfaad2`), fixes CalDAV UNIQUE crash-loop on calendar ID migration

Before reapplying one of these stopgaps after a future update, check whether its
branch was submitted and merged upstream. Once merged, remove the corresponding
local diff and numbered entry from this file.

## Agent handoff

Read this entire file before updating either checkout. The local and server
working trees intentionally contain uncommitted changes. Do not run
`git reset --hard`, `git clean`, or discard a modified file just to make
`git pull` succeed.

### Environment

- Local checkout: `/Users/joshp/Projects/odysseus`
- Local branch: `dev`
- Server: `root@192.168.1.176`
- SSH identity: `/Users/joshp/.ssh/id_ed25519`
- Server checkout: `/opt/stacks/odysseus`
- Dockge: `http://192.168.1.176:5001`
- Application: `http://192.168.1.176:7000`
- Backup root: `/opt/stacks/backups`
- Canonical Dockge compose:
  `deploy/docker-compose.dockge.yml`
- Active Dockge compose: `docker-compose.yml`

The canonical compose publishes only Odysseus on port 7000. ChromaDB and ntfy
must remain accessible only on the Compose network. It also sets
`FASTEMBED_CACHE_PATH` to `/app/data/fastembed_cache`; do not replace that
default with an empty value, because FastEmbed cannot initialize a cache at
`""`. After any interrupted update, restore the active compose before running
Docker:

```bash
cd /opt/stacks/odysseus
cp deploy/docker-compose.dockge.yml docker-compose.yml
docker compose up -d --build --remove-orphans
```

### Expected dirty files

These files normally differ from upstream:

```text
src/agent_loop.py
docker-compose.yml
```

These local deployment files are intentionally untracked:

```text
FIXES.md
deploy/
docker-compose.dockge.yml
```

Upstream refactors may move a fix. Search the entire new tree before concluding
that a patch is missing or reapplying it to an obsolete path.

Before opening any PR, search merged commits and all open/closed PRs. A nearby
PR touching the same file is not automatically a duplicate; inspect its body
and changed files. As of June 10, 2026, PRs #1941, #3566, and #1747 touch
memory code but do not address the OAuth resolver, request timeout, or complete
memory listing fixes documented below.

## Update procedure

### 1. Inspect both working trees

```bash
cd /Users/joshp/Projects/odysseus
git status --short --branch
git log -1 --oneline

ssh -i /Users/joshp/.ssh/id_ed25519 root@192.168.1.176 \
  'cd /opt/stacks/odysseus && git status --short --branch && git log -1 --oneline'
```

Do not assume local and server commits match. The goal at completion is for
both to use the same upstream `dev` commit and the same patched file contents.

### 2. Run the server updater

```bash
ssh -i /Users/joshp/.ssh/id_ed25519 root@192.168.1.176
cd /opt/stacks/odysseus
./deploy/update-odysseus-dockge.sh
```

The updater:

1. Keeps only one backup.
2. Backs up `.env`, persistent data, and logs.
3. Excludes rebuildable caches and installed runtime trees.
4. Restores the tracked upstream `docker-compose.yml` before pulling.
5. Pulls `dev`.
6. Replaces `docker-compose.yml` with the canonical Dockge compose.
7. Rebuilds and restarts the stack.

Because the fixes are intentionally uncommitted, the pull may stop with
`Your local changes would be overwritten`. This is expected. The backup has
already completed, but the active `docker-compose.yml` may now be the upstream
file. Continue with the conflict procedure below; do not repeatedly rerun the
updater and create another large backup.

### 3. Check upstream before reapplying patches

Fetch first, then search `origin/dev` for the behavior described by every fix:

```bash
cd /opt/stacks/odysseus
git fetch origin dev
git log --oneline HEAD..origin/dev
git diff --name-status HEAD..origin/dev
git grep -n -E \
  'moonshot|reasoning_content|max_output_tokens|memory/audit|memories\[:100\]|manage_memory' \
  origin/dev -- app.py routes src mcp_servers tests
```

For each numbered fix below:

1. Inspect the upstream implementation, not just commit titles.
2. If upstream fully fixes and tests the behavior, do not reapply the local
   patch. Remove that item from the expected patch set and update this file.
3. If upstream only changes adjacent code, port the local behavior into the new
   implementation.
4. Record the checked upstream commit and date near the top of this file.

### 4. Rebase the local patches

Stash only the known patched source files. Do not stash `.env`, persistent data,
deployment files, or unrelated user changes.

```bash
cd /Users/joshp/Projects/odysseus

git stash push -m codex-local-fixes-YYYYMMDD -- \
  routes/memory_routes.py \
  src/agent_loop.py

git pull --ff-only origin dev
git stash pop
```

Resolve conflicts by preserving both new upstream behavior and the still-needed
local fix. For example, when upstream added NVIDIA provider detection at the
same location as the Moonshot patch, both provider branches were retained.

After resolving:

```bash
git add <resolved-files>
git restore --staged \
  routes/memory_routes.py \
  src/agent_loop.py

git diff --check
python -m py_compile \
  routes/memory_routes.py \
  src/agent_loop.py
```

Keep the stash until validation succeeds. Then remove only the named temporary
stash created for this update.

### 5. Rebase or deploy to the server

The server can be rebased using the same selective-stash sequence. Alternatively,
after updating and resolving locally, transfer the reconciled files:

```bash
cd /Users/joshp/Projects/odysseus

rsync -av --relative \
  -e 'ssh -i /Users/joshp/.ssh/id_ed25519' \
  FIXES.md \
  routes/memory_routes.py \
  src/agent_loop.py \
  tests/test_memory_audit_endpoint_resolution.py \
  deploy/update-odysseus-dockge.sh \
  deploy/docker-compose.dockge.yml \
  root@192.168.1.176:/opt/stacks/odysseus/
```

On the server, ensure Git is at the intended upstream commit, resolve any
remaining stash conflict state, and rebuild:

```bash
cd /opt/stacks/odysseus
git status --short --branch
git log -1 --oneline
git diff --check

cp deploy/docker-compose.dockge.yml docker-compose.yml
docker compose up -d --build --remove-orphans
docker compose ps
curl -fsS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7000/login
```

Expected port output:

```text
odysseus   0.0.0.0:7000->7000/tcp
chromadb   8000/tcp
ntfy       80/tcp
```

ChromaDB and ntfy must not show host bindings such as
`127.0.0.1:8100->8000` or `127.0.0.1:8091->80`.

### 6. Verify patched behavior

At minimum, run:

```bash
cd /opt/stacks/odysseus
docker compose logs --tail=150 odysseus
docker compose exec -T odysseus python - <<'PY'
import asyncio
import inspect
import re

from app import _TIMEOUT_EXEMPT_PREFIXES
from mcp_servers import memory_server
from routes.memory_routes import setup_memory_routes
from src import llm_core

assert llm_core._detect_provider(
    "https://api.moonshot.ai/v1/chat/completions"
) == "moonshot"
assert llm_core._moonshot_rejects_custom_temperature(
    "moonshot", "kimi-k2.6"
)
assert not llm_core._moonshot_rejects_custom_temperature(
    "openai", "kimi-k2.6"
)
assert "/api/memory/audit" in _TIMEOUT_EXEMPT_PREFIXES

audit_source = inspect.getsource(setup_memory_routes)
assert 'resolve_endpoint(' in audit_source
assert '"default"' in audit_source
assert "owner=user" in audit_source

async def verify_memory_list():
    result = await memory_server.call_tool("manage_memory", {"action": "list"})
    text = result[0].text
    match = re.match(r"Found (\d+) memory entries:", text)
    assert match
    rows = [line for line in text.splitlines() if line.startswith("- [")]
    assert len(rows) == int(match.group(1))
    assert not text.splitlines()[-1].startswith("... and ")

asyncio.run(verify_memory_list())
print("Local Odysseus fixes verified")
PY
```

Also compare checksums for patched files between local and server when a local
checkout was used as the source of truth.

## Cache and backup policy

Safe rebuildable caches to remove when disk usage grows:

```bash
cd /opt/stacks/odysseus
rm -rf data/.cache/pip data/.npm/_cacache data/.npm/_npx
mkdir -p data/.cache/pip data/.npm/_cacache data/.npm/_npx
docker builder prune -af
df -h /
```

Do not remove these without separately confirming they are disposable:

```text
data/.local
data/local
data/fastembed_cache
data/mcp
data/uploads
Docker named volumes
```

`data/.local` and `data/local` contain installed runtime packages, not merely
download caches. `data/fastembed_cache` contains the local embedding model.

The updater keeps one backup under `/opt/stacks/backups`. Its data backup must
exclude `.cache`, `.local`, `.npm`, `cache`, `fastembed_cache`, and `local`.
A normal backup is a few hundred MB even when the complete `data/` directory is
many GB.

## Known non-blocking logs

- ONNX Runtime may report failed CPU affinity calls inside the LXC.
- ONNX Runtime may attempt CUDA, fail because the mini PC has no CUDA runtime,
  and then use `CPUExecutionProvider`.
- The configured HTTP embedding endpoint may return `401`; local FastEmbed is a
  valid fallback when logs subsequently show the model loaded and the memory
  vector store initialized.
- If the configured HTTP embedding endpoint is offline, local FastEmbed should
  still load from `/app/data/fastembed_cache`. An `[Errno 2] No such file or
  directory: ''` error means the compose file injected an empty
  `FASTEMBED_CACHE_PATH`; restore the canonical Dockge compose and recreate the
  Odysseus service.

## 1. Dynamic tool awareness in compact mode

- File: `src/agent_loop.py`
- Failure:
  When a user starts or tests a chat asking "what tools do you have?", the request is classified as low_signal (no domains match) and skips RAG tool retrieval. This loads only the three base tools (`ask_user`, `manage_memory`, `update_plan`). Because API/compact system prompts did not mention that other tools exist, the model told the user these three were its only capabilities.
- Fix:
  Add a hint block in compact mode (`_assemble_prompt` in `src/agent_loop.py`) when other tools are not shown, explaining that extra tools are automatically enabled on subsequent turns on demand.
- Verify:
  A compact system prompt with fewer tools shows the `(Other tools automatically enabled on subsequent turns...)` hint block in the system prompt.
## 2. CalDAV UNIQUE constraint crash-loop after multi-account migration

- File: `src/caldav_sync.py`
- Commit: `9cfaad2` (June 18, 2026)
- Failure:
  When a CalDAV account is migrated from the legacy single-account prefs key to
  the new `caldav_accounts` list format, `_stable_cal_id()` generates a different
  hash because `account_id` is now non-empty. A new `CalendarCal` row is created
  with the new hash, but existing `CalendarEvent` rows remain under the old
  `calendar_id`. On every subsequent sync, `_find_existing_event()` only queries
  by `(uid, new_calendar_id)`, finds nothing, and tries to bulk-INSERT all events
  as new rows — which hits the `UNIQUE constraint on calendar_events.uid`. The
  entire batch rolls back and the next sync repeats the same failure, causing a
  **sustained 120% CPU spike** from the continuous crash-and-retry cycle.
  Observed on the Nextcloud Contact Birthdays calendar (15 events).
- Fix:
  1. `_find_existing_event()` now accepts an optional `owner` argument and falls
     back to an owner-scoped join across all CalDAV calendars when the
     calendar-scoped lookup finds nothing. This detects events under a stale
     `calendar_id` and returns them for reassignment rather than triggering a
     duplicate INSERT.
  2. The per-calendar `db.commit()` is now wrapped to catch `IntegrityError` and
     fall back to per-row upsert (with global UID lookup) so one conflicting UID
     cannot abort an entire calendar batch.
  3. Hot-fixed the immediate DB state by reassigning the 15 birthday events from
     the old `calendar_id` to the new one and deleting the orphaned `CalendarCal`
     row directly in production.
- Verify:
  After deploy, CalDAV sync should complete without `IntegrityError` and CPU
  should return to idle. Confirm with:
  `docker stats --no-stream | grep odysseus` (expect < 5% at idle) and
  `docker logs odysseus-odysseus-1 --since 5m | grep -i 'integrity\|unique'` (expect no output).
