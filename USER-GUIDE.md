# FCP Backup Manager User Guide

FCP Backup Manager is a menu bar app for backing up Final Cut Pro backup bundles to one or more destinations such as local folders, external drives, NAS shares, SMB shares, or other mounted volumes.

This guide reflects `v0.8.0` / `build 1`.

## Overview

FCP Backup Manager:

- scans your Final Cut backup source folder
- compresses each new `.fcpbundle` into a zip archive
- writes each zip into a per-machine destination folder
- tracks completed copies in a manifest
- logs every run locally and at each attempted destination

The app lives in the macOS menu bar and does not appear in the Dock.

## Where to Put Screenshots

Suggested screenshots for this guide:

1. Menu bar popup with destination status and Back Up Now
2. Settings window showing the sidebar navigation and one of the main configuration areas

Suggested placement:

- Add a menu bar screenshot below **Daily Use**
- Add a Settings screenshot below **Initial Setup**

## Initial Setup

### 1. Launch the App

After launch, look for the FCP Backup Manager icon in the menu bar.

The menu shows:

- last backup time
- next scheduled run
- Back Up Now
- destination status
- pending config alerts
- links to Settings and logs
- current app version and build

### 2. Open Settings

Use either:

- **Settings…** from the menu bar menu
- the standard macOS Settings shortcut `Cmd+,`

The Settings window uses a sidebar with:

- **Source** for source path, bundle filters, and history
- **Destinations** for destinations, testing, manual backup, and retention
- **Schedule** for backup interval, preferred time, quiet hours, and launch at login
- **Import / Export** for setup-file import and config export

### 3. Add Destinations

A destination is any folder where backup zips should be written.

Examples:

- `/Users/you/Backups/FCP`
- `/Volumes/MySSD/FCPBackups`
- `/Volumes/NAS/Projects/FCP/ProjectBackups`

Steps:

1. Open **Settings**.
2. In **Destinations**, click **Add Destination**.
3. Choose a folder.
4. Give it a clear alias, such as `Primary NAS` or `Post`.

Each destination row shows:

- alias
- path
- mounted/accessibility status
- enable toggle
- remove button for non-MDM destinations
- folder reveal button

### 4. Test Destinations

Click **Test Destinations** in Settings before your first backup.

The test checks:

- source accessibility
- destination accessibility
- ability to create the app’s hostname folder
- ability to write a small test file
- reported free space

Results appear in an alert, and a detailed record is written to the app diagnostic log.

### 5. Verify the Version

The current app version and build appear in:

- the menu bar popup
- the lower part of the Settings sidebar

## Daily Use

### Manual Backup Workflow

Use this workflow when you want to force a backup immediately.

1. Open the menu bar app or Settings.
2. Click **Back Up Now**.
3. The app scans the source, zips new bundles, copies them to each enabled and mounted destination, and updates logs.

What you should expect:

- if new bundles exist, the app reports how many were copied
- if everything is already up to date, the app stays quiet
- if an error occurs, the app shows an alert and logs details

### Scheduled Backup Workflow

Scheduled backups run automatically while the app is running. The schedule starts at app launch — no need to open the menu bar first.

You can choose an interval:

- Every Hour
- Every Day
- Every Week

If you want automatic scheduled backups after login, enable **Launch at Login** in Settings.

#### Preferred Time

For daily or weekly schedules, you can optionally set a specific time of day for backups to run (e.g., 6:00 PM). Enable **Run at a specific time** in Settings and pick the time. This replaces the rolling interval with a fixed daily or weekly time.

#### Quiet Hours

Enable **Quiet hours** to block scheduled backups during a time window (e.g., 9:00 AM to 5:00 PM). This is useful when you don't want backup I/O during editing sessions. If a backup would fire during quiet hours, it defers to when the window ends.

Overnight windows are supported (e.g., 10:00 PM to 6:00 AM).

#### Overdue Catch-Up

If a scheduled backup is missed (the app was quit, or the Mac was asleep), the app runs it shortly after launch or wake instead of waiting for the next full interval.

