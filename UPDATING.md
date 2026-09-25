# Monthly update — how OwnTV stays on the newest mpv

**Once a month, early in the month.** The goal is that OwnTV's mpv is never more than about a month
behind official mpv master, and FFmpeg never misses a point release. Each step says how it is checked.
Nothing here is released without a device test on the TV and the phone.

## 0. What moves, and who notices

| Source | Pinned in `depinfo.sh` as | Noticed by |
|---|---|---|
| mpv master | `v_mpv` (commit) | **`monthly-update.yaml`**, 1st of the month, 06:00 UTC — opens a PR |
| FFmpeg releases (`n9.0.x`, later `n9.1`, `n10`) | `v_ffmpeg` | Renovate PR, if the Renovate app is installed on this repo; else step 2 below |
| libplacebo, libass, dav1d, mbedTLS, freetype, fribidi, harfbuzz, libunibreak, libxml2, fontconfig, Lua | `v_*` | Renovate PR, else step 2 |
| NDK / SDK / Gradle / AGP | `depinfo.sh`, `libmpv/build.gradle.kts`, `gradle/` | Renovate PR, else step 2 |
| the upstream wrapper (jdtech) | the `upstream` remote | step 6 |

## 1. The mpv pull request

On the 1st, **Actions → Monthly mpv update** runs by itself (or run it by hand: *Run workflow*).

- **A PR "Monthly mpv update (…)" appeared.** The pin is bumped, the full build and the contract check
  have already passed in that run, and the PR body lists every new entry in mpv's
  `DOCS/interface-changes` since the old pin. Go to step 3.
- **No PR, run green.** mpv master has not moved. Nothing to do for mpv this month.
- **No PR, run red.** The bump did not build. Open the failed *Build the bump* job: usually a new
  minimum version of a dependency (raise it in `depinfo.sh` on the update branch and push; the push
  builds it) or a meson option renamed in `buildscripts/scripts/mpv.sh`.

## 2. The other dependencies

Merge Renovate's PRs if they are green. Without Renovate, check by hand and bump in `depinfo.sh`:

- FFmpeg: newest `n9.0.x` tag at https://github.com/FFmpeg/FFmpeg/tags. A point release is always taken
  (security fixes). A new major (`n9.1`, `n10.0`) is taken only once mpv master builds against it —
  the build proves it.
- The rest: their release pages (`renovate.json` names each source).

Every bump goes through a PR so `build.yaml` runs the build and the contract on it.

## 3. Review the mpv changes against what OwnTV sets

OwnTV configures mpv in exactly one file: `OwnTV_Core/player-core/src/main/java/tv/own/owntv/player/OwnTVPlayer.kt`.
List what it uses:

```sh
grep -ohE '(setOptionString|setProperty[A-Za-z]*|getProperty[A-Za-z]*|observeProperty)\("[^"]+"' OwnTVPlayer.kt | sort -u
```

For every interface-change entry in the PR body, ask: does it rename, remove, or change the default of
anything in that list? Known sensitive areas, with the setting OwnTV pins today to keep its behaviour:

| mpv area | Why it matters | OwnTV sets |
|---|---|---|
| network backend (`--curl-*`) | curl ignores FFmpeg's `reconnect_*` options and never retries a live (non-seekable) stream | `curl-enabled=no` — and this build has no libcurl anyway |
| TLS (`--tls-verify`) | master defaults to `yes`; IPTV panels often have bad certificates | `tls-verify=no` |
| editions / programs | `track-list` filtered to one HLS variant or TS program — breaks the Quality menu | `flatten-editions=yes` |
| `hwdec*`, `vd-lavc-*` | hardware decoding on Realtek / Amlogic | `hwdec-codecs` list, `vd-lavc-*` on the software rung |
| `demuxer-*`, `cache-*`, `stream-lavf-o` | live buffering and reconnects | the `PlayerBudget` values |
| `deinterlace`, `af`, `vf` | need the filter allowlist | `deinterlace=auto` off the direct path |

Also look at whether the two files OwnTV's main TV path depends on changed — they had not between 0.41.0
and `2a4eb8067`:

```sh
git -C <mpv clone> log --oneline <old>..<new> -- video/out/vo_mediacodec_embed.c audio/out/ao_audiotrack.c
```

Anything that needs a new setting in OwnTV goes into core in the same update, never into an app.

## 4. Release this library

Merge the PR(s) into `main`, then tag: `git tag vYYYY.MM.0 && git push origin vYYYY.MM.0`.
`publish.yaml` builds, checks, publishes `tv.own.owntv:libmpv:YYYY.MM.0` and creates the release.
Add a section to [CHANGELOG.md](CHANGELOG.md) first: mpv commit + describe, FFmpeg version, what changed.

## 5. Take it into OwnTV

In `OwnTV_Core`:

1. `gradle/libs.versions.toml` → `libmpv = "YYYY.MM.N"`.
2. Core builds: `./gradlew :core:assembleRelease :player-core:assembleRelease`, unit tests, and both
   apps' `assembleStandardRelease` against core's source.
3. **Device test** on the TCL TV and the phone (release APKs, `adb install -r`, data kept) — the `mpv ready`
   log line shows the mpv and FFmpeg versions:
   - VOD 4K HDR (direct path) and an old Xvid / MPEG-2 file (copy rung)
   - live on mpv: an HLS and a raw TS channel, one that drops (reconnect), an HTTPS panel
   - catch-up, ExoPlayer ⇄ mpv handover both ways, Multiview
   - image and text subtitles, surround and stereo, Night mode on mpv (by ear)
   - Quality menu on a multi-variant HLS channel
4. Core changelog under the next `core-<version>`; the core release carries it into both apps through
   the usual pin-bump PRs.

**A regression found in step 3 means no core bump this month.** Keep the old version pinned in core, write
what broke in this repo's CHANGELOG under the unreleased version, and either pin mpv one commit earlier or
wait for next month. The library release itself can stay — core decides what it ships.

## 6. Upstream wrapper changes (a few times a year)

```sh
git fetch upstream
git log --oneline main..upstream/main
git merge upstream/main
```

Conflicts can only be in the files listed in the README's "Differences from upstream" table. Keep ours
for the version pins and publishing, take theirs for JNI/Kotlin fixes. Upstream's own version bumps are
ignored: `depinfo.sh` here is the source of truth.

## 7. Record it

One line per month in [CHANGELOG.md](CHANGELOG.md), even for "no change": the date, what was checked,
and what was released or why not.
