# FLAC authenticity detection: FLAC_Detective verdict model, the spectral hi-res wall, Navidrome real-path reporting, and Lidarr's fail/blocklist/re-search flow

## Context

Building a report-only music-library authenticity scanner needed three things that are
under-documented upstream and had to be read from source or found by experimentation:

1. A detection design that flags lossy transcodes and fake hi-res **without** false-positiving
   genuine band-limited CD masters — which meant understanding FLAC_Detective's numeric verdict
   model and pairing it with an independent spectral measurement.
2. A way for a consumer (a dashboard) to join **Navidrome** albums to a filesystem-keyed report —
   which depends on a non-default Navidrome Subsonic setting with a subtle player-registration
   caveat.
3. A source-verified plan for **Lidarr**-side remediation (fail a bad grab, blocklist it, re-search
   for a clean release) for a future automated phase.

Everything below is verified against **FLAC_Detective v1.7.0**, **Navidrome 0.62.0**, and
**Lidarr v3.1.0.4875** (shared NzbDrone base, so Sonarr/Radarr behave identically for the Lidarr
findings).

## Finding — two-axis detection design (why one score is not enough)

A single "is this fake?" score conflates two unrelated defects, and each has a different
false-positive trap. The robust design uses **two independent axes** that never cross-escalate.

**Axis 1 — lossy transcode, from FLAC_Detective's numeric score.** FLAC_Detective returns a rich
dict from `FLACAnalyzer.analyze_file(path)`; the durable, machine-usable fields are `verdict`,
`score` (0..100+), `estimated_mp3_bitrate` (the estimated lossy source bitrate — a codec
fingerprint; 0 when none found), `cutoff_freq`, and `is_fake_high_res` (its own bit-depth-padding
detector). Its verdict bands (v1.7.0, `flac_detective.analysis.new_scoring.constants`):

| Verdict | Score | Meaning |
|---|---|---|
| AUTHENTIC | ≤ 30 | no evidence of transcoding |
| WARNING | 31–54 | borderline — check manually |
| SUSPICIOUS | 55–85 | likely a transcode |
| FAKE_CERTAIN | ≥ 86 | definitely transcoded |

`SCORE_SUSPICIOUS == 55`, `SCORE_FAKE_CERTAIN == 86`. FLAC_Detective's guiding principle is
"protect authentic files first" — a false alarm on real music is worse than missing a borderline
fake. A conservative consumer can raise the bar further: only call a file a hard **suspect** at
`score >= 86` **AND** with a non-zero `estimated_mp3_bitrate` (a codec fingerprint present); a high
score with no fingerprint stays a softer "review". Prefer the numeric `score` + fingerprint flag
over the raw verdict string, so the score→action mapping lives in exactly one place. Run the fast
path (`FLACAnalyzer(sample_duration=30.0, deep=False)`) — the optional 12th ML rule is off by
default and not needed for triage.

**Axis 2 — fake hi-res, from a spectral content-ceiling-vs-Nyquist "wall".** This is measured with
ffmpeg/ffprobe + numpy, independently of FLAC_Detective:

- **Content ceiling** — the highest frequency still carrying real energy — is found by a bisection
  that, at each candidate frequency, runs a chained `highpass=f=N:poles=2` (x3) + `volumedetect`
  over a mid-file window and checks whether `max_volume` clears a floor (~-70 dBFS). Everything
  above the ceiling is empty. This is more reliable than reading a single FFT bin because the
  3-stage highpass gives a steep, unambiguous rolloff.
