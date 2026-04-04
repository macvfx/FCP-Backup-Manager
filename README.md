# FCP Backup Manager

Automated backup for Final Cut Pro backup bundles.

`v0.8.0` · `build 1` · `macOS 14+` · `SwiftUI` · `Menu Bar Utility`

---

FCP Backup Manager watches your Final Cut Pro backup source folder, compresses new `.fcpbundle` backups into zip files, and copies them to one or more destinations such as a local folder, external drive, NAS, SMB share, or other mounted volume.

## What It Does

- Discovers Final Cut backup bundles under `~/Movies/Final Cut Backups.localized/` by default
- Compresses each bundle with `ditto` to preserve macOS metadata
- Copies each bundle once per destination using manifest-based dedup
- Writes destination logs plus local troubleshooting logs
- Supports manual runs, scheduled runs with optional preferred time and quiet hours, retention cleanup, bundle-name filters, Setup JSON import, and MDM-managed deployment

## Quick Start

1. Open the menu bar app and choose **Settings…**.
2. In the **Source** area, confirm the source path and add any optional `.fcpbundle` keyword filters.
3. In the **Destinations** area, add one or more destinations.
4. Click **Test Destinations** to confirm access and writable status.
5. Click **Back Up Now**.
6. Use **View Logs…** in the menu bar or Settings if you need troubleshooting details.

## How It Works

```text
1. SCAN      Discover .fcpbundle directories under the source path
2. DEDUP     Skip bundles already recorded for each destination
3. ZIP       Create a per-bundle zip archive with ditto
4. COPY      Write the zip into destination/FCPBackupManager/<hostname>/
5. LOG       Write destination logs, session logs, and a local app diagnostic log
6. CLEANUP   Remove temporary staging files
```

Each discovered bundle becomes its own zip archive. A bundle is copied once per destination unless the destination copy is missing, in which case the stale manifest entry is repaired and the bundle is copied again.

## Bundle Filters

Project Filters in Settings, Setup JSON, and MDM now work as bundle-name filters.

- Matching is case-insensitive
- Filters are checked against the `.fcpbundle` folder name
- A filter like `VIP` will skip any backup bundle whose name contains `VIP`
- Diagnostic logs record the exact keyword and bundle name when a filter matches

## Destination Layout

```text
/Volumes/MyDrive/
└── FCPBackupManager/
    └── edit-suite-03/
        ├── ProjectA_20250623_1929_PDT.zip
        ├── ProjectB_20250710_0815_PDT.zip
        ├── backup.log
        └── edit-suite-03-2026-03-23_0926.log
```

The stable `backup.log` is the latest destination log. The timestamped log provides a per-run record.

## Logs

FCP Backup Manager writes logs in three places:

- Destination run logs at `destination/FCPBackupManager/<hostname>/`
- Session logs at `~/Library/Application Support/FCPBackupManager/Logs/`
- App diagnostic log at `~/Library/Application Support/FCPBackupManager/Logs/app-diagnostic.log`, plus daily rotated `app-diagnostic-YYYY-MM-DD.log` files

These are especially useful for network-volume troubleshooting because they show destination filtering, bundle-filter matches, copy attempts, fallback writes, and per-run results.

## Configuration Priority

Settings resolve in this order:

| Priority | Source | Behavior |
|----------|--------|----------|
| 1 | MDM Profile | Auto-applied and locked in the UI |
| 2 | Setup JSON in App Support | Schedule/source/retention applied automatically; destinations require user acceptance |
| 3 | Imported JSON | Same behavior as Setup JSON |
| 4 | User Settings UI | Manual control for non-managed settings |

See [MDM-ADMIN-GUIDE.md](MDM-ADMIN-GUIDE.md) for managed preference details.

## Documentation

- [User Guide](USER-GUIDE.md)
- [MDM Admin Guide](MDM-ADMIN-GUIDE.md)
- [Archive Format Comparison](ARCHIVE-FORMAT-COMPARISON.md)

## License

Apache 2.0
