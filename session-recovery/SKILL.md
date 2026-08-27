---
name: session-recovery
description: Recover Kiro chat sessions that disappeared after workspace folder changes. Use when sessions are missing from history, after adding/removing folders from a multi-root workspace, or when the user reports lost chat history.
---

## Overview

Kiro stores sessions at `$HOME/.kiro/sessions/<hash>/sess_<uuid>/` where the hash is derived from the sorted workspace folder paths. Adding or removing any folder from a multi-root workspace changes the hash, silently orphaning all previous sessions. The data remains on disk but becomes invisible to the UI.

This skill recovers those orphaned sessions by copying them into the correct current hash directory.

## When to use

- User reports missing or disappeared chat sessions
- User recently added or removed folders from a workspace
- User opened a workspace that previously had a different folder composition
- Session history panel is empty but the user had prior conversations

## Recovery procedure

### Step 1: Find workspace files

Locate the user's `.code-workspace` files. Common locations:

```bash
find "$HOME/git" -maxdepth 2 -name "*.code-workspace" 2>/dev/null
```

Ask the user which workspaces are affected if multiple exist.

### Step 2: Run the recovery script

Write the script below to `/tmp/recover-kiro-sessions.py`, substituting the workspace file paths into `WORKSPACE_FILES`. Then run it.

```python
#!/usr/bin/env python3
"""
Recover orphaned Kiro sessions lost when workspace folder composition changed.

For each .code-workspace file:
- Computes the current workspace hash
- Scans all session hash directories for sessions whose workspacePaths
  are a subset of the current workspace's folder list
- Copies orphaned sessions into the correct current hash directory
- Updates session.json with current workspace paths

Originals are never modified or deleted. Safe to run multiple times.
"""
import hashlib
import json
import os
import platform
import shutil
from pathlib import Path

SESSIONS_BASE = Path.home() / ".kiro" / "sessions"

# SUBSTITUTE actual .code-workspace file paths here
WORKSPACE_FILES = [
    # "$HOME/projects/my-workspace.code-workspace",
]


def compute_workspace_hash(paths):
    """Compute the hash Kiro uses for session storage directory names."""
    def normalize(p):
        p = os.path.abspath(p).replace("\\", "/")
        if platform.system() == "Windows":
            p = p.lower()
        return p
    normalized = sorted(normalize(p) for p in paths)
    return hashlib.sha256("\x00".join(normalized).encode()).hexdigest()[:16]


def resolve_workspace_paths(ws_file):
    """Read a .code-workspace file and resolve relative paths to absolute."""
    with open(ws_file) as f:
        data = json.load(f)
    folders = data.get("folders", [])
    ws_dir = os.path.dirname(os.path.abspath(ws_file))
    paths = []
    for folder in folders:
        p = folder.get("path", "")
        if not os.path.isabs(p):
            p = os.path.normpath(os.path.join(ws_dir, p))
        paths.append(p)
    return paths


def find_orphaned_sessions(target_paths, target_hash):
    """Find sessions whose workspacePaths are a subset of target_paths."""
    target_set = set(target_paths)
    results = []

    for ws_hash_dir in SESSIONS_BASE.iterdir():
        if not ws_hash_dir.is_dir():
            continue
        if ws_hash_dir.name in ("_global", "cli"):
            continue
        if ws_hash_dir.name == target_hash:
            continue

        for session_dir in ws_hash_dir.iterdir():
            if not session_dir.is_dir():
                continue
            session_json = session_dir / "session.json"
            if not session_json.exists():
                continue
            with open(session_json) as f:
                data = json.load(f)
            session_paths = set(data.get("workspacePaths", []))
            if session_paths and session_paths.issubset(target_set):
                results.append((ws_hash_dir.name, session_dir, data))

    return results


def migrate_sessions(target_paths, target_hash, sessions):
    """Copy orphaned sessions to the target hash directory."""
    target_dir = SESSIONS_BASE / target_hash
    copied = 0
    skipped = 0

    for src_hash, src_session_dir, data in sessions:
        session_name = src_session_dir.name
        if not session_name.startswith("sess_"):
            session_name = f"sess_{session_name}"

        dest = target_dir / session_name
        if dest.exists():
            skipped += 1
            continue

        target_dir.mkdir(parents=True, exist_ok=True)
        shutil.copytree(src_session_dir, dest)

        dest_json = dest / "session.json"
        with open(dest_json) as f:
            session_data = json.load(f)
        session_data["id"] = session_name
        session_data["workspacePaths"] = target_paths
        session_data.setdefault("schemaVersion", "1.0.0")
        session_data.setdefault("dataModelVersion", 1)
        session_data.setdefault("semanticReviewEnabled", True)
        session_data.setdefault("ftaEnabled", False)
        with open(dest_json, "w") as f:
            json.dump(session_data, f, indent=2)

        title = data.get("title", "(untitled)")[:60]
        print(f"    Recovered: {session_name[:40]}  \"{title}\"")
        copied += 1

    return copied, skipped


def main():
    if not WORKSPACE_FILES:
        print("ERROR: WORKSPACE_FILES is empty. Fill in your .code-workspace file paths.")
        return

    total_recovered = 0
    total_skipped = 0

    for ws_file in WORKSPACE_FILES:
        if not os.path.exists(ws_file):
            print(f"\nWARNING: {ws_file} not found, skipping.")
            continue

        ws_name = os.path.basename(ws_file)
        paths = resolve_workspace_paths(ws_file)
        target_hash = compute_workspace_hash(paths)

        print(f"\n{'='*60}")
        print(f"Workspace: {ws_name}")
        print(f"  Folders: {len(paths)}")
        print(f"  Hash: {target_hash}")
        print(f"  Scanning for orphaned sessions...")

        sessions = find_orphaned_sessions(paths, target_hash)

        if not sessions:
            print(f"  No orphaned sessions found.")
            continue

        print(f"  Found {len(sessions)} orphaned sessions. Recovering...")
        copied, skipped = migrate_sessions(paths, target_hash, sessions)
        total_recovered += copied
        total_skipped += skipped

        if skipped:
            print(f"  Skipped {skipped} (already exist in target)")

    print(f"\n{'='*60}")
    print(f"DONE: Recovered {total_recovered} sessions, skipped {total_skipped} duplicates.")
    print(f"\nFully quit Kiro (all windows) and reopen your workspaces for sessions to appear.")


if __name__ == "__main__":
    main()
```

### Step 3: Clean up and inform user

```bash
rm /tmp/recover-kiro-sessions.py
```

Tell the user to **fully quit Kiro** (all windows, not just reload) and reopen their workspace for the recovered sessions to appear in the history panel.

## Technical details

- Hash formula: `SHA-256(sorted_absolute_paths.join("\0"))[:16]`
- On Linux/macOS paths are used as-is; on Windows they're lowercased with forward slashes
- The script finds sessions whose `workspacePaths` are a subset of the current workspace (handles workspace growth)
- Originals are never modified or deleted
- Idempotent — safe to run repeatedly
- Sessions already present in the target directory are skipped

## Known issue

This is tracked upstream: [kirodotdev/Kiro#9650](https://github.com/kirodotdev/Kiro/issues/9650), [#10534](https://github.com/kirodotdev/Kiro/issues/10534), [#10545](https://github.com/kirodotdev/Kiro/issues/10545)
