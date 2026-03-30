# Archive Format Comparison for FCP Backup Bundles

> **Context:** FCPBackupManager needs to archive `.fcpbundle` directory bundles (~5-10 MB each) containing XML timelines, plists, and metadata. The archive must preserve the project folder name and internal structure so that unarchiving recreates the original FCP hierarchy.

---

## Candidates

| Format | Tool | Produces | macOS Built-in |
|--------|------|----------|----------------|
| **ZIP** (via ditto) | `/usr/bin/ditto -c -k` | `.zip` | Yes |
| **ZIP** (via zip) | `/usr/bin/zip -r` | `.zip` | Yes |
| **TAR** | `/usr/bin/tar` | `.tar`, `.tar.gz`, `.tar.bz2` | Yes |
| **DMG** | `/usr/bin/hdiutil create` | `.dmg` | Yes |
| **AAR** (Apple Archive) | `/usr/bin/aa` | `.aar` | Yes (macOS 11+) |

---

## Detailed Comparison

### 1. ZIP via `ditto -c -k` ⭐ RECOMMENDED

```bash
ditto -c -k --keepParent --zlibCompressionLevel 6 \
  "ProjectName/BundleName.fcpbundle" \
  "ProjectName_BundleName.zip"
```

| Attribute | Rating | Notes |
|-----------|--------|-------|
| **macOS metadata** | ✅ Excellent | Preserves resource forks, extended attributes, `.localized`, ACLs |
| **Cross-platform read** | ✅ Good | Standard zip — opens on Windows/Linux (metadata stored in `__MACOSX/` sidecar, ignorable) |
| **Compression** | ✅ Good | zlib deflate, configurable level 1-9, ~60-70% on XML-heavy FCP data |
| **Speed** | ✅ Fast | ~100+ MB/s on typical FCP bundles |
| **Granularity** | ✅ | One zip per bundle, clean incremental model |
| **`--keepParent`** | ✅ | Preserves the parent folder name inside the zip — critical for our use case |
| **macOS availability** | ✅ | Ships with every macOS, SIP-signed, stable since 10.4 |
| **Finder integration** | ✅ | Double-click to extract. Finder uses ditto internally for zip operations |
| **Programmatic access** | ✅ | `Process()` in Swift, trivial to invoke |
| **Error handling** | ✅ | Exit codes, stderr output, well-documented |

**Why ditto over `/usr/bin/zip`:** The `zip` command does not preserve resource forks or extended attributes. `ditto` is Apple's recommended tool for creating macOS-aware archives. It stores HFS+ metadata in a `__MACOSX/` directory inside the zip, which macOS reads on extraction and other platforms safely ignore.

---

### 2. ZIP via `/usr/bin/zip -r`

```bash
zip -r "ProjectName_BundleName.zip" "ProjectName/BundleName.fcpbundle"
```

| Attribute | Rating | Notes |
|-----------|--------|-------|
| **macOS metadata** | ❌ Poor | Loses resource forks, extended attributes, ACLs |
| **Cross-platform read** | ✅ Good | Standard zip |
| **Compression** | ✅ Good | Same zlib deflate |
| **Speed** | ✅ Fast | Comparable to ditto |
| **`.localized` handling** | ⚠️ Risky | May not preserve Finder localization metadata |
| **macOS availability** | ✅ | Ships with macOS |

**Verdict:** Functional but loses macOS-specific metadata. Since FCP bundles may rely on extended attributes and the `.localized` suffix behavior, this is risky. No advantage over ditto.

---

### 3. TAR (`.tar.gz` / `.tar.bz2`)

```bash
tar -czf "ProjectName_BundleName.tar.gz" -C "source" "ProjectName/BundleName.fcpbundle"
```

