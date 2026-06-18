---
name: export-cursor-transcripts
description: Copy authorized Cursor agent chat transcripts into a single local folder the user can access. Use when the user asks to export, back up, collect, or package Cursor transcript JSONL or TXT files across local, remote, container, Windows, macOS, or Linux environments.
disable-model-invocation: true
---

# Export Cursor Transcripts

Copy authorized local Cursor transcript files into a single user-accessible export folder.

## Reliability Promise

Make the export work in as many environments as possible by probing likely Cursor locations, using portable tooling first, and falling back gracefully. If transcripts are not accessible from the current environment, create the export folder and manifest anyway, then report what was checked and what the user must provide.

## Safety Rules

- Export only transcripts the current user owns or is explicitly authorized to share.
- Do not collect chats from another user's account, device, filesystem, or cloud storage unless that user runs this skill themselves or gives explicit authorization.
- Do not bypass permissions, scrape private folders, read secrets, or upload hidden project data unrelated to Cursor transcripts.
- If the user asks to gather other people's chats without consent, refuse that part and offer the team workflow below.
- Treat transcript contents as sensitive. Confirm the export folder is intended for the people who will access it.

## Team Workflow

For a team project, each participant should run this skill in their own Cursor session. The result is one local export folder per user/machine.

## Inputs

Optional:
- Export folder. Default: `~/cursor-transcript-export`.
- Transcript source root. Default: auto-detect likely Cursor transcript roots.
- Project slug or workspace name to filter transcript paths
- Export label, such as the project name or teammate name

## Steps

1. Confirm authorization:
   - The user is exporting their own Cursor transcripts, or
   - The user has explicit permission for the transcript files being exported.

2. Decide the export folder:
   - Use the folder provided by the user, if any.
   - Otherwise use `~/cursor-transcript-export`.

3. Find all Cursor transcript files:
   - Search recursively under all likely Cursor roots listed in "Source Discovery".
   - Filter to the current project slug if the request is project-specific.
   - Include current `.jsonl` transcript files and legacy `.txt` transcript files.
   - Include files only when their path contains `agent-transcripts`.

4. Create a timestamped folder inside the export folder.

5. Copy transcript files into staging while preserving enough path context to identify the project/session.

6. Write `manifest.json` with:
   - export timestamp
   - source machine/user label if provided
   - workspace path
   - transcript source roots checked
   - count of files exported
   - relative filenames included
   - skipped roots and errors, if any

7. Optionally create a `.zip` archive next to the timestamped folder if the user asks for a compressed package.

8. Verify the export:
   - Confirm the timestamped export folder exists.
   - Confirm `manifest.json` exists.
   - Confirm the copied transcript count matches the manifest count.

9. Report:
   - number of transcript files packaged
   - export folder path
   - archive path, if created
   - whether verification succeeded

## Source Discovery

Check all paths that exist and skip paths that do not. Do not fail the whole export because one candidate path is missing or unreadable.

Likely transcript roots:

- `~/.cursor/projects`
- `$CURSOR_AGENT_HOME/projects` if `CURSOR_AGENT_HOME` is set
- `%USERPROFILE%/.cursor/projects` on Windows
- `$HOME/.cursor/projects` on macOS/Linux
- The attached/current project transcript root if Cursor exposes one in the session context

If no transcript files are found:

1. Create the timestamped export folder anyway.
2. Write `manifest.json` with `transcript_count: 0`, checked roots, and errors.
3. Ask the user for the folder that contains their Cursor `agent-transcripts` files.

For remote shells, WSL, SSH, containers, or Codespaces:

- First search the environment where the agent is running.
- If that environment cannot see the host Cursor data folder, ask the user for a mounted host path or run the skill from the host Cursor session.
- Do not claim success unless files were actually copied.

## Tool Fallbacks

Prefer this order:

1. Python standard library script below.
2. If Python is unavailable, use Node.js with `fs`, `path`, and `os`.
3. If neither Python nor Node.js is available, use the current shell's native copy commands.
4. If no executable method is available, provide exact manual copy instructions and still explain where the export folder should be created.

## Cross-Platform Packaging Pattern

Use Python for OS-independent export. It works on Windows, macOS, and Linux when Python is available.

```python
from datetime import datetime, timezone
from pathlib import Path
import json
import os
import shutil
import socket

home = Path.home()
workspace = Path.cwd()
export_root = home / "cursor-transcript-export"
timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
stage = export_root / timestamp

stage.mkdir(parents=True, exist_ok=True)

candidate_roots = []

def add_root(value):
    if value:
        path = Path(value).expanduser()
        if path.name != "projects":
            path = path / "projects"
        if path not in candidate_roots:
            candidate_roots.append(path)

add_root(home / ".cursor" / "projects")
add_root(os.environ.get("CURSOR_AGENT_HOME"))
add_root(os.environ.get("USERPROFILE") and Path(os.environ["USERPROFILE"]) / ".cursor" / "projects")
add_root(os.environ.get("HOME") and Path(os.environ["HOME"]) / ".cursor" / "projects")

files = []
checked_roots = []
errors = []

for root in candidate_roots:
    checked_roots.append(str(root))
    if not root.exists():
        continue
    try:
        files.extend(
            path
            for path in root.rglob("*")
            if path.is_file()
            and path.suffix in {".jsonl", ".txt"}
            and "agent-transcripts" in path.parts
        )
    except OSError as error:
        errors.append({"root": str(root), "error": str(error)})

files = sorted(set(files))

copied = []
for source in files:
    try:
        relative = source.relative_to(home)
    except ValueError:
        relative = Path("external") / source.drive.replace(":", "") / source.relative_to(source.anchor)
    destination = stage / relative
    destination.parent.mkdir(parents=True, exist_ok=True)
    try:
        shutil.copy2(source, destination)
        copied.append(str(relative))
    except OSError as error:
        errors.append({"file": str(source), "error": str(error)})

manifest = {
    "exported_at": datetime.now(timezone.utc).isoformat(),
    "workspace": str(workspace),
    "source_roots_checked": checked_roots,
    "export_root": str(export_root),
    "stage": str(stage),
    "machine": socket.gethostname(),
    "user": os.environ.get("USER") or os.environ.get("USERNAME"),
    "transcript_count": len(copied),
    "files": copied,
    "errors": errors,
}

(stage / "manifest.json").write_text(
    json.dumps(manifest, indent=2),
    encoding="utf-8",
)

print(stage)
```

If Python is not available, use the current shell's native file-copy commands while preserving the same behavior and manifest fields.

## Output Style

Keep the final response brief:

```markdown
Export complete.

- Packaged: [N] transcript files
- Folder: `[path]`
- Archive: `[path or not created]`
- Verification: [passed/failed]
```
