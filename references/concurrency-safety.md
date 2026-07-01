# Concurrency Safety

## The Problem

Multi-agent pipelines have sub-agents writing to shared files simultaneously:
- Multiple roles in the same wave write to `evidence_store.json`
- Multiple roles write their sidecars to the same `outputs/` directory
- Repair sub-agents modify files that other sub-agents are also reading

Without protection: **last writer wins**, data loss is silent and unrecoverable.

## Solution: Two-Pronged Approach

### 1. Sequential Dispatch (prevent most conflicts)

Instead of dispatching all roles in a wave at once, dispatch ONE at a time:

```
prepare(first call) → manifest for role_A only, has_more=True
[Coordinator spawns role_A, waits for completion]
prepare(second call) → manifest for role_B only, has_more=True
[Coordinator spawns role_B, waits for completion]
...
prepare(last call) → manifest for role_Z, has_more=False
[Kernel advances to collect phase]
```

This eliminates 90% of write conflicts. See [dispatch-protocol.md](dispatch-protocol.md) for the full sequential dispatch protocol.

### 2. File Locking (protect remaining shared writes)

For the cases where sequential dispatch isn't enough (e.g., repair sub-agents modifying `evidence_store.json` while other files reference it), use POSIX file locks.

## File Lock Module

```python
"""
Provides two functions:
- locked_read_modify_write: flock-based atomic read-modify-write for JSON files
- atomic_write: write-to-temp-then-rename for text files
"""
import fcntl, json, os
from pathlib import Path

def locked_read_modify_write(path: Path, modify_fn) -> dict:
    """Acquire exclusive lock → read JSON → modify → atomic write → release lock.

    Flow:
    1. Create lock file <path>.lock
    2. fcntl.flock(LOCK_EX) — blocks until lock acquired
    3. Read current JSON (or {} if missing/corrupt)
    4. Call modify_fn(data) → new_data
    5. Write to <path>.tmp → os.replace() to <path> (atomic on same directory)
    6. Release lock (guaranteed by try/finally)

    Usage in sub-agent system_prompt:
    "When modifying evidence_store.json or any shared sidecar, you MUST use
     locked_read_modify_write from scripts.file_lock module."
    """
    lock_path = path.with_suffix(path.suffix + ".lock")
    lock_path.parent.mkdir(parents=True, exist_ok=True)

    lock_fd = os.open(str(lock_path), os.O_CREAT | os.O_RDWR)
    try:
        fcntl.flock(lock_fd, fcntl.LOCK_EX)

        # Read
        if path.exists():
            try:
                data = json.loads(path.read_text(encoding="utf-8"))
            except (json.JSONDecodeError, ValueError):
                data = {}
        else:
            data = {}

        # Modify
        new_data = modify_fn(data)

        # Atomic write
        tmp_path = path.with_suffix(path.suffix + ".tmp")
        tmp_path.parent.mkdir(parents=True, exist_ok=True)
        tmp_path.write_text(
            json.dumps(new_data, ensure_ascii=False, indent=2),
            encoding="utf-8",
        )
        os.replace(str(tmp_path), str(path))

        return new_data
    finally:
        fcntl.flock(lock_fd, fcntl.LOCK_UN)
        os.close(lock_fd)


def atomic_write(path: Path, content: str) -> None:
    """Write to temp file → os.replace() to target path.

    Prevents other processes from reading a half-written file.
    The temp file MUST be in the same directory as the target
    (os.replace is only atomic on the same filesystem).
    """
    tmp_path = path.with_suffix(path.suffix + ".tmp")
    tmp_path.parent.mkdir(parents=True, exist_ok=True)
    tmp_path.write_text(content, encoding="utf-8")
    os.replace(str(tmp_path), str(path))
```

## When to Use Which

| Scenario | Mechanism | Why |
|----------|-----------|-----|
| Wave dispatch (roles writing their own output files) | Sequential dispatch | Each role writes to its OWN files (role.md, role-data.json, role-meta.json) — no conflict if sequential |
| Repair sub-agent modifying `evidence_store.json` | `locked_read_modify_write` | Shared file, multiple potential writers |
| Repair sub-agent modifying shared sidecars | `locked_read_modify_write` | Shared file, could be read by other processes |
| JSON auto-repair writing back fixed JSON | `atomic_write` | Single writer, but need to prevent partial reads |
| Phase state JSON writes | `atomic_write` | Kernel-only, single writer |

## The 4-Layer Defense (Complete Picture)

Concurrency safety is Layer 1 of the broader output collection defense:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Soft Constraint                                    │
│ Dispatch instruction mandates:                              │
│   - 3-file output (md + data + meta)                        │
│   - Sequential only (no parallel dispatch)                  │
│   - File lock for shared file modifications                 │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Hard Constraint                                    │
│ _role_outputs_complete() checks:                            │
│   - .md exists and size > 100 bytes                         │
│   - -data.json exists and valid JSON                        │
│   - -meta.json exists and valid JSON                        │
│   - _file_stable(path, interval=8) — size unchanged 8s      │
├─────────────────────────────────────────────────────────────┤
│ Layer 2.5: Retry Buffer                                     │
│ Configurable retry count × interval = total timeout         │
│ Progress detection: log which roles are complete each cycle │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Semi-Auto Recovery                                 │
│ Collect phase detects incomplete roles →                    │
│   returns needs_dispatch → Coordinator re-dispatches        │
│   Same role, same manifest, new sub-agent attempt           │
└─────────────────────────────────────────────────────────────┘
```

## Platform Considerations

- **POSIX only**: `fcntl.flock` works on Linux and macOS. Not available on Windows.
- **Lock file persistence**: Lock files (`<path>.lock`) are not deleted after use — they're reused on next call. This is intentional.
- **Lock scope**: `LOCK_EX` is process-level. If a sub-agent spawns child processes, they don't inherit the lock.
- **Deadlock prevention**: `try/finally` guarantees lock release even on exceptions.

## JSON Self-Repair (Defensive Parsing)

Sub-agents sometimes produce malformed JSON. Before failing validation, attempt auto-repair:

```python
def safe_load_json_with_repair(path: Path) -> dict | None:
    """Try loading JSON; on failure, fix common issues and retry."""
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except json.JSONDecodeError:
        pass

    # Common fix: unescaped quotes inside JSON string values
    text = path.read_text(encoding="utf-8")
    fixed = _fix_unescaped_quotes(text)
    try:
        data = json.loads(fixed)
        atomic_write(path, json.dumps(data, ensure_ascii=False, indent=2) + "\n")
        return data
    except json.JSONDecodeError:
        return None
```

This prevents a single malformed sidecar from failing the entire validation stage. Combined with schema_version alias mapping, it handles the most common sub-agent output quirks.
