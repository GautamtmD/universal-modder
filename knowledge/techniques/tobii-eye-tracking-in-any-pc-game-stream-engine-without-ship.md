---
kind: technique
title: 'Tobii eye tracking in any PC game: Stream Engine without shipping Tobii''s files'
status: working
agents:
- Claude Code (Opus 5.5)
humans:
- GautamtmD
date: '2026-10-09'
links:
- https://developer.tobii.com/pc-gaming/downloads/
- https://developer.tobii.com/pc-gaming/develop/tobii-game-integration/getting-started/
- https://www.tobii.com/products/integration/tobii-sdk-license
- https://docs.rs/tobii-sys
tags: [tobii, eye-tracking, head-tracking, stream-engine, licensing, native-hook, input]
---
# Tobii eye tracking in any PC game: Stream Engine without shipping Tobii's files

> To add Tobii eye/head tracking to a game mod, load the `tobii_stream_engine.dll` that Tobii's own software
> (Tobii Experience) already installed on the player's PC. Call it through `GetProcAddress` with the handful of
> declarations you need, and ship only your own code. This avoids redistributing Tobii's runtime, which the
> license doesn't allow. The public documentation alone is not enough to get it right: the units, signatures
> and usage rules that matter are only in the SDK headers. This note lists them.

## When to use it
- A mod (REFramework plugin, ASI, BepInEx native shim, ReShade add-on, ...) wants gaze and/or head pose from a
  Tobii Eye Tracker 4C/5 on Windows, for camera lean ("extended view"), aim-at-gaze, clean UI and so on.
- You want to publish the mod (Nexus etc.) and can't ship Tobii's DLL.
- Not for research, recording or analytics: that needs Tobii's analytical license (gotcha 7).

## How
1. **Get the SDK for development.** Developers get headers from Tobii. Today the only public gaming download
   is **Tobii Game Integration (TGI)** 9.0.4 (developer.tobii.com/pc-gaming/downloads). The old Stream Engine
   pages under `/product-integration/stream-engine/` now redirect to a landing page. A Stream Engine SDK bundle
   (headers + `.lib` + DLL) still works if you have one: the headers embed the full API reference as comments.
   Tobii's SDK license page grants a development license "without commercial use or distribution"
   (distribution needs a separate agreement), and the headers carry Tobii AB's notice forbidding reproduction
   without written permission. So keep the headers private, and never ship them or the DLL.
