# FCPBackupManager MDM Admin Guide

Reference for IT administrators deploying FCPBackupManager via managed preferences.

---

## Preference Domain

```
com.matx.FCPBackupManager
```

## Managed Keys

| Key | Type | Example | Description |
|-----|------|---------|-------------|
| `ScheduleInterval` | String | `"1d"` | Backup interval: `"1h"` (hourly), `"1d"` (daily), `"1w"` (weekly) |
| `SourcePath` | String | `"~/Movies/Final Cut Backups.localized"` | Path to the FCP backup source folder. Tilde is expanded at runtime. |
| `LaunchAtLogin` | Boolean | `true` | Whether the app starts automatically on user login |
| `BackupDestinations` | Array of Dict | See below | One or more backup destinations pushed to the client |
| `ProjectFilters` | Array of String | `["VIP", "archive"]` | Case-insensitive keyword filters. If a `.fcpbundle` name contains one of these strings, that bundle is skipped. |
| `AllowAdditionalDestinations` | Boolean | `false` | When `BackupDestinations` is managed, controls whether users or JSON setup files can add extra destinations. Defaults to `false` (fully locked — only MDM destinations allowed). Set to `true` to allow JSON setup and user-added destinations alongside MDM-managed ones. |
| `RetentionEnabled` | Boolean | `false` | Enable automatic cleanup of old backup zips at destinations |
| `RetentionKeepLastN` | Integer | `10` | Keep only the last N backups per project. Ignored if RetentionEnabled is false. |
| `RetentionDeleteOlderThanDays` | Integer | `90` | Delete backups older than N days. Ignored if RetentionEnabled is false. |

### BackupDestinations Format

Each entry in the `BackupDestinations` array is a dictionary with these keys:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `alias` | String | Yes | Display name shown in the UI and used in the manifest dedup key |
| `path` | String | Yes | Absolute path to the destination folder (e.g. `/Volumes/ProductionNAS/FCPBackups`) |
| `enabled` | Boolean | No | Whether this destination is active. Defaults to `true` if omitted. |

---

## How Managed Keys Appear in the UI

When a key is delivered via MDM configuration profile:

- The corresponding field in the Settings window is **greyed out and locked**.
- A lock indicator appears next to the field.
- The user cannot modify, disable, or remove MDM-managed values.
- MDM-managed destinations appear at the top of the destination list.

The app checks for managed status using `UserDefaults.objectIsForced(forKey:)`, which returns `true` for any key delivered through the macOS managed preferences system (written to `/Library/Managed Preferences/`).

---

## Priority System

Settings are resolved in this order (highest priority first):

1. **MDM Managed Preferences** — auto-applied, locked in UI, cannot be overridden
2. **Setup JSON files** — dropped in `~/Library/Application Support/FCPBackupManager/Setup/`. Non-destination fields auto-apply; destinations require user Accept.
3. **User Settings UI** — full manual control over anything not managed by MDM

If MDM manages `BackupDestinations` and `AllowAdditionalDestinations` is `false` (the default), then Setup JSON and user-added destination entries are blocked. Set `AllowAdditionalDestinations` to `true` to allow JSON setup and user-added destinations alongside MDM-managed ones. Users can never modify or remove MDM-managed entries regardless of this setting.

---

## Sample mobileconfig

