# Changelog

One section per release, plus a line for every monthly check (UPDATING.md, step 7).

## 2026.09.0 — unreleased

First OwnTV build, forked from jarnedemeulemeester/libmpv-android `8687e8c` (after its 1.0.0).

- mpv `2a4eb8067` (v0.41.0-1072, master of 2026-09-23), was 0.41.0 in upstream 1.0.0.
- FFmpeg n9.0.2, was n8.1 in upstream 1.0.0.
- FFmpeg filter allowlist: mpv's deinterlacer (`bwdif`) and `af=lavfi` (night mode, levelling) now work.
- `mpegts` and `matroska` muxers; still no encoders.
- ABIs armeabi-v7a, arm64-v8a, x86_64 (32-bit x86 dropped).
- Published as `tv.own.owntv:libmpv` to OwnTV's public Maven repository (https://ahxn00.github.io/OwnTV_Core/maven,
  no login); Java package `dev.jdtech.mpv` unchanged.
