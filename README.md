# Reolink integration for NVR firmware 2.x

Home Assistant's official `reolink` integration, rebuilt automatically from every
Home Assistant release with a small patch that makes it work on NVRs running
**firmware 2.x** — devices for which Reolink never shipped a 3.x firmware, so
"just update the device" is not an option.

Installed through HACS, it lands in `custom_components/reolink/` and **overrides
the built-in integration**. Nothing needs to be disabled: Home Assistant always
prefers a custom integration over the one shipped with it.

Verified on an **RLN8-410-E** (hardware H3MB16, firmware `v2.0.0.269_20042901`).

## What is patched

The integration itself needs one change, in `media_source.py`: NVR recordings are
normally fetched with the `Download` command, which firmware 2.x does not have,
so they are requested over plain RTMP instead.

Everything else lives in [materko/reolink_aio](https://github.com/materko/reolink_aio)
(branch `fw2x`), the patched Reolink API library this build depends on:

- **`Search` may not span two calendar months** on firmware 2.x — it answers
  `rspCode -12` ("get config failed"). Upstream deliberately queries a two-month
  range to halve the number of requests, which is exactly what breaks. Searches
  are split per month instead.
- **Recordings play only over plain RTMP.** The `/flv?...playback.bcs` wrapper
  returns 404 and the `Playback` command answers "not support"; the device's own
  web interface uses `rtmp://<host>:1935/bcs/playback.bcs?...&token=...`.
- **RTMP accepts token authentication only** — user/password fails with an I/O
  error, for live streams as well as playback.
- `GetDevInfo` reports `type` instead of `exactType` (already handled upstream).

All of it is gated behind a `firmware_v2` check, so behaviour on 3.x firmware is
untouched.

## Installation

Add this repository to HACS as a custom repository of type **Integration**, download
it, and restart Home Assistant. After setup, open **Configure** on the integration
and switch the stream protocol to **RTMP** — firmware 2.x does not serve a working
RTSP main stream.

## How it stays up to date

`.github/workflows/build.yml` runs daily. It fetches the `reolink` integration from
the newest Home Assistant release, applies `patches/fw2x.patch`, points the manifest
at the pinned library commit, and — only when the result actually differs from the
last build — commits it and publishes a release named after the Home Assistant
version. HACS then offers it as an update.

If Home Assistant changes the same lines the patch touches, `git apply` fails and the
workflow errors out, which GitHub reports by e-mail. That is the one step no
automation can take over: someone has to decide how the two changes fit together.
The patch is refreshed in [materko/homeassistant-core](https://github.com/materko/homeassistant-core)
(branch `fw2x`) and copied back here.

Updating the library is deliberately manual too: the `REOLINK_AIO` variable at the
top of the workflow pins an exact commit, so a build never silently picks up
something new.

## Related repositories

| Repository | Role |
| :--- | :--- |
| [materko/reolink_aio](https://github.com/materko/reolink_aio) | patched API library, branch `fw2x` |
| [materko/homeassistant-core](https://github.com/materko/homeassistant-core) | Home Assistant fork where the patch is developed, branch `fw2x` |
| [materko/reolink_cctv](https://github.com/materko/reolink_cctv) | the older standalone integration, kept as a fallback |