Save the following as a `.mobileconfig` file and deploy via your MDM solution. Replace the UUIDs, organization name, and destination paths as needed.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadType</key>
			<string>com.matx.FCPBackupManager</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>PayloadIdentifier</key>
			<string>com.matx.FCPBackupManager.settings</string>
			<key>PayloadUUID</key>
			<string>A1B2C3D4-E5F6-7890-ABCD-EF1234567890</string>
			<key>PayloadDisplayName</key>
			<string>FCPBackupManager Settings</string>

			<key>ScheduleInterval</key>
			<string>1d</string>

			<key>SourcePath</key>
			<string>~/Movies/Final Cut Backups.localized</string>

			<key>LaunchAtLogin</key>
			<true/>

			<key>RetentionEnabled</key>
			<true/>

			<key>RetentionKeepLastN</key>
			<integer>10</integer>

			<key>RetentionDeleteOlderThanDays</key>
			<integer>90</integer>

			<key>BackupDestinations</key>
			<array>
				<dict>
					<key>alias</key>
					<string>Primary NAS</string>
					<key>path</key>
					<string>/Volumes/ProductionNAS/FCPBackups</string>
					<key>enabled</key>
					<true/>
				</dict>
			</array>

			<key>ProjectFilters</key>
			<array>
				<string>VIP</string>
				<string>archive</string>
			</array>

			<!-- Optional: allow users/JSON to add destinations alongside MDM ones -->
			<key>AllowAdditionalDestinations</key>
			<true/>
		</dict>
	</array>
	<key>PayloadDisplayName</key>
	<string>FCPBackupManager Configuration</string>
	<key>PayloadIdentifier</key>
	<string>com.matx.FCPBackupManager.profile</string>
	<key>PayloadOrganization</key>
	<string>Your Organization</string>
	<key>PayloadType</key>
	<string>Configuration</string>
	<key>PayloadUUID</key>
	<string>FEDCBA98-7654-3210-FEDC-BA9876543210</string>
	<key>PayloadVersion</key>
	<integer>1</integer>
</dict>
</plist>
```

### Adding Multiple Destinations

To push multiple destinations, add more `<dict>` entries inside the `BackupDestinations` array:

```xml
<key>BackupDestinations</key>
<array>
	<dict>
		<key>alias</key>
		<string>Primary NAS</string>
		<key>path</key>
		<string>/Volumes/ProductionNAS/FCPBackups</string>
		<key>enabled</key>
		<true/>
	</dict>
	<dict>
		<key>alias</key>
		<string>Secondary NAS</string>
		<key>path</key>
		<string>/Volumes/BackupNAS/FCPBackups</string>
		<key>enabled</key>
		<true/>
	</dict>
</array>
```

---

## Deployment via MDM Solutions

### SimpleMDM

1. In SimpleMDM, navigate to **Configs > Profiles **.
2. Click "Create Profile" and choose a **Custom Configuration Profile** payload.
3. Choose a name and drescription.
4. Upload the `.mobileconfig` file or enter the keys manually.
5. Assign the profile to the appropriate computer groups.

### Mosyle

1. Go to **Management > Profiles > Add new profile**.
2. Choose **Custom** profile type.
3. Upload the `.mobileconfig` file.
4. Assign to the target device group.
---

## Verifying Deployment

On a managed Mac, confirm the profile is installed:

```bash
# List installed profiles
sudo profiles list

# Check if keys are forced
defaults read /Library/Managed\ Preferences/com.matx.FCPBackupManager.plist
```

In the app, open Settings. Managed fields will display with a lock indicator and cannot be edited.

---

## Updating Configuration

To change managed settings, push an updated profile through your MDM. The app reads managed preferences on launch and periodically during runtime. Changes take effect on the next app launch or preference refresh cycle.

To remove management of a specific key, remove that key from the profile payload and re-deploy. The field will become user-editable in Settings.

---

## Configuration Scope Reference

| Setting | MDM Profile | JSON Setup | User Settings UI |
|---------|------------|------------|------------------|
| Schedule Interval | Yes (locked) | Yes (auto-applied) | Yes |
| Source Path | Yes (locked) | Yes (auto-applied) | Yes |
| Launch at Login | Yes (locked) | Yes (auto-applied) | Yes |
| Backup Destinations | Yes (locked) | Yes (requires Accept) | Yes |
| Project Filters | Yes (locked) | Yes (auto-applied) | Yes |
| Allow Additional Destinations | Yes | N/A | N/A |
| Retention Policy | Yes (locked) | Yes (auto-applied) | Yes |
| Retention Keep Last N | Yes (locked) | Yes (auto-applied) | Yes |
| Retention Delete Older Than Days | Yes (locked) | Yes (auto-applied) | Yes |
