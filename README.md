# OwnTV libmpv

The mpv engine inside OwnTV, packaged as an Android library: **libmpv + FFmpeg + the Kotlin/JNI
wrapper**, published as `tv.own.owntv:libmpv` to GitHub Packages and consumed by
[OwnTV Core](https://github.com/ahXN00/OwnTV_Core)'s `:player-core`. Both OwnTV apps (TV and mobile) get
it through core; neither app declares it.

It is a fork of [jarnedemeulemeester/libmpv-android](https://github.com/jarnedemeulemeester/libmpv-android)
(the library Findroid uses), kept as the `upstream` remote. The Kotlin API is unchanged, so OwnTV's
player code does not know the difference.

> **Monthly maintenance lives in [UPDATING.md](UPDATING.md).** Read it before touching a version.

## Why OwnTV builds its own

The P15b audit (`OwnTV_Core/future_work/ENGINE_SWAP_AUDIT_2026-09-25.md`, local only) compared the
upstream library, the mpvEx library and official mpv-android. In short:

| Need | Upstream jdtech 1.0.0 | mpvEx library | **This build** |
|---|---|---|---|
| Newest mpv (mpv makes no point releases; fixes live on master) | 0.41.0 (Dec 2025) | master, unpinned | **master, pinned commit, bumped monthly** |
| FFmpeg filters (deinterlace, night mode, levelling) | none — mpv's `bwdif` deinterlacer and every `af=lavfi` fail silently | all | **allowlist** (the ones mpv and OwnTV use) |
| Several mpv instances at once — OwnTV's `hardReset()` starts a fresh core while a stuck one is still being destroyed | yes | **no** (one global instance; a second `create()` aborts the app) | **yes** (upstream's JNI) |
| Reproducible from pinned sources | yes | no | **yes** |
| Size (arm64 + armv7 native, deflated) | 21.6 MiB | 30.7 MiB (Vulkan) | ≈ upstream + filters (no Vulkan) |

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
| `libmpv/build.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml` | publishes `tv.own.owntv:libmpv` to GitHub Packages instead of Maven Central; `abiFilters` |
| `renovate.json` | the mpv rule removed (mpv is bumped by the monthly workflow) |
| `tools/inspect_aar.py` | new — the build contract |
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
3. publishes `tv.own.owntv:libmpv:YYYY.MM.N` to GitHub Packages,
4. creates the GitHub Release with the AAR and the contract report.

A published version is immutable. Re-running on a tag that already published stops with `409 Conflict`;
fix forward with the next `N`.

Then OwnTV Core bumps `libmpv` in its `gradle/libs.versions.toml` — see [UPDATING.md](UPDATING.md).

## Consuming it

```kotlin
// settings.gradle.kts — GitHub's Maven registry needs a token with read:packages even for downloads.
maven {
    url = uri("https://maven.pkg.github.com/ahXN00/OwnTV_libmpv")
    credentials {
        username = providers.gradleProperty("gpr.user").orNull
        password = providers.gradleProperty("gpr.token").orNull
    }
    content { includeGroup("tv.own.owntv") }
}

// gradle/libs.versions.toml
libmpv = { group = "tv.own.owntv", name = "libmpv", version.ref = "libmpv" }
```

Credentials live in `~/.gradle/gradle.properties` or CI secrets, never in a repository.

## Licence

The wrapper code (Kotlin, JNI, build scripts) is MIT — see [LICENSE](LICENSE), kept from upstream.
The **binaries are GPLv3 as a whole**: FFmpeg is configured with `--enable-gpl --enable-version3` and mpv
is built with its GPL parts. Anything shipping this AAR ships GPLv3 code and owes its users the
corresponding source: this repository at the release tag plus the pinned upstream sources it names.