### Example Workflow: One Local Folder and Two Network Shares

Example setup:

- `D` → `/Users/you/FCPBackups`
- `NAS` → `/Volumes/NAS/Projects/FCP/ProjectBackups`
- `Post` → `/Volumes/finalcutpro/version/FCP/ProjectBackups`

Typical flow:

1. Add all three destinations.
2. Run **Test Destinations**.
3. Confirm all destinations show accessible and writable.
4. Run **Back Up Now**.
5. Open **View Logs…** if you want to confirm each destination was attempted.

### Example Workflow: Skip Matching Bundle Names

Use **Project Filters** in the Source area when you want to skip specific backup bundles by keyword.

Examples:

- `VIP`
- `temp`
- `archive`

Behavior:

- matching is case-insensitive
- the app checks the `.fcpbundle` name
- if a bundle name contains one of the keywords, that bundle is skipped before zip and copy

The app diagnostic log records the exact keyword and bundle name when a filter matches.

## How Backups Are Stored

Backups are written under:

```text
destination/FCPBackupManager/<hostname>/
```

Example:

```text
/Volumes/NAS/Projects/FCP/ProjectBackups/
└── FCPBackupManager/
    └── edit-suite-01/
        ├── ProjectA_20260323_0916_PDT.zip
        ├── ProjectB_20260323_1010_PDT.zip
        ├── backup.log
        └── edit-suite-01-2026-03-23_0916.log
```

## Dedup and Manifest Behavior

FCP Backup Manager tracks every completed copy in:

```text
~/Library/Application Support/FCPBackupManager/manifest.json
```

The dedup key is:

```text
(project name) + (bundle name) + (destination alias)
```

This means:

- the same bundle is copied once per destination
- adding a new destination causes previously known bundles to be copied there
- renaming a destination alias creates a new dedup identity

### Trust-but-Verify

If the manifest says a bundle was already backed up, the engine still checks whether the destination zip is actually present. If the file is missing, the stale entry is removed and the bundle is copied again.

### Reset Manifest

Use **Reset Manifest…** in Settings > History if you want the app to treat all bundles as new again.

Use this when:

- destination zips were deleted manually
- the manifest seems out of sync
- you want a full re-copy test

## Logs and Troubleshooting Files

FCP Backup Manager writes three useful log outputs.

### 1. Destination Logs

Written to each attempted destination:

```text
destination/FCPBackupManager/<hostname>/
```

Files:

- `backup.log` — latest destination log
- `<hostname>-YYYY-MM-DD_HHmm.log` — timestamped per-run log

### 2. Session Logs

Written locally to:

```text
~/Library/Application Support/FCPBackupManager/Logs/
```

These summarize the full run across all attempted destinations.

### 3. App Diagnostic Log

Written locally to:

```text
~/Library/Application Support/FCPBackupManager/Logs/app-diagnostic.log
```

The app also keeps daily rotated diagnostic files beside it:

```text
~/Library/Application Support/FCPBackupManager/Logs/app-diagnostic-YYYY-MM-DD.log
```

This log is best for troubleshooting:

- destination filtering
- bundle-filter matches and skipped `.fcpbundle` names
- test destination output
- free-space warnings
- destination log success/failure
- copy fallback behavior

### View Logs

You can open the local log folder from:

- the menu bar menu via **View Backup Logs…**
- Settings > History via **View Logs…**

## Settings Reference

### Source

The source path defaults to:

```text
~/Movies/Final Cut Backups.localized
```

You can change it in Settings if your Final Cut backups live elsewhere.

### Project Filters

Project Filters are optional keyword filters for `.fcpbundle` names.

Examples:

- `VIP`
- `temp`
- `review`

If a discovered `.fcpbundle` name contains one of these keywords, that bundle is skipped for all destinations in the run.

### Schedule

Available intervals:

| Interval | Meaning |
|----------|---------|
| Every Hour | Run once per hour while the app is running |
| Every Day | Run once per day while the app is running |
| Every Week | Run once per week while the app is running |

Optional schedule settings:

