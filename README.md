# Cursor Transcript Export Skill

This repository contains a Cursor project skill that packages local Cursor agent chat transcripts and uploads them to a Google Drive folder.

Skill path:

```text
.cursor/skills/export-cursor-transcripts/SKILL.md
```

## What The Skill Does

The `export-cursor-transcripts` skill:

- Finds local Cursor transcript `.jsonl` files.
- Packages them into a timestamped zip archive.
- Writes a `manifest.json` with export metadata.
- Uploads the zip archive to a Google Drive folder.
- Verifies the upload by listing the target Drive folder.

The skill is designed for team projects where each team member can run the same skill with the same Google Drive folder link.

## High-Level Design

```text
User prompt with Google Drive folder link
        |
        v
Cursor skill parses the folder ID
        |
        v
Find local Cursor transcript JSONL files
        |
        v
Copy transcripts into staging folder
        |
        v
Create manifest.json
        |
        v
Compress staging folder into zip archive
        |
        v
Upload archive using rclone or Google Drive Desktop
        |
        v
Verify upload and report result
```

## How To Install

Clone this repo into a project or copy the skill directory into an existing Cursor project:

```text
.cursor/skills/export-cursor-transcripts/
```

Cursor discovers project skills from `.cursor/skills/<skill-name>/SKILL.md`.

## How To Use

In Cursor chat, run:

```text
/export-cursor-transcripts https://drive.google.com/drive/folders/<folder-id>
```

Example:

```text
/export-cursor-transcripts https://drive.google.com/drive/folders/1jr_I3R7KIsvdo9Wjyp3gVcSi66ZmxLgX
```

The folder link should be a Google Drive folder URL, not a file URL.

## Prompt Format

Use a direct prompt with the skill name and Drive folder link:

```text
/export-cursor-transcripts <google-drive-folder-link>
```

Optional extra context can be added after the link:

```text
/export-cursor-transcripts <google-drive-folder-link> export only this project and label it as Vidhusan laptop
```

Good prompts:

```text
/export-cursor-transcripts https://drive.google.com/drive/folders/<folder-id>
```

```text
/export-cursor-transcripts https://drive.google.com/drive/folders/<folder-id> export only the current project
```

```text
/export-cursor-transcripts https://drive.google.com/drive/folders/<folder-id> use rclone
```

## Rclone Sign-In

The preferred automated upload method is `rclone`.

First, the skill checks whether `rclone` is installed:

```powershell
rclone version
```

If it is missing on Windows, install it with `winget`:

```powershell
winget install --id Rclone.Rclone --exact --accept-source-agreements --accept-package-agreements
```

After install, the current shell may not immediately find `rclone`. In that case, the skill can use the Winget package path directly:

```powershell
$rclone = "$env:LOCALAPPDATA\Microsoft\WinGet\Packages\Rclone.Rclone_Microsoft.Winget.Source_8wekyb3d8bbwe\rclone-v1.74.3-windows-amd64\rclone.exe"
& $rclone version
```

Then create the Google Drive remote:

```powershell
& $rclone config create gdrive drive scope drive root_folder_id "<folder-id>" --auto-confirm
```

This opens a browser. Sign in to Google and approve access. When authorization completes, `rclone` stores the remote as `gdrive`.

## Upload And Verify

After packaging the transcripts, upload the archive:

```powershell
rclone copy "<archive.zip>" "gdrive:"
```

Verify the uploaded file:

```powershell
rclone lsf "gdrive:"
```

If the zip filename appears in the listing, the upload succeeded.

## Output

The skill reports:

```text
Export complete.

- Packaged: <N> transcript files
- Archive: <local archive path>
- Uploaded to: Google Drive folder <folder-id>
- Verification: passed
```

## Notes

- Transcript files are usually stored under the local Cursor projects directory.
- The generated archive is written under `data/cursor-transcript-export/`.
- Each team member should run the skill from their own Cursor environment if the goal is to collect multiple people's transcripts.