- A file only "claims hi-res" when its **sample rate exceeds 48 kHz OR its bit depth exceeds
  16-bit**. On a >48 kHz file whose real content stalls well below the hi-res band (ceiling below
  ~26 kHz, or not found), the sample rate was **upsampled**. A >16-bit container whose real
  resolution is ~16-bit (FLAC_Detective's `is_fake_high_res`) is **bit-depth-padded**. Otherwise
  it is genuine hi-res.

**The false-positive guard is the whole point.** Two rules keep genuine CD-quality masters clean:

1. A file that makes **no** hi-res claim (≤ 48 kHz **and** ≤ 16-bit) is *always* classified
   `not_hires` — never "upsampled". A 44.1 kHz master legitimately rolls off around 20–22 kHz; its
   naturally low ceiling must never be read as upsampling.
2. The spectral wall must **never** feed the transcode axis. A band-limited genuine 44.1 kHz file
   is correctly judged at CD rate by FLAC_Detective; if a low ceiling could raise the transcode
   score, every clean CD rip would look transcoded. Keeping the axes independent is what prevents
   that.

A useful third output for human review (not a verdict input) is a **per-octave-band profile** —
energy (dB), temporal variance, and spectral flatness per octave from 20 Hz to Nyquist, from a
Hann-windowed FFT of a mono mid-segment. Flatness discriminates real structured content (tonal,
low flatness) from noise-floor / codec-fabricated content (flat), which visualises *why* a wall
was called.

## Finding — Navidrome real-path reporting seeds on newly-registered players only

To join a Navidrome album to a filesystem-keyed report by folder, the Subsonic API must return the
**real filesystem path** of a song, not the default tag-synthesized path. Navidrome exposes this as
`ND_SUBSONIC_DEFAULTREPORTREALPATH=true` (Navidrome 0.62.0): it makes `search3` / `getSong` report
the actual on-disk path so a consumer can derive each album's real library-relative folder.

**The caveat that costs debugging time:** the default is applied **only when a `(user, client)`
player is first registered.** Navidrome persists a per-player `ReportRealPath` flag at registration
time; a player that already exists (registered before the env var was set) keeps its old value —
tag-synthesized paths — until it is deleted and re-registered. A brand-new deployment therefore
"just works", but flipping the flag on an existing install silently does nothing for already-seen
clients.

**Workaround that makes it deterministic:** have the consumer query under a **distinct, dedicated
Subsonic client name** (e.g. a `gateway-quality` client separate from whatever fetches cover art).
A never-before-seen client name forces a fresh player registration, which picks up the current
`DEFAULTREPORTREALPATH` value automatically — no manual player deletion, works on a fresh deploy.
Match albums by normalised library-relative folder first, with (artist, album) as a fallback for
edge cases.

## Finding — Lidarr's fail → blocklist → re-search flow (for automated remediation)

`POST /api/v1/history/failed/{id}` is the single API call that both **blocklists** a grab and
**re-searches** for a replacement. Verified in Lidarr v3.1.0.4875:

- The endpoint is `HistoryController.MarkAsFailed([FromRoute] int id)` → `IFailedDownloadService
  .MarkAsFailed(id)`. **`{id}` is a history *record* id, and it must be the `Grabbed` event's
  history id** — not an album id, not a track id, not an import event. `MarkAsFailed(historyId)`
  looks the record up, reads its `DownloadId`, collects the `Grabbed` history for that download,
  and publishes a `DownloadFailedEvent` carrying the album ids taken from the grabbed history. (The
  code comment is explicit: "If the history item is a grabbed item (it should be, at least from the
  UI)".)
- `DownloadFailedEvent` has multiple handlers. Two matter:
  - `BlocklistService.Handle(DownloadFailedEvent)` inserts a `Blocklist` row keyed by ArtistId,
    AlbumIds, SourceTitle, Quality, Indexer, and (for torrents) the info hash — so Lidarr will not
    re-grab **that same release** again.
  - `RedownloadFailedDownloadService.Handle(DownloadFailedEvent)` (runs `EventHandleOrder.Last`)
    auto-searches for a replacement: it pushes an `AlbumSearchCommand(albumIds)` for one album (or
    the whole-artist / multi-album variants), i.e. it looks for a *different* release.
- The re-search is gated by config **`AutoRedownloadFailed`, which defaults to `true`**
  (`ConfigService.AutoRedownloadFailed` → `GetValueBoolean("AutoRedownloadFailed", true)`). So out
  of the box, one `history/failed/{grabbedId}` call blocklists the bad release **and** immediately
  kicks off a fresh search. There is a `skipRedownload` path in `FailedDownloadService`, but the
  **HTTP endpoint does not expose it** — the API always redownloads. To blocklist-only you would
  have to also disable `AutoRedownloadFailed` globally (or blocklist via a different mechanism).

**Anti-loop state model — mandatory for any automated remediation.** Because blocklisting the bad
release and auto-searching are coupled, a naive "fail every suspect album" loop will thrash:

- Blocklisting prevents re-grabbing the *same* release, but the AlbumSearch grabs the *next*
  candidate. If every available release for an album is itself a fake/transcode, the scanner flags
  the newly-imported one, fails it, searches again, grabs another fake — forever.
- The remediator must therefore keep its **own** per-album (or per-release) state: record which
  grabbed events it has already failed, and cap attempts per album (e.g. stop after N fails, then
  leave the album blocklisted-but-unsearched and surface it for manual review). Never re-fail a
  grabbed event you already failed.
- It must correlate on-disk suspect album → Lidarr `albumId` → the **most recent `Grabbed`
  history event** to obtain the correct `{id}`. If the grabbed event has aged out of history, the
  album cannot be auto-failed and must go to manual review.
- Act only on genuinely new/changed imports, not on every scan of an already-adjudicated library,
  or the daily scan itself becomes the loop driver.

## Source

FLAC_Detective v1.7.0 (`repos/FLAC_Detective`):

- Verdict bands + constants: `README.md` verdict table; `tests/test_verdict_thresholds.py` and
  `tests/test_new_scoring.py` (`SCORE_SUSPICIOUS == 55`, `SCORE_FAKE_CERTAIN == 86`);
  `flac_detective.analysis.new_scoring.constants` (`SCORE_WARNING`).
- Result dict + API: `src/flac_detective/analysis/analyzer.py` (`FLACAnalyzer.analyze_file`);
  `docs/api-reference.md` (`analyze_file`, result fields incl. `is_fake_high_res`,
  `estimated_mp3_bitrate`, `cutoff_freq`); `src/flac_detective/analysis/hires.py`
  (`is_fake_high_res` + high-depth → padded).

Navidrome 0.62.0: `ND_SUBSONIC_DEFAULTREPORTREALPATH` (real-path reporting on the Subsonic
`search3`/`getSong` response). Player-registration caveat discovered via experimentation — the flag
seeds a per-player `ReportRealPath` value at first registration; pre-existing players keep the old
value until re-registered. Workaround: query under a distinct Subsonic client name to force a fresh
registration.

Lidarr v3.1.0.4875 (`repos/Lidarr`; identical under the shared NzbDrone base in `repos/Sonarr`,
`repos/Radarr`):

- `src/Lidarr.Api.V1/History/HistoryController.cs` (`[HttpPost("failed/{id}")] MarkAsFailed` →
  `_failedDownloadService.MarkAsFailed(id)`).
- `src/NzbDrone.Core/Download/FailedDownloadService.cs` (`MarkAsFailed(historyId)`; requires the
  `Grabbed` event; publishes `DownloadFailedEvent` with album ids; `skipRedownload` overload not
  reachable from the HTTP endpoint).
- `src/NzbDrone.Core/Blocklisting/BlocklistService.cs` (`Handle(DownloadFailedEvent)` inserts the
  blocklist row).
- `src/NzbDrone.Core/Download/RedownloadFailedDownloadService.cs` (`Handle(DownloadFailedEvent)`,
  `EventHandleOrder.Last`; gated on `AutoRedownloadFailed`; pushes `AlbumSearchCommand`).
- `src/NzbDrone.Core/Configuration/ConfigService.cs` (`AutoRedownloadFailed` default `true`;
  `AutoRedownloadFailedFromInteractiveSearch` default `true`).
- `src/NzbDrone.Core/IndexerSearch/AlbumSearchCommand.cs` (`AlbumIds` payload).
