# Session Recap — April 2026 (Session 2)

## Branch

Original: `claude/fix-hop-count-display-mUIrw` — all fixes combined, for local use.

Upstream PR branches (each independently mergeable into `main`):

| Branch | Upstream issue |
|--------|---------------|
| `claude/fix-hop-count-issue52` | #52 |
| `claude/fix-spi-display-issue53` | #53 |
| `claude/fix-message-autoscroll` | — |
| `claude/fix-fallback-position-issue51` | #51 |
| `claude/feat-gps-settings-ui` | — |

---

## Hardware context

Raspberry Pi CM5 Lite, uConsole form factor, Debian Trixie.
HackerGadgets AIO LoRa v1 board (SPI1, GPS on `/dev/ttyAMA0`, no fix acquired this session).
App launched via `./scripts/run-gtk-pi.sh`.

---

## Changes this session

### 1. GPS serial port corrected to `/dev/ttyAMA0`

The AIO v1 board's GPS module is wired to the primary UART (`/dev/ttyAMA0`), not the
mini-UART (`/dev/ttyS0`). Both `UConsoleGps.SERIAL_PORT` and the path check in
`create_gps_provider()` were using `/dev/ttyS0`.

**Fix:** Changed both to `/dev/ttyAMA0`. Commit `a20efe8` on the original branch.

---

### 2. GPS serial device configurable in Hardware settings

**File:** `src/meshcore_console/meshcore/settings.py` — new field `gps_serial_port: str = "/dev/ttyAMA0"`

**Architecture changes:**
- `UConsoleGps.__init__(serial_port: str = "/dev/ttyAMA0")` — stores as `self._serial_port`; `start()` uses it
- `create_gps_provider(serial_port: str = "/dev/ttyAMA0")` — checks `Path(serial_port).exists()`, passes to `UConsoleGps`
- `MeshcoreClient.__init__` now loads settings **before** creating the GPS provider so `self._settings.gps_serial_port` is available
- Settings UI: "GPS Device" text entry row at the bottom of the Hardware panel, saved and reloaded with all other hardware settings

Requires an app restart to apply (same as SPI/GPIO pin settings).

**Branch:** `claude/feat-gps-settings-ui`, commits `a20efe8` + `e00f2d2`

---

### 3. GPS status overlay on the map page

A compact status card appears in the **top-right corner** of the map, on top of the
tile layer. It updates every 2 seconds on the existing GPS poll timer.

```
GPS
Found  Yes      <- green if hardware detected (not NullGps)
Fix    No       <- orange while acquiring, green when fixed
Sats   7        <- satellite count from $GNGGA sentences (UConsoleGps only)
Pos    52.1234, 6.4321  <- lat/lon to 4 dp, or -- if no fix
```

Color coding: green = ok (`@mc_accent`), orange = warning (`@mc_warn`), muted = unavailable.

**Protocol additions required:**

| Symbol | Location | Notes |
|--------|----------|-------|
| `GpsProvider.get_num_satellites() -> int` | `platform/gps.py` | `UConsoleGps` reads `_last_num_sats` from GGA; `GpsdProvider` and `NullGps` return 0 |
| `MeshcoreService.has_gps_hardware() -> bool` | `core/services.py` | True if provider is not `NullGps` |
| `MeshcoreService.get_gps_num_satellites() -> int` | `core/services.py` | Forwards to provider |
| `MockGps.get_num_satellites() -> int` | `mock/gps.py` | Returns 8 when running |

**Branch:** `claude/feat-gps-settings-ui`, commit `0714ffb`

---

## GPS on the AIO v1 board

The app detects GPS automatically at startup via `create_gps_provider()`:

Priority:
1. `MESHCORE_MOCK=1` -> MockGps
2. gpsd reachable -> GpsdProvider
3. Configured serial device (`gps_serial_port` setting) exists -> UConsoleGps
4. Fallback -> NullGps (returns `None`; callers use fixed coordinates from settings)

**`UConsoleGps` specifics:**
- Serial: `/dev/ttyAMA0` at 9600 baud (default; configurable in Settings > Hardware > GPS Device)
- Enable pin: GPIO 27
- NMEA parsing: `$GNGGA` (position + fix quality + sat count), `$GNRMC` (position backup)
- Satellite count exposed via `get_num_satellites()` -- shown in map overlay

**To enable the physical GPS:**
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

Look for `GPS: opened /dev/ttyAMA0 at 9600 baud` in logs. Satellite fix takes 1-5 minutes outdoors.

---

## Branch separation details

The original branch had all fixes squashed together. This session split them into
5 independent branches for upstream PRs. Key decisions:

- **CHANGELOG entries** were written fresh per branch (the original had them all in one commit)
- **Debug commits** `f51b4af` + `7d84c95` (add/remove hop-count tracing) were dropped -- they cancel out
- **`uv.lock`** update `0b1f913` was dropped from all upstream branches (unrelated to fixes)
- **`session-recap.md`** was excluded from all upstream branches
- **`e00f2d2`'s `MockGps->NullGps` test assertion change** belongs in `fix-fallback-position` (where `NullGps` is introduced), NOT in `feat-gps-settings-ui` (which keeps `MockGps` as the fallback on `main`)

No conflicts between branches when merged into `main` in any order -- they touch non-overlapping
functions within the same files (`client.py`, `gps.py`).

---

## Files changed per branch

| File | hop-count | spi | autoscroll | fallback-pos | gps-settings |
|------|:---------:|:---:|:----------:|:------------:|:------------:|
| `scripts/bootstrap-pi.sh` | | v | | | |
| `src/meshcore_console/meshcore/client.py` | v | | | v | v |
| `src/meshcore_console/meshcore/settings.py` | | | | | v |
| `src/meshcore_console/platform/gps.py` | | | | v | v |
| `src/meshcore_console/core/services.py` | | | | | v |
| `src/meshcore_console/mock/client.py` | | | | | v |
| `src/meshcore_console/mock/gps.py` | | | | | v |
| `src/meshcore_console/ui_gtk/views/messages.py` | | | v | | |
| `src/meshcore_console/ui_gtk/views/settings.py` | | | | | v |
| `src/meshcore_console/ui_gtk/views/map.py` | | | | | v |
| `src/meshcore_console/ui_gtk/resources/app.css` | | | | | v |
| `tests/unit/test_gps.py` | | | | v | v |
| `CHANGELOG.md` | v | v | v | v | |

---

## What was NOT done / pending

- **GPS satellite count for `GpsdProvider`:** TPV records from gpsd do not include sat count.
  SKY records do -- adding a SKY-stream parser to `GpsdProvider` would expose this.
- **Queue mis-correlation in `_unenriched_grp`:** If two group messages arrive nearly
  simultaneously, `popleft()` may pair the wrong packet with a message event. Still unresolved.
- **`INSERT OR REPLACE` for path data:** Existing DB records with `path_len=0` cannot be
  corrected without deleting the DB.
- **No unit tests for `NullGps`** or the new GPS callsite fallback logic in `client.py`.
- **No PRs created.** All branches pushed; PRs must be opened manually against upstream.