| Attribute | Rating | Notes |
|-----------|--------|-------|
| **macOS metadata** | ⚠️ Partial | BSD tar on macOS can store xattrs with `--xattrs` but it's fragile and inconsistent across versions |
| **Cross-platform read** | ✅ Good | Universal on Linux/macOS; needs 7-Zip on Windows |
| **Compression** | ✅ Excellent | gzip fast; bzip2/xz better ratio but slower |
| **Speed** | ✅ Fast | gzip comparable to zlib; bzip2 slower |
| **Finder integration** | ⚠️ Partial | macOS opens `.tar.gz` via Archive Utility but creates a nested folder structure that can confuse users |
| **Granularity** | ✅ | Same per-bundle model works |
| **Tooling familiarity** | ⚠️ | Video editors are less familiar with tar than zip |

**Verdict:** Solid technically, but worse macOS metadata handling than ditto-zip, and less user-friendly for the target audience (FCP editors, not sysadmins). The xattr preservation story on macOS tar is inconsistent across OS versions — not worth the risk.

---

### 4. DMG (`hdiutil create`)

```bash
hdiutil create -srcfolder "ProjectName/BundleName.fcpbundle" \
  -format UDBZ "ProjectName_BundleName.dmg"
```

| Attribute | Rating | Notes |
|-----------|--------|-------|
| **macOS metadata** | ✅ Perfect | DMG is a disk image — it preserves *everything* (it's literally a filesystem snapshot) |
| **Cross-platform read** | ❌ Poor | Windows/Linux cannot natively open DMGs |
| **Compression** | ✅ Good | UDBZ (bzip2), ULFO (lzfse), UDZO (zlib) — configurable |
| **Speed** | ⚠️ Slower | Creating a disk image has more overhead than zipping: allocates virtual disk, creates HFS+/APFS filesystem, copies, compresses |
| **Granularity** | ⚠️ Awkward | One DMG per bundle is heavy. One DMG for all bundles loses incrementality |
| **Finder integration** | ⚠️ | Double-click mounts as volume — then user must copy out. Extra step vs zip extract |
| **Size overhead** | ❌ | DMG has filesystem overhead: partition map, volume header, allocation bitmap. For a 5 MB bundle, the DMG wrapper adds 1-5 MB |
| **Per-file overhead** | ❌ | Creating a DMG for each 5 MB bundle is disproportionate — designed for distributing large apps, not small backups |
| **Mount/unmount** | ⚠️ | Each DMG mount creates a `/Volumes/` entry. Browsing many backups means mounting many volumes |

**Verdict:** This was the original shell script approach (`hdiutil create`). It works, but it's the wrong tool for the job:
- **Massive overhead** for small bundles (filesystem metadata per DMG)
- **Poor granularity** — either one giant DMG (no incremental) or many tiny DMGs (wasteful)
- **macOS-only** — can't share backups with colleagues on other platforms
- **User friction** — mount → copy → unmount vs. just double-click a zip

DMG makes sense for distributing applications (one large image, used once). For recurring incremental backups of small bundles, it's the worst fit.

---

### 5. AAR (Apple Archive, `/usr/bin/aa`)

```bash
aa archive -i "ProjectName/BundleName.fcpbundle" \
  -o "ProjectName_BundleName.aar" \
  -a lzfse
```

| Attribute | Rating | Notes |
|-----------|--------|-------|
| **macOS metadata** | ✅ Excellent | Purpose-built by Apple to preserve all metadata: xattrs, ACLs, resource forks, flags, timestamps |
| **Cross-platform read** | ❌ Poor | No native support on Windows/Linux. Apple-only format |
| **Compression** | ✅ Excellent | LZFSE (Apple's modern algorithm) — better ratio + speed than zlib on Apple Silicon |
| **Speed** | ✅ Excellent | LZFSE is hardware-accelerated on Apple Silicon, very fast |
| **Finder integration** | ❌ None | Double-click does nothing. Must use `aa extract` or Archive Utility on macOS 12+ |
| **macOS availability** | ⚠️ | macOS 11+ only. `aa` CLI exists but AppleArchive framework has limited Swift API surface |
| **Maturity** | ⚠️ | Relatively new (2020). Less battle-tested. Sparse documentation |
| **Tooling** | ⚠️ | No third-party support. If Apple changes the format, there's no fallback |

**Verdict:** Technically the most metadata-faithful and fastest on Apple Silicon. But:
- **Zero cross-platform support** — locked to Apple ecosystem
- **No Finder double-click** — terrible UX for video editors
- **Vendor lock-in risk** — if Apple deprecates `aa`, there's no ecosystem to fall back on
- **Overkill** — FCP bundles are XML + plists. They don't have resource forks or complex ACLs that would benefit from AAR over ditto-zip

AAR would be interesting for archiving complex macOS app bundles with code signatures and entitlements. For FCP backup XML, it's using a sledgehammer to hang a picture.

---

## Summary Matrix

| Criteria | ditto zip | /usr/bin/zip | tar.gz | DMG | AAR |
|----------|-----------|-------------|--------|-----|-----|
| macOS metadata preserved | ✅ | ❌ | ⚠️ | ✅ | ✅ |
| Cross-platform readable | ✅ | ✅ | ✅ | ❌ | ❌ |
| Finder double-click extract | ✅ | ✅ | ⚠️ | ⚠️ mount | ❌ |
| Compression (FCP XML data) | Good | Good | Good | Good | Best |
| Speed | Fast | Fast | Fast | Slow | Fastest |
| Per-bundle overhead | Low | Low | Low | High | Low |
| Incremental-friendly | ✅ | ✅ | ✅ | ❌ | ✅ |
| User familiarity (editors) | ✅ | ✅ | ⚠️ | ✅ | ❌ |
| `--keepParent` (folder name) | ✅ | Manual | Manual | N/A | ✅ |
| Battle-tested | ✅ 20yr | ✅ 30yr | ✅ 50yr | ✅ 20yr | ⚠️ 5yr |
| Available macOS version | 10.4+ | 10.0+ | 10.0+ | 10.0+ | 11.0+ |

---

## Recommendation: `ditto -c -k --keepParent`

**ditto-zip wins because it's the only option that scores well on ALL dimensions:**

1. **Preserves macOS metadata** — resource forks, xattrs, `.localized` (unlike `/usr/bin/zip` or `tar`)
2. **Cross-platform readable** — standard zip, opens everywhere (unlike DMG or AAR)
3. **Finder-native extraction** — double-click just works (unlike AAR)
4. **`--keepParent` flag** — automatically preserves the parent folder name inside the zip. This is critical because we need `ProjectName/BundleName.fcpbundle/` as the internal structure
5. **Low overhead** — no filesystem wrapper like DMG
6. **Incremental-friendly** — one zip per bundle, trivial dedup model
7. **20+ years of stability** — ships with every macOS, is what Finder itself uses
8. **Fast enough** — FCP bundles are ~5-10 MB. Even unoptimized, the entire 129 MB source zips in seconds

The only tool that beats ditto technically is AAR (faster, better compression, perfect metadata). But AAR's complete lack of cross-platform support and Finder integration makes it unsuitable for a user-facing backup tool. If Apple ever ships AAR Finder integration and the format matures, it could be reconsidered in a future version.

### Exact Command

```bash
/usr/bin/ditto -c -k --keepParent --zlibCompressionLevel 6 \
  "~/Movies/Final Cut Backups.localized/MDOYVR2025-june12-pearl-mini/20250623_1929_PDT.fcpbundle" \
  "/tmp/FCPBackupManager/MDOYVR2025-june12-pearl-mini_20250623_1929_PDT.zip"
```

Produces a zip containing:
```
20250623_1929_PDT.fcpbundle/
├── Settings.plist
├── __BackupInfo.plist
├── CurrentVersion.flexolibrary
├── CurrentVersion.plist
└── 2025-06-12-part1/
    └── ...
```

> **Note:** `--keepParent` preserves the `.fcpbundle` folder name but not the grandparent project folder. In the backup engine, we handle this by structuring the destination with project-name subdirectories, or by wrapping in a parent directory before calling ditto. See implementation plan for details.