| Setting | Effect |
|---------|--------|
| Run at a specific time | For daily/weekly, fire at a fixed time of day instead of a rolling interval |
| Quiet hours | Suppress scheduled backups during a time window (e.g., 9 AM to 5 PM) |

### Launch at Login

When enabled, the app registers itself to launch automatically when the user logs in.

### Retention Policy

Retention runs after backup completion and can remove older zips using either or both of these rules:

- keep last N backups per project
- delete backups older than N days

Retention is disabled by default.

### History

The History section shows:

- recent run summaries
- manifest entry count
- View Logs
- Reset Manifest

## Import / Export Config

### Import Config

Import a JSON setup file from Settings.

Behavior:

- schedule, source path, launch-at-login, project filters, and retention values are applied automatically unless MDM-managed
- destinations are queued for user acceptance

### Export Current Config

Export the current settings to a JSON file for reuse on another machine.

MDM-managed destinations are excluded from export.

## Setup JSON Workflow

Setup JSON files live at:

```text
~/Library/Application Support/FCPBackupManager/Setup/
```

Expected filename pattern:

```text
FCPBackupManagerSetup-YYYYMMDD.json
```

Example:

```json
{
  "configDate": "20260403",
  "schedule": "1d",
  "sourcePath": "~/Movies/Final Cut Backups.localized",
  "launchAtLogin": true,
  "projectFilters": ["VIP", "archive"],
  "retentionEnabled": true,
  "retentionKeepLastN": 10,
  "retentionDeleteOlderThanDays": 90,
  "preferredTimeEnabled": true,
  "preferredTime": "18:00",
  "quietHoursEnabled": true,
  "quietHoursStart": "09:00",
  "quietHoursEnd": "17:00",
  "destinations": [
    {
      "alias": "Primary NAS",
      "path": "/Volumes/PostStorage/FCP/ProjectBackups"
    }
  ]
}
```

Workflow:

1. Drop the file into the Setup folder.
2. Launch the app.
3. Non-destination settings are applied automatically.
4. Destinations appear as pending items.
5. Accept or reject each destination in Settings.

## MDM Workflow

FCP Backup Manager supports MDM-managed preferences under the domain:

```text
com.matx.FCPBackupManager
```

MDM-managed settings:

- apply automatically
- are locked in the UI
- override Setup JSON and local user settings

See [MDM-ADMIN-GUIDE.md](MDM-ADMIN-GUIDE.md) for the full managed-key reference.

## Troubleshooting

### No destinations are attempted

Check:

- destination is present in Settings
- destination is enabled
- destination is mounted and accessible
- destination was not excluded earlier by configuration state

Use:

- **Test Destinations**
- `app-diagnostic.log`

### A network destination says writable but does not copy

Check:

- latest `app-diagnostic.log`
- latest session log
- destination `backup.log`

Current versions of the app attempt mounted destinations even when free-space reporting is unreliable on network volumes.

### Nothing new was copied

This usually means the bundles were either:

- already backed up for those destinations and skipped by the manifest
- skipped by a Project Filter because the `.fcpbundle` name matched a keyword

If you want a fresh test, use **Reset Manifest…** and run again.

### Source folder not found

Verify the source path exists and matches where Final Cut is writing backups.

### Pending config badge remains

Open Settings and accept or reject pending destinations.

### MDM-locked settings cannot be changed

Those settings are controlled by managed preferences and must be changed through MDM.

## Configuration Scope

| Setting | JSON Setup | User Settings UI |
|---------|------------|------------------|
| Schedule Interval | Yes | Yes |
| Preferred Time | Yes | Yes |
| Quiet Hours | Yes | Yes |
| Source Path | Yes | Yes |
| Launch at Login | Yes | Yes |
| Project Filters | Yes | Yes |
| Backup Destinations | Yes, requires Accept | Yes |
| Retention Policy | Yes | Yes |
| Retention Keep Last N | Yes | Yes |
| Retention Delete Older Than Days | Yes | Yes |

MDM can manage and lock all of the above.
