# Session Recap — April 2026

## Branch

`claude/fix-hop-count-display-mUIrw` — pushed, not yet merged.

## Hardware context

Raspberry Pi CM5 Lite, uConsole form factor, Debian Trixie.
HackerGadgets AIO LoRa board (SPI1, no GPS module connected during this session).
App launched via `./scripts/run-gtk-pi.sh` from git checkout alongside an apt-installed version.

---

## Bugs fixed

### 1. Hop count always showing "Direct" (issue: hops display)

**Root cause:** `GroupTextHandler` in pyMC_core publishes the decrypted
`mesh.channel.message.new` event without `path_len` / `path_hops`. Those fields only
exist on the earlier raw `packet` event. The adapter already correlated these two
events to enrich sender names (`_enrich_sender_names` in `meshcore/client.py`) but
did not copy the path and signal fields across.

**Fix:** Extended the correlation block to also propagate `path_len`, `path_hops`,
`snr`, and `rssi` from the raw packet event into the handler event before it is
processed and stored. Commit `7909e2e`.

**Important nuance confirmed on hardware:** "Direct" is sometimes *correct*. In
MeshCore flood routing every node receives whichever physical copy reaches it first.
If the sender was physically close (RSSI ≈ −35 dBm) the uConsole received the
direct copy; a remote iPhone received the same logical message via 8 hops. Both
readings are accurate.

**DB caveat:** `INSERT OR IGNORE` prevents updating old records. Delete
`~/.local/share/meshcore-console/meshcore.db` to start fresh; all new messages
will show correct hop data.

---

### 2. Bootstrap script blanking the uConsole display

**Root cause:** `scripts/bootstrap-pi.sh` called `raspi-config nonint do_spi 0`
which appended `dtparam=spi=on` to `/boot/firmware/config.txt`. On the CM5 uConsole,
SPI0 is used by the DSI display driver; enabling it conflicts and blanks the screen.

**Fix:** Removed the `raspi-config` call. The AIO LoRa board only needs
`dtoverlay=spi1-1cs` (SPI1). Added an auto-remediation `sed` step that strips any
pre-existing `dtparam=spi=on` line for users who ran the old script. Commit `bd6d973`.

---

### 3. Messages not auto-scrolling to new incoming messages

**Root cause:** `GLib.idle_add(scroll_to_bottom)` was called from `_poll_messages()`
immediately after appending the new widget. The call fired *before* GTK's layout
pass had updated `adj.get_upper()`, so the scroll landed at the old bottom.

**Fix:** Connected to `Gtk.Adjustment::changed` signal
(`ui_gtk/views/messages.py`), which fires after layout has updated `upper`. The
handler schedules `scroll_to_bottom` via `GLib.idle_add` so the scroll executes
with the final content height.

A second issue emerged: calling `adj.set_value()` directly inside the `"changed"`
callback (during layout) used a preliminary widget height, clipping the last
message. Fixed by using `GLib.idle_add` inside the handler instead of a direct
set. Commits `9eee284`, `85f14f9`.

---

### 4. Fixed position in settings not applied to adverts, telemetry, or map (issue #51)

Three layered bugs, all in `meshcore/client.py` and `platform/gps.py`:

**Bug A — `send_advert()` ignored lat/lon entirely.**
Called `session.send_advert(name=name, route_type=route_type)` with no coordinates;
pyMC_core defaulted to `0.0, 0.0`. Fix: read `share_position` from settings, prefer
live GPS, fall back to `settings.latitude` / `settings.longitude`. Commit `070fd2e`.

**Bug B — `create_gps_provider()` fell back to `MockGps` (San Francisco).**
When no gpsd and no `/dev/ttyAMA0` were found, the production code returned `MockGps`,
which has `has_fix()=True` and `get_location()` returning SF waypoints. Bug A's fix
checked `if loc:` first — found SF — and used it, silently overriding settings.
Fix: replaced the production fallback with `NullGps` (always returns `None`).
`MockGps` is now only used in explicit mock mode (`MESHCORE_MOCK=1`). Commit `8e51e9d`.

**Bug C — `get_device_location()` only read from GPS provider.**
Map marker disappeared; "Center on device" showed "GPS acquiring satellites...".
Fix: same GPS → settings fallback as `send_advert()`. Commit `26be2c4`.