2. **Find the runtime at runtime**, in this order:
   - next to the game exe (a user's own copy);
   - an ini override path;
   - `%ProgramFiles%\Tobii\Tobii EyeX\tobii_stream_engine.dll` (installed by Tobii Experience; 4.25.0.3 on the
     test PC), or `...\Tobii\Tobii Experience\...`;
   - plain `LoadLibrary("tobii_stream_engine.dll")`.

   Log which one loaded. If none loads, log once and stay idle: eye tracking is optional and must never take
   the game down.
3. **Declare only what you use** (from the headers or the API reference, in your own code), and `GetProcAddress`
   each one. Treat a missing export as "no eye tracking": `tobii_api_create`,
   `tobii_enumerate_local_device_urls`, `tobii_device_create`, `tobii_gaze_point_subscribe`,
   `tobii_head_pose_subscribe`, `tobii_device_process_callbacks`, `tobii_device_reconnect`, `tobii_error_message`,
   plus the matching destroy/unsubscribe calls. Calling convention `__cdecl`; errors are an `int` enum
   (0 = no error).
4. **Connect:** `tobii_api_create(&api, nullptr, nullptr)`, then enumerate URLs (copy the string inside the
   callback), then `tobii_device_create(api, url, TOBII_FIELD_OF_USE_INTERACTIVE /*1*/, &device)`, then
   subscribe to gaze point and head pose. Retry every few seconds if any step fails (tracker unplugged, service
   starting).
5. **Pump** `tobii_device_process_callbacks(device)` from a thread that runs at least 10×/s: a per-frame hook
   works, or a dedicated thread with `tobii_wait_for_callbacks`. Callbacks run synchronously inside that call.
   Copy the values into atomics; do the game-side work elsewhere.
6. **Map to the game.**
   - Gaze is normalized screen space: x 0 → 1 left → right, y 0 → 1 top → bottom.
   - Head pose: position in mm, rotation in radians (gotchas 3-4).
   - Smooth both (two-stage EMA worked well), use a deadzone, glide back to centre when data goes invalid,
     and gate off in cutscenes and menus.

## Gotchas
1. **Public declarations crash against the installed runtime.** **Cause:** the only browsable Stream Engine
   declarations online (the `tobii-sys` Rust crate on docs.rs) come from v1.2.1 headers, where
   `tobii_device_create(api, url, device)` has 3 parameters. The 4.x runtime Tobii Experience installs takes 4:
   `(api, url, field_of_use, device)`. Called the old way, the device pointer lands in the `field_of_use` slot.
   **Fix:** use the 4-parameter form with `TOBII_FIELD_OF_USE_INTERACTIVE` (1). The gaze point and head pose
   struct layouts are unchanged between those versions.
2. **The official docs moved and the gaming docs describe a different API.** Developer.tobii.com's PC Gaming
   section documents TGI (`ITobiiGameIntegrationApi`, `GetLatestGazePoint`, `GetLatestHeadPose`), not Stream
   Engine. Its getting-started page has one sample and no units, ranges or threading rules. **Fix:** treat the
   SDK headers' embedded reference as the source of truth, and this note's facts as verified on Stream Engine
   4.x (4.1.0.3 SDK copy and 4.25.0.3 installed copy).
3. **Head rotation units are undocumented, and community code gets them wrong.** The header says only
   "Euler angles using right-handed rotations around each axis". TGI's `HeadPose` *is* in degrees
   (`YawDegrees`), and the hand-written Go wrapper this project started from assumed degrees too. **Measured: Stream
   Engine reports radians.** Treating them as degrees made head-driven camera motion about 57× too weak.
   **Fix:** use radians; keep any user-facing range in degrees and convert.
4. **Axis mapping and signs.** `rotation_xyz[1]` = yaw, `[0]` = pitch, `[2]` = roll (around the axis pointing at
   the user). Each axis has its own validity flag (`rotation_validity_xyz[i]`). In RE2 the head yaw needed the
   opposite sign from our first guess, while pitch was right. **Fix:** ship hot-reloadable invert flags and
   verify with a human turning their head. `position_xyz` is mm from the display centre.
5. **Gaze leaves the 0–1 range.** Documented, but easy to miss: looking below the monitor gave y ≈ 1.7.
   **Fix:** clamp after mapping, or treat values well outside the range as "looking away".
6. **Tracking silently dies after a loading screen.** **Cause:** `process_callbacks` must run at least 10×/s or the
   connection drops. A present-driven pump stalls during long loads, after which the call returns
   `TOBII_ERROR_CONNECTION_FAILED` (5) or `..._DRIVER` (18). **Fix:** check the return value and call
   `tobii_device_reconnect(device)` with a cooldown, or pump from a dedicated thread using
   `tobii_wait_for_callbacks`.
7. **Logging gaze is "storing" it.** **Cause:** the license-free `TOBII_FIELD_OF_USE_INTERACTIVE` says eye-tracking data
   "is only used as a user input ... and cannot be stored, transmitted, nor analyzed". Periodic debug lines
   with gaze/head coordinates in a log file, or a WebSocket server streaming gaze to a browser, break that.
   Analytical use needs a Tobii license. **Fix:** log counts and validity rates only, and put raw-value logging
   behind an off-by-default developer switch.
8. **Head pose validity near zero while gaze is fine.** Usually the user is outside the head-tracking box (too
   close or far, off to one side) or partly occluded. It's not a struct-layout bug: verify the layout once
   against the header, then check seating. **Fix:** count valid/total per stream in the log, so "no data" and
   "bad data" are distinguishable.
9. **Timestamps drift if you never call `tobii_wait_for_callbacks`.** The epoch is undefined and the clocks drift
   unless `tobii_update_timesync` is called periodically. Harmless if timestamps are used only between
   consecutive samples (as here); matters for latency maths. **Fix:** call timesync about every 30 s, or don't
   rely on absolute timestamps.
10. **"Can I just ship the DLL with my mod?"** Not without Tobii's permission: development license only, plus a
    copyright notice that forbids reproduction. **Fix:** load the installed copy (step 2). Tested 2026-10-09 in
    RE2: with no runtime present the game ran normally with one log line; with only the installed 4.25 runtime
    the plugin connected to an Eye Tracker 5 with valid gaze and head pose.

## Seen in
- [Resident Evil 2 (2019): DLSS5 NR + DLSS frame generation + Tobii](../games/resident-evil-2-2019/dlss5-neural-rendering-dlss-frame-generation-in-a-custom-ref.md):
  REFramework plugin, gaze + head camera lean, cutscene gate, Tobii runtime loaded from Tobii Experience's
  install.
