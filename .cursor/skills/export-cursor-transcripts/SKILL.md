---
name: export-cursor-transcripts
description: Export authorized Cursor agent chat transcripts to a Google Drive folder. Use when the user provides a Google Drive folder link and asks to back up, share, or collect Cursor transcript JSONL files from consenting project members.
disable-model-invocation: true
---

# Export Cursor Transcripts

Package authorized local Cursor transcript files and upload them to a Google Drive folder provided by the user.

## Safety Rules

- Export only transcripts the current user owns or is explicitly authorized to share.
- Do not collect chats from another user's account, device, filesystem, or cloud storage unless that user runs this skill themselves or gives explicit authorization.
- Do not bypass permissions, scrape private folders, read secrets, or upload hidden project data unrelated to Cursor transcripts.
- If the user asks to gather other people's chats without consent, refuse that part and offer the team workflow below.
- Treat transcript contents as sensitive. Confirm the target Drive folder is intended for the project team before uploading.

## Team Workflow

For a team project, each participant should run this skill in their own Cursor session with the same Google Drive folder link. The result is one upload package per user/machine.

## Inputs

Required:
- Google Drive folder link, for example `https://drive.google.com/drive/folders/<folder-id>`

Optional:
- Transcript source glob. Default on Windows: `$env:USERPROFILE/.cursor/projects/*/agent-transcripts/**/*.jsonl`
- Project slug or workspace name to filter transcript paths
- Export label, such as the project name or teammate name

## Upload Methods

Prefer one of these authorized upload methods:

- `rclone` with a configured Google Drive remote, usually named `gdrive`
- Google Drive for desktop mounted as a local folder

If neither upload method is available, create the export archive and ask the user which authorized Drive method to use.

## Rclone Setup

When the user chooses `rclone`, first check whether it is available:

```powershell
rclone version
```

If `rclone` is not installed on Windows and `winget` is available, install it:

```powershell
winget install --id Rclone.Rclone --exact --accept-source-agreements --accept-package-agreements
```

If the current shell cannot find `rclone` immediately after install, use the Winget package path directly:

```powershell
$rclone = "$env:LOCALAPPDATA\Microsoft\WinGet\Packages\Rclone.Rclone_Microsoft.Winget.Source_8wekyb3d8bbwe\rclone-v1.74.3-windows-amd64\rclone.exe"
& $rclone version
```

Create or update the `gdrive` remote for the target folder ID:

```powershell
& $rclone config create gdrive drive scope drive root_folder_id "<folder-id>" --auto-confirm
```

This may open a browser for Google login. Wait for the authorization to complete before uploading.

## Steps

1. Confirm authorization:
   - The user is exporting their own Cursor transcripts, or
   - The user has explicit permission for the transcript files being exported.

2. Parse the Google Drive folder ID from the link:
   - Folder URL pattern: `/folders/<folder-id>`
   - Query URL pattern: `?id=<folder-id>`
   - If no folder ID is visible, ask the user for a valid Drive folder link.

3. Find transcript files:
   - Use the default local transcript glob unless the user gives a source.
   - Filter to the current project slug if the request is project-specific.
   - Include only `.jsonl` transcript files.

4. Create a timestamped staging directory under `data/cursor-transcript-export/`.

5. Copy transcript files into staging while preserving enough path context to identify the project/session.

6. Write `manifest.json` with:
   - export timestamp
   - source machine/user label if provided
   - workspace path
   - transcript glob used
   - count of files exported
   - relative filenames included
   - target Drive folder ID

7. Compress the staging directory to a `.zip` archive.

8. Upload the `.zip` archive:
   - With `rclone`:
     ```powershell
     rclone copy "<archive.zip>" "gdrive:"
     ```
   - With Google Drive for desktop:
     ```powershell
     Copy-Item "<archive.zip>" "<local-google-drive-folder>"
     ```

9. Verify upload:
   - For `rclone`, run:
     ```powershell
     rclone lsf "gdrive:"
     ```
   - For Drive for desktop, verify the archive exists at the mounted folder path.

10. Report:
   - number of transcript files packaged
   - archive path
   - upload method used
   - whether upload verification succeeded

## PowerShell Packaging Pattern

Use this pattern when generating the archive. Set `$driveFolderLink` from the user's Google Drive folder link before running it.

```powershell
$driveFolderLink = "<google-drive-folder-link>"
$folderId = if ($driveFolderLink -match '/folders/([^/?#]+)') {
  $Matches[1]
} elseif ($driveFolderLink -match '[?&]id=([^&#]+)') {
  $Matches[1]
} else {
  throw "Could not parse a Google Drive folder ID from the provided link."
}

$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$exportRoot = "data/cursor-transcript-export"
$stage = Join-Path $exportRoot $timestamp
$archive = Join-Path $exportRoot "cursor-transcripts-$timestamp.zip"
New-Item -ItemType Directory -Force -Path $stage | Out-Null

$files = Get-ChildItem -Path "$env:USERPROFILE\.cursor\projects" -Recurse -Filter "*.jsonl" |
  Where-Object { $_.FullName -like "*agent-transcripts*" }

foreach ($file in $files) {
  $relative = $file.FullName.Substring($env:USERPROFILE.Length).TrimStart('\')
  $destination = Join-Path $stage $relative
  New-Item -ItemType Directory -Force -Path (Split-Path $destination -Parent) | Out-Null
  Copy-Item $file.FullName $destination
}

$manifest = [ordered]@{
  exported_at = (Get-Date).ToString("o")
  workspace = (Get-Location).Path
  target_drive_folder_id = $folderId
  transcript_count = $files.Count
  files = $files.FullName
}
$manifest | ConvertTo-Json -Depth 5 | Set-Content -Encoding UTF8 (Join-Path $stage "manifest.json")

Compress-Archive -Path (Join-Path $stage "*") -DestinationPath $archive -Force
Write-Output $archive
```

## Output Style

Keep the final response brief:

```markdown
Export complete.

- Packaged: [N] transcript files
- Archive: `[path]`
- Uploaded to: Google Drive folder `[folder-id]`
- Verification: [passed/failed/not configured]
```