**Consistency fix:** `send_advert()` and `_get_local_telemetry()` were missing the
`(lat != 0.0 or lon != 0.0)` guard that `get_device_location()` already had. Without
it, unconfigured users broadcast `(0.0, 0.0)`, placing their node in the Gulf of
Guinea. All three callsites now treat `(0.0, 0.0)` as "not set". Commit `7911522`.

---

## Key technical findings

### Version conflict (apt vs git)

The user had both the apt-installed package and the git checkout. Both use the same
SQLite database at `~/.local/share/meshcore-console/meshcore.db`. `INSERT OR IGNORE`
means whichever process stores a message first "wins" — the other silently skips it.
`./scripts/run-gtk-pi.sh` sets `PYTHONPATH=src` and uses `.venv/bin/python`, which
should load from source, but stale DB records from the apt version can persist.

If both processes run simultaneously (e.g. apt service auto-started), they race.
Check: `ps aux | grep meshcore`. Disable the apt service if found:
```bash
sudo systemctl stop meshcore-console
sudo systemctl disable meshcore-console
```

### Log export captures full DEBUG output

The app writes DEBUG-level logs to
`~/.local/state/meshcore-uconsole/app.log` (rotating, 1 MB × 3). The UI's "Export
Logs" function reads this file. Absence of a log line from a module is definitive
proof that the code path did not execute.

### pyMC_core event timing

`GroupTextHandler` uses `event_service.publish_sync()` which internally calls
`asyncio.create_task()`. This means `mesh.channel.message.new` arrives
*asynchronously* after the raw `packet` callback. The correlation queue
`_unenriched_grp` (a `deque`) is FIFO and relies on messages arriving in order.
For simultaneous messages this can mis-correlate — not fixed in this session.

### `"changed"` vs `"value-changed"` on `Gtk.Adjustment`

- `"value-changed"` fires when the *scroll position* changes (user scrolling).
- `"changed"` fires when `upper` / `lower` / `page-size` change (content resize).
Connecting auto-scroll to `"changed"` + `GLib.idle_add` is the correct GTK4 pattern
for keeping a list pinned to the bottom as content grows.

---

## GPS on the AIO board (not tested this session)

The app supports the AIO board's GPS via `UConsoleGps` (`platform/gps.py`):
- Serial: `/dev/ttyAMA0` at 9600 baud
- Enable pin: GPIO 27
- Detection: `create_gps_provider()` checks for `/dev/ttyAMA0` before falling back to `NullGps`

To enable:
```bash
# Enable UART hardware
echo 'enable_uart=1' | sudo tee -a /boot/firmware/config.txt
# Disable serial console (so UART is free for GPS)
sudo raspi-config nonint do_serial_hw 0
sudo raspi-config nonint do_serial_cons 1
# Grant serial access
sudo usermod -aG dialout $USER
# Reboot, then verify NMEA data
timeout 5 cat /dev/ttyAMA0   # should print $GNGGA, $GNRMC lines
```

If `/dev/ttyAMA0` exists on startup, `UConsoleGps` is used automatically. Look for
`GPS: opened /dev/ttyAMA0 at 9600 baud` in logs. Satellite fix takes 1–5 minutes
outdoors.

---

## What was NOT done / pending

- **Tests:** `NullGps` has no unit tests. The `cayennelpp` dependency is missing
  from the dev environment, so `uv run pytest` fails entirely. Pre-existing issue.
- **Queue mis-correlation:** If two group messages arrive nearly simultaneously,
  `_unenriched_grp.popleft()` may pair the wrong packet with a message event,
  producing incorrect hop data for one of them.
- **INSERT OR REPLACE for path updates:** Existing DB records with `path_len=0`
  cannot be corrected without deleting the DB. A future improvement could upsert
  path data if the incoming message has better data than the stored record.
- **No PR created.** All work is on `claude/fix-hop-count-display-mUIrw`.

---

## Files changed

| File | Change |
|------|--------|
| `src/meshcore_console/meshcore/client.py` | `send_advert`, `_get_local_telemetry`, `get_device_location` — GPS/settings fallback |
| `src/meshcore_console/platform/gps.py` | Added `NullGps`; changed production fallback |
| `src/meshcore_console/ui_gtk/views/messages.py` | Auto-scroll via `Gtk.Adjustment::changed` |
| `scripts/bootstrap-pi.sh` | Removed `dtparam=spi=on`; added remediation |
| `CHANGELOG.md` | New `Unreleased` section documenting all four fixes |
