# Lidarr/Servarr media-management config: recycle bin, rescan, and library-watch quirks

## Context

Lidarr on the homelab entered a re-download loop. A folder scan that returned a
partial directory listing zeroed a completed album's track-file links, the album
flipped to "missing" (0 files), the album search re-grabbed it, and import
permanently deleted the old files (empty recycle bin) then re-copied the whole
fileset — which triggered another scan and closed the loop. Mitigating it meant
reconciling `/api/v1/config/mediamanagement` to disable the high-frequency scan
triggers and add a recovery recycle bin.

The exact behavior of `recycleBin`, `recycleBinCleanupDays`, `rescanAfterRefresh`,
and `watchLibraryForChanges` is under-documented in the Servarr wiki and had to be
read from source. Everything below is verified against Lidarr **v3.1.0.4875**, and
because the code lives in the shared NzbDrone base it applies identically to Sonarr
and Radarr (the config keys and enum names are the same; only the refresh service is
per-app — `RefreshArtistService` in Lidarr, `RefreshSeries`/`RefreshMovie` elsewhere).

## Finding

**`recycleBin` must already exist when you set it via the API.** The
`config/mediamanagement` PUT runs `PathExistsValidator` on the `RecycleBin` field, so
a non-existent path returns HTTP 400 — Lidarr does NOT create the recycle-bin root
for you. It only auto-creates the per-delete *subfolders* inside the bin at delete
time (`RecycleBinProvider.CreateFolder(destinationFolder)`). Pre-create the folder
(e.g. Ansible `file: state=directory`) before the reconcile task runs.

**Dot-folders are auto-excluded from library scans, so a `.recycle` bin inside a root
folder is safe.** `DiskScanService.ExcludedSubFoldersRegex` skips any path segment
matching `\.[^\\/]+` (alongside `extras`, `@eadir`, `extrafanart`, `plex versions`,
`.@__thumb`). A recycle bin named `.recycle` living *inside* a Lidarr root folder is
therefore never re-scanned or re-imported. A plainly-named folder (e.g. `recycle`) in
a root folder would be picked up as library content and re-imported — the leading dot
is load-bearing.

**Put the recycle bin on the same mount as the library to make deletes cheap and
reversible.** `DiskTransferService` moves files to the bin with `TransferMode.Move`.
When source and target are on the same mount
(`sourceMount.RootDirectory == targetMount.RootDirectory`) it does a single
`MoveFolder` — a rename, instant, no data copied. Across mounts it falls back to
copying every file and then deleting the source. On an NFS-mounted library, a recycle
bin under the same NFS export is a rename; one on the container's local disk is a full
copy+delete of every deletion.

**`recycleBinCleanupDays` is dead in this version — nothing prunes the bin.**
`RecycleBinProvider` implements `IExecute<CleanUpRecycleBinCommand>`, but
`CleanUpRecycleBinCommand` is NOT in `TaskManager`'s scheduled-task list (the only
scheduled tasks are RefreshArtist, RescanFolders, Housekeeping, Backup, RssSync,
ImportListSync, CheckHealth, ApplicationUpdateCheck, MessagingCleanup, and
RefreshMonitoredDownloads). With no scheduler firing the command, the setting never
takes effect and the bin grows forever. An external cron must prune it.

**Config PUT is full-object only — never send a partial body.**
`ConfigController.SaveConfig` deserializes the whole `MediaManagementConfigResource`,
then reflects over *every* public property and writes each one back via
`SaveConfigDictionary`. Any field omitted from the JSON sits at its C# default before
that write, so a partial PUT silently clobbers value-type fields (int -> 0,
bool -> false, string -> null) to their defaults. Validation also runs on the whole
resource, so a partial or internally inconsistent body can 400. Always GET the current
resource, merge your changes into it, and PUT the full object back.

**`rescanAfterRefresh` does NOT gate the hardcoded 24h folder rescan.**
`rescanAfterRefresh` (`always` | `afterManual` | `never`, default `always`) is read
only in the per-app refresh service (`RefreshArtistService` in Lidarr). It controls
whether a rescan runs *as part of an artist/series/movie refresh*, and `afterManual`
restricts that to manual triggers. It does not touch `RescanFoldersCommand`, which
`TaskManager` schedules unconditionally every 24h (`Interval = 24 * 60`). No config
disables that scheduled rescan. Setting `rescanAfterRefresh: afterManual` removes the
refresh-driven rescans, but the daily RescanFolders survives — strong mitigation for a
scan-driven re-grab loop, not a guaranteed cure.

**`watchLibraryForChanges` is a `FileSystemWatcher` (inotify on Linux) and is
unreliable over NFS.** Default `true`. `RootFolderWatchingService` creates one .NET
`FileSystemWatcher` per root folder; on Linux that is backed by inotify, which does
not receive events for writes made on the NFS *server* (an NFS client only sees local
inotify events). On an NFS-mounted library the watcher both misses real changes and,
combined with partial scans, acts as a high-frequency trigger for spurious rescans.
Disable it (`watchLibraryForChanges: false`) on NFS libraries and rely on the 24h
RescanFolders plus explicit import-time updates.

## Source

Verified by reading the vendored Lidarr source in this knowledge base
(`repos/Lidarr`, v3.1.0.4875); the same files exist under the shared NzbDrone base in
`repos/Sonarr` and `repos/Radarr`:

- `recycleBin` PathExists requirement + subfolder auto-create:
  `src/Lidarr.Api.V1/Config/MediaManagementConfigController.cs` (`RuleFor(c =>
  c.RecycleBin).SetValidator(pathExistsValidator)`);
  `src/NzbDrone.Core/MediaFiles/RecycleBinProvider.cs`
  (`CreateFolder(destinationFolder)`, `TransferFile(..., TransferMode.Move)`).
- Dot-folder scan exclusion: `src/NzbDrone.Core/MediaFiles/DiskScanService.cs`
  (`ExcludedSubFoldersRegex`, pattern segment `\.[^\\/]+`).
- Same-mount rename vs cross-mount copy+delete:
  `src/NzbDrone.Common/Disk/DiskTransferService.cs` ("If we're on the same mount, do a
  simple folder move." -> `MoveFolder`).
- `recycleBinCleanupDays` unscheduled: `CleanUpRecycleBinCommand` is handled in
  `src/NzbDrone.Core/MediaFiles/RecycleBinProvider.cs` but is absent from the
  scheduled-task list in `src/NzbDrone.Core/Jobs/TaskManager.cs`.
- Full-object config PUT: `src/Lidarr.Api.V1/Config/ConfigController.cs` (`SaveConfig`
  reflects all public properties into `SaveConfigDictionary`).
- `rescanAfterRefresh` scope + enum:
  `src/NzbDrone.Core/Configuration/RescanAfterRefreshType.cs`
  (`Always`/`AfterManual`/`Never`), consumed only in
  `src/NzbDrone.Core/Music/Services/RefreshArtistService.cs`; hardcoded 24h rescan in
  `src/NzbDrone.Core/Jobs/TaskManager.cs` (`RescanFoldersCommand`, `Interval = 24 *
  60`).
- `watchLibraryForChanges` watcher:
  `src/NzbDrone.Core/Configuration/ConfigService.cs` (default `true`);
  `src/NzbDrone.Core/MediaFiles/RootFolderWatchingService.cs` (`FileSystemWatcher`).
