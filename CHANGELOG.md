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

#### Hop count always showing "Direct" in message details

Group channel messages always displayed **Hops: Direct** in the message details
panel, even for messages that arrived via multiple repeaters.

**Root cause:** MeshCore's `GroupTextHandler` publishes the decrypted message event
(`mesh.channel.message.new`) without the raw packet fields `path_len` and
`path_hops`. Those fields are only present on the earlier `packet` event that fires
before decryption. The adapter already correlated these two events to enrich sender
names, but it did not copy the path and signal fields across.

**Fix:** Extended the existing packet↔handler correlation in `_enrich_sender_names()`
to also propagate `path_len`, `path_hops`, `snr`, and `rssi` from the raw packet
event into the handler event before it is processed and stored.

> **Note:** Messages stored before this fix have `path_len = 0` and cannot be
> updated in-place (`INSERT OR IGNORE` prevents overwrites). Delete
> `~/.local/share/meshcore-console/meshcore.db` to start fresh; all new messages
> will display correct hop data immediately.

