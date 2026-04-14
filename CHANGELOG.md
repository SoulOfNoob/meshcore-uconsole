# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## v1.9.0 (2026-03-08)

### Feat

- Add loading/shutdown screen widget
- Add pre-flight conflict detection for radio hardware (#48)

## v1.8.0 (2026-02-22)

### Feat

- Add telemetry request and response

### Fix

- Don't show self in contacts
- Use gpsd if available

## v1.7.0 (2026-02-19)

### Feat

- Analyzer shows decrypted content and route indicators
- Store channel kind explicitly in Channel model
- Surface radio hardware errors in StatusPill and toast

### Fix

- Fix private channel messaging and message dedup
- Need to fall back out of decrypted not existing
- Add hardware presets
- Saving settings shouldn't freeze UI

## v1.6.1 (2026-02-18)

### Fix

- Fix unread message handling

## v1.6.0 (2026-02-17)

### Feat

- Add favorite capabilities for peers

### Fix

- Migrate all polling behavior to use glib async
- Accessibility

## v1.5.0 (2026-02-17)

### Feat

- Dynamic scaling
- Add mentions
- Add more packet handling to analyzer
- Add hashtag-channel adding from UI
- Add CONTROL packet handling

### Fix

- Fix main width again and wraparound message text
- Update sizing method with larger fonts
- Slight width overflow
- Message wordwrap
- Sort peers reverse-chronilocallcally
- Rely on font with emojis on Pi
- Multibyte unicode not displaying correctly
- Mentions are always bracket-wrapped
- Potentially fix emoji / UTF8 node names
- Add channel to channel list after import

## v1.4.0 (2026-02-16)

### Feat

- Add path view on messages

### Fix

- Autoscroll channel to bottom when loading channel pages
- Speculative grp_text fixes
- Group texts not appearing in channels

## v1.3.1 (2026-02-15)

### Fix

- Day separator was not showing
- Details box shows details for wrong packet

## v1.3.0 (2026-02-15)

### Feat

- Add monkey testing script for UI stress testing

### Fix

- Resolve GTK widget assertions found by monkey testing
- Fixup screenshot generation (#23)

## v1.2.0 (2026-02-14)

### Feat

- Add ability to import private channel

### Fix

- Consistent timestamps across GUI
- Sending and receiving messages different channels
- Let pymc handle out_path
- Peer data not refreshing on advert
- Historical packets showed wrong timestamp
- Fix dms again

## v1.1.0 (2026-02-13)

### Feat

- Add emoji support
- Add more node info badges
- Add node prefix to channel view, fix viewport
- Add log level setting to settings and log it somewhere (#17)

### Fix

- Node disposal issues
- Make analyzer columns a bit bigger
- Make content text more robust
- Case-sensitive DMs
- Remove double CLIs

## v1.0.0 (2026-02-13)

### Feat

- Move from json-based cache to sqlite
- Add day change analyzer line
- Add map follow mode
- Add autoconnect setting

### Fix

- Resolve sender nodes of packets from known peers
- Add pycore to deps
- Don't crash gtk on gpio issues
- Not getting messages in channels
- Poll on GPS pin
- Repeaters show as nodes
- Fix settings savings crashing the session
- Hook up DM<->Channel correctly
- Refresh channel list when channel created
- Create DM channels when DMs received
- Stable key
- Add contacts db
- Also fix received callbacks
- Wrong dispatcher callback
- Public key not showing on settings page

### Perf

- Reduce CPU/IO load on Raspberry Pi hot paths

## v0.2.1 (2026-02-13)

### Fix

- Hallucinated APIs used in pymc_core

## v0.2.0 (2026-02-13)

### Feat

- add conventional commits and automated releases

### Fix

- Don't land under 'Internet'
- **deb**: Update to libgpiod3

## Unreleased

### Fix

#### Fixed position in settings not applied to adverts, telemetry, or map

Setting a latitude/longitude under *Public Info* in Settings and enabling
*Share GPS Position* had no effect. Sent adverts always contained `lat=0.0, lon=0.0`,
telemetry responses returned no location, and the device map marker disappeared when
no hardware GPS was connected.

Three layered bugs, all in `meshcore/client.py` and `platform/gps.py`:

**Bug A — `send_advert()` ignored lat/lon entirely.**
Called `session.send_advert()` without forwarding coordinates; pyMC_core defaulted
to `0.0, 0.0`. Fix: read `share_position` from settings, prefer live GPS fix, fall
back to `settings.latitude` / `settings.longitude`.

**Bug B — `create_gps_provider()` fell back to `MockGps` (San Francisco).**
When no gpsd and no GPS serial device were found, the production code returned
`MockGps`, which has `has_fix()=True` and returns SF waypoints. Bug A's fix checked
`if loc:` — found SF — and used it, silently overriding settings. Fix: replaced the
production fallback with `NullGps` (always returns `None`). `MockGps` is now only
used in explicit mock mode (`MESHCORE_MOCK=1`).

**Bug C — `get_device_location()` only read from GPS provider.**
Map marker disappeared; "Center on device" showed "GPS acquiring satellites…". Fix:
same GPS → settings fallback as `send_advert()`.

**Consistency fix:** All three callsites treat `(0.0, 0.0)` as "not set" and send no
location rather than placing the node at the equator/prime-meridian intersection.

