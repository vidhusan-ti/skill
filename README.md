# Cursor Transcript Export Skill

Project skill for exporting local Cursor chat transcripts into one accessible folder.

## Use

```text
/export-cursor-transcripts
```

## What It Does

- Finds Cursor `agent-transcripts` files across common local, remote, WSL, container, Windows, macOS, and Linux setups.
- Copies `.jsonl` and legacy `.txt` transcripts into `~/cursor-transcript-export/<timestamp>/`.
- Writes a `manifest.json` with copied files, checked paths, and errors.
- Falls back from Python to Node, shell commands, or manual instructions when needed.

The skill exports only transcripts the current user owns or is authorized to share.
