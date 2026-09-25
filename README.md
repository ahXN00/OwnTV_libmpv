# OwnTV libmpv

The mpv engine inside OwnTV, packaged as an Android library: **libmpv + FFmpeg + the Kotlin/JNI
wrapper**, published as `tv.own.owntv:libmpv` to OwnTV's public Maven repository
(https://ahxn00.github.io/OwnTV_Core/maven, no login needed) and consumed by
[OwnTV Core](https://github.com/ahXN00/OwnTV_Core)'s `:player-core`. Both OwnTV apps (TV and mobile) get
it through core; neither app declares it.

It is a fork of [jarnedemeulemeester/libmpv-android](https://github.com/jarnedemeulemeester/libmpv-android)
(the library Findroid uses), kept as the `upstream` remote. The Kotlin API is unchanged, so OwnTV's
player code does not know the difference.

> **Monthly maintenance lives in [UPDATING.md](UPDATING.md).** Read it before touching a version.

## Why OwnTV builds its own

| What OwnTV needs | Upstream — jarnedemeulemeester/libmpv-android 1.0.0 | **Ours — OwnTV libmpv** |
|---|---|---|
| **The newest mpv.** mpv makes no bug-fix releases; every fix after a release is only on its main branch | 0.41.0 (December 2025) | **newest main branch, checked and updated every month** |
| **Interlaced TV channels without comb lines** (mpv's automatic deinterlacing) | ❌ does not work — the needed FFmpeg filter is missing | ✅ works |
| **Night mode and volume levelling on the mpv player** | ❌ plays silence — FFmpeg filters are missing | ✅ available |
| **Recovery from a frozen stream** — OwnTV starts a fresh player while the frozen one is still shutting down | ✅ | ✅ (same code as upstream) |
| **Newest FFmpeg** (the part that opens and decodes the streams) | 8.1 | **9.0.2** |
| **Rebuildable exactly** — every ingredient pinned to a fixed version | ✅ | ✅ |
| **Download size** (arm64, compressed) | 11.0 MB | 11.3 MB |
| **How often it gets updated** | whenever its one maintainer releases (5 times in 27 months) | **every month** ([UPDATING.md](UPDATING.md)) |

## Versions compared

What official mpv offers, what the upstream library we forked ships, and what this build ships.
The "ours" column changes with every release — update it in the same commit as `depinfo.sh`.

| | Official mpv | Upstream — jarnedemeulemeester/libmpv-android 1.0.0 | **Ours — OwnTV libmpv** |
|---|---|---|---|
| mpv | newest release **0.41.0** (2025-12-21); no point releases, fixes land on master only | 0.41.0 | **master `2a4eb8067`** (0.41.0 + 1,072 commits, 2026-09-23) |
| FFmpeg | — (newest FFmpeg release: 9.0.2) | 8.1 | **9.0.2** |
| FFmpeg filters | — | none | **29, allowlisted** (deinterlace, rotation, audio dynamics, EQ + plumbing) |
| FFmpeg muxers / encoders | — | none / none | mpegts, matroska / none |
| libplacebo · libass · dav1d | — | 7.360.1 · 0.17.4 · 1.5.3 | 7.360.1 · 0.17.5 · 1.5.4 |
| Several instances at once | — | yes | yes (same JNI) |
| ABIs | — | armeabi-v7a, arm64-v8a, x86, x86_64 | armeabi-v7a, arm64-v8a, x86_64 |
| Published as | — | `dev.jdtech.mpv:libmpv` (Maven Central) | `tv.own.owntv:libmpv` (OwnTV Maven, no login) |
| How often it moves | a release every 6–9 months | when its maintainer tags (5 releases in 27 months) | **monthly** ([UPDATING.md](UPDATING.md)) |

## What is inside

Every version is pinned in [`buildscripts/include/depinfo.sh`](buildscripts/include/depinfo.sh).

- **mpv**: a pinned commit on `master` (`v_mpv`). Built with Lua, libass, libplacebo; no libcurl,
  no Vulkan.
- **FFmpeg**: a release tag (`v_ffmpeg`), `--enable-gpl --enable-version3`, MediaCodec + JNI, mbedTLS,
  dav1d, libxml2 (DASH). **No encoders.** Muxers: `mpegts`, `matroska` only (so mpv's `stream-record`
  can write what it is already playing). Filters: the allowlist in
  [`buildscripts/scripts/ffmpeg.sh`](buildscripts/scripts/ffmpeg.sh), each commented with who needs it.
- **ABIs**: `armeabi-v7a`, `arm64-v8a`, `x86_64` (no 32-bit x86, which neither app ships).
  64-bit libraries are 16 KB page-aligned.
- **minSdk** 26. **Java package** `dev.jdtech.mpv` (kept from upstream on purpose).

### Differences from upstream, in full

| File | Change |
|---|---|
| `buildscripts/include/depinfo.sh` | `v_mpv` is a master commit, not a release |
| `buildscripts/include/download-deps.sh` | mpv cloned blobless + checkout of the pinned commit |
| `buildscripts/scripts/ffmpeg.sh` | filter allowlist, `mpegts`/`matroska` muxers |
| `buildscripts/build.sh` | no 32-bit x86 |
| `libmpv/build.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml` | publishes `tv.own.owntv:libmpv` to OwnTV's Maven repository instead of Maven Central; `abiFilters` |
| `renovate.json` | the mpv rule removed (mpv is bumped by the monthly workflow) |
| `tools/inspect_aar.py` | new — the build contract |
| `tools/publish_pages.py` | new — writes into OwnTV's Maven repository (identical copy in OwnTV_Core) |
| `.github/workflows/*` | `build`, `publish`, `monthly-update` (upstream's `publish.yaml` removed) |

Keep this table true: it is what makes an `upstream` merge reviewable.

## The build contract

Every CI build ends with `tools/inspect_aar.py`, which fails the build unless:

- the AAR holds exactly the three ABIs, every 64-bit `.so` is 16 KB-aligned, nothing links `libvulkan`;
- the mpv inside is the commit pinned in `depinfo.sh`;
- every allowlisted filter is really compiled in (read from FFmpeg's generated `filter_list.c`).

It also prints the mpv/FFmpeg versions and the per-ABI size; the publish workflow puts that report in
the GitHub Release notes.

## Building

The native build needs Linux (or macOS); it runs in GitHub Actions. On a Linux machine:

```sh
cd buildscripts
./download.sh        # SDK, NDK and all pinned sources
./patch.sh           # patches in buildscripts/patches/ (none at the moment)
./build.sh           # all ABIs, then the AAR
python3 ../tools/inspect_aar.py ../libmpv/build/outputs/aar/libmpv-release.aar
```

Single component or ABI: `./build.sh -n ffmpeg`, `./build.sh --arch arm64 mpv`, then `./build.sh -n`.

## Releasing

Versions are date-based: **`YYYY.MM.N`** (`2026.09.0`, then `2026.10.0`; `N` counts extra releases in
the same month). Push the tag `vYYYY.MM.N` on `main`:

1. `publish.yaml` checks the tag format,
2. runs the full build and contract check (the same `build.yaml` as every push),
3. publishes `tv.own.owntv:libmpv:YYYY.MM.N` to OwnTV's Maven repository (the `gh-pages` branch of
   OwnTV_Core, written with the `CORE_BUMP_TOKEN` secret; the newest 12 versions are kept),
4. creates the GitHub Release with the AAR and the contract report,
5. opens a **"Pin libmpv YYYY.MM.N" pull request on OwnTV_Core**. It never merges itself — a new
   engine waits for the device test.

A published version is immutable. Re-running on a tag that already published stops with "already
published"; fix forward with the next `N`.

Then the pin PR on OwnTV Core is reviewed, device-tested and merged — see [UPDATING.md](UPDATING.md).

## Consuming it

```kotlin
// settings.gradle.kts — public, no credentials
maven {
    url = uri("https://ahxn00.github.io/OwnTV_Core/maven")
    content { includeGroup("tv.own.owntv") }
}

// gradle/libs.versions.toml
libmpv = { group = "tv.own.owntv", name = "libmpv", version.ref = "libmpv" }
```


## Licence

The wrapper code (Kotlin, JNI, build scripts) is MIT — see [LICENSE](LICENSE), kept from upstream.
The **binaries are GPLv3 as a whole**: FFmpeg is configured with `--enable-gpl --enable-version3` and mpv
is built with its GPL parts. Anything shipping this AAR ships GPLv3 code and owes its users the
corresponding source: this repository at the release tag plus the pinned upstream sources it names.
