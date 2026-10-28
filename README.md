# CanSat 2027 GCS — Team 1059 (web)

Browser dashboard for the Team 1059 ground station. It is the web version of
`GCS2027.py` (Tkinter). The radio, parsing, simulation and logging logic is the
same code. Only the UI moved to the browser.

## Run

```bash
cd ~/Downloads/gcs_web
python3 -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
python main.py --virtual
```

Open <http://127.0.0.1:8000>.

`--virtual` needs no hardware: it starts the loopback virtual CanSat. **Nothing
streams at startup** (with or without `--virtual`). Telemetry is shown only after
the operator runs the CMD-uplink sequence:

| Path | Commands | What happens |
|---|---|---|
| Simulation | **SIM → SIM ENABLE → SIM ACTIVATE** | SIM ENABLE: SOURCE = FILE and no profile loaded → the file picker opens. SOURCE = LINK → the sidebar and log say whether the CanSat is connected; not connected = SIM ENABLE is refused; connected but no profile = the file picker opens. SIM ACTIVATE: telemetry display on, and the SIMP stream starts by itself once the payload is armed (LINK: once `MODE=S` is seen) and a profile is loaded |
| Flight | **SIM → FLIGHT ENABLE → FLIGHT ACTIVATE** | FLIGHT ENABLE sets the mode badge (stops the virtual payload). FLIGHT ACTIVATE turns the real-telemetry display on. Neither is sent on air (SPEC has no FLIGHT command) |
| Leave | SIM DISABLE / FLIGHT DISABLE | display off; start again with SIM |

Out-of-order presses are refused with a message. While the display is off, real
packets still count in LINKS / RADIO (RAW RX), still acknowledge commands and
still prove SIM arming, but are not shown, plotted or written to CSV (one log line
says so).

Other commands:

* **ST** opens a dialog: **GPS** takes the newest `GNSS_TIME` the CanSat sent
  (PocketQube telemetry) and sends it as `ST,HH:MM:SS`; **Manual** sends the time
  you type. After it goes out, the header shows `GPS TIME hh:mm:ss UTC [ST GPS|MANUAL]`,
  ticking; with GPS it re-syncs to every new `GNSS_TIME`. Typed: `ST GPS`, `ST 12:30:00`.
* **CAM_SET** button = rotate the camera 10° clockwise per press (`MEC,ROT,10`);
  the FSW SEQUENCE **Cam rot** button does the same. Typed `CAM_SET,5` rotates 5°
  (whole degrees, ±360). The running total is in the log and the FSW SEQUENCE row.
* Commands that exist on one vehicle only (CAL, ARM, SOLAR_*, BOOM_*, CAM_SET, MEC
  RELEASE …) always go to that vehicle on the selected radio, so their buttons
  are never greyed out by the UPLINK target. Still one link per command.
* CX ON / CX OFF buttons were removed (typing `CMD,1059C,CX,ON` still works).

| Flag | Effect |
|---|---|
| `--virtual` (alias `--sim`) | loopback virtual CanSat; nothing streams until SIM → SIM ENABLE → SIM ACTIVATE |
| `--auto-arm` | with `--virtual`: load the demo profile and run SIM → SIM ENABLE → SIM ACTIVATE by script (old `--virtual` behaviour) |
| `--manual-arm` | no effect any more (manual is the default); accepted so old command lines still run |
| `--sim-speed N` | simulator speed multiplier |
| `--profile FILE` | load a pressure profile at startup (overrides the demo profile) |
| `--log` | start GS13 CSV logging at startup |
| `--config FILE` | radio config (default `radio_config.json` in this folder) |
| `--host`, `--port` | bind address (default `127.0.0.1:8000`) |

With real radios, run `python main.py`. Then open **Links / Radio → Radio
configuration**, set the ports and press Connect.

### Same screen as the Tkinter GCS

The page follows `GCS2027.py` panel for panel:

* **Sidebar** (366 px): TEAM / connection state, Connect all / Disconnect, the
  `MODE: …  [FILE|LINK]` badge, SIM SOURCE, UPLOAD CSV + Start / Pause / Stop,
  FSW SEQUENCE (collapsible) + Release PQ / Boom / Cam rot, SERIAL PORTS
  (collapsible), UPLINK radio + target, COMMANDS (Setup / Flight / Mech / Sim),
  and at the bottom the CMD ECHO event log and the CMD entry.
* **Header**: GPS TIME, GS11 counts, Merged / Xbee / Espnow.
* **Tabs**: CONTAINER (state machine, 4 plots, GS3, live packet table),
  POCKETQUBE (state machine, GS4 plots in 3 sub-tabs, GS5, IMAGE_STAB / SCIENCE_EXP),
  ALTITUDE (apogee + 90 % lines, PQ timer), MAP, LINKS / RADIO, COMMANDS (ACK,
  CSV buttons, two telemetry logs with the measured Hz).
* Double-click a tab to open it in its own window (Tk "tear-off"); close that
  window or press Dock Back to return it. Ctrl+L toggles CSV logging, Ctrl+R
  refreshes the serial ports, as in the Tk app.
* `MODE: NaN` is the Tk app's own text for "no mode chosen yet" (`ui_mode`
  UNSELECTED). It then shows SIM, SIM ENABLED, SIMULATION, FLIGHT ENABLED or FLIGHT.

### Works offline

Chart.js 4.4.1 and Leaflet 1.9.4 are vendored in `static/vendor/`, and the page
uses system fonts (Helvetica / Menlo, like the Tk app). Nothing loads from the
internet except **MAP: ONLINE** satellite tiles. For the field, press
**CACHE MAP** while online. Tiles go into `map_cache.db` next to `gcs_core.py`
(same tables as tkintermapview, so a cache made by the Tk GCS can be copied in).
Then switch to **MAP: OFFLINE**, which serves tiles from `GET /tiles/{z}/{x}/{y}`.

If the header says **SERVER OFFLINE — retrying**, the page is open but
`python main.py` is not running. Start it and the page reconnects by itself.

## Files

| File | What it is |
|---|---|
| `main.py` | FastAPI app: `GET /`, `/static`, `GET /api/log/{C,P}.csv`, `GET /tiles/{z}/{x}/{y}`, WebSocket `/ws`, CLI |
| `gcs_core.py` | Constants and logic from `GCS2027.py`, plus `GroundStation` (the non-UI half of the old `GCSApp`) |
| `templates/index.html` | Page layout and CSS |
| `static/app.js` | WebSocket client, Tk-equivalent panels, plots (TracePlot / AltitudePlot ports), map, tear-off windows |
| `static/vendor/` | Chart.js 4.4.1, Leaflet 1.9.4 (offline copies) |
| `profiles/sim_profile_1059.csv` | Demo SIMP profile (207 rows, 1 Hz) |

CSV logs keep the original names and location: `Flight_1059C.csv` /
`Flight_1059P.csv`, or `SIM_Flight_*.csv` while the virtual payload runs. They
are written next to `gcs_core.py`.

## WebSocket `/ws`

### Server → browser

| `type` | When | Payload |
|---|---|---|
| `config` | on connect | field lists (`container_fields`, `pq_fields`, `pq_trailing`, `pq_log_fields`), `command_groups`, states and colours, `link_keys` and link config, `radios`, display/sim sources, `mech_bits`, theme, `chart_points`, `log_max_lines`, ports |
| `snapshot` | on connect and after a display-source change | `display_source`, `rows` (`{C: [...], P: [...]}` recent telemetry), `history` (plot history per vehicle, `history_fields` order), `alt_series`, `track` (map), `events`, `status` |
| `telemetry` | **every accepted packet, immediately** | `vehicle`, `link`, `sources` (display sources that accepted this copy), `virtual`, `recv`, `checksum`, `packet_count`, `text` (Tk TELEMETRY LOG line), `values` (packet table), `display` (GS3/GS5 text with units), `extra`, `num` (one history row), `alt_pt` |
| `status` | every 0.25 s and after each browser action | link state, rx/dup/chk/age/first, staleness per link, GS11 counts, measured Hz, GPS, vehicle states, MECH bits, apogee, PQ timer, FSW sequence, uplink, ACK text, pending command, SIM block, CSV logging, validation warnings |
| `log` | event-log line | `t`, `tag` (sent/ack/state/alert/warn/err/info), `msg` |
| `cmd` | uplink | `event`: `sent` (with `wires`, `link`, `virtual`), `not_sent`, `ack` (`command_count`, `cmd_echo`), `noack` |
| `tx` | bytes written to a serial port | `link`, `wire` |
| `alert` | replaces the Tk message boxes | `level`, `title`, `text` |
| `confirm` | kPa/hPa profile needs conversion | `id: "profile_convert"`, `title`, `text` |
| `request_profile` | SIM ENABLE / ACTIVATE with no profile loaded | `text`, `source` |
| `reset` | mission reset | — |
| `ports` | serial port list | `ports` |
| `map_cache` | CACHE MAP progress | `running`, `stat`, `value`, `max` |

### Browser → server

| `type` | Fields | Does |
|---|---|---|
| `command` | `name`, `arg?`, `vehicle?` | a COMMAND_GROUPS button. The browser handles the argument prompt and the destructive confirmation, the same way `_send_spec` did |
| `raw` | `text`, `vehicle?` | the text box (rules below) |
| `uplink` | `radio?`, `target?` | uplink radio / target vehicle |
| `display_source` | `value`: MERGED / XBEE / ESPNOW | |
| `link` | `action`: connect / disconnect / connect_all / disconnect_all / refresh_ports / validate / save_config, `key?`, `cfg?` (`{LINK: {port, baud, channel, ident}}`) | |
| `virtual` | `action`: start / stop | kept for scripts; no button, as in the Tk app (`--virtual` or the first SIM ENABLE starts the loopback) |
| `sim_source` | `value`: FILE / LINK | |
| `profile` | `name`, `text` | upload a pressure profile |
| `profile_convert` | `accept` | answer to `confirm` |
| `simp` | `action`: start / pause / stop | |
| `csv_log` | `on` | GS13 logging |
| `seq` | `action`: release / boom / cam, `arg?` | FSW SEQUENCE buttons |
| `map_mode` | — | MAP: ONLINE ↔ OFFLINE |
| `map_cache` | `action`: start (`lat`, `lon`, `radius`, `z0`, `z1`) / cancel | CACHE MAP |

### Raw text rules

1. `CMD,...` is sent exactly as typed, as in the original.
2. If the text starts with a command name from `COMMAND_GROUPS` (local FLIGHT …
   included), it does exactly what the button does: same mode sequence, same
   target rules, ACK tracking. Matching ignores case and treats commas and spaces
   the same, the longest name wins, and a leading `MAC` is read as `MEC`.
   Examples: `SIM ENABLE`, `FLIGHT ACTIVATE`, `cam_set,5` → `CMD,1059P,MEC,ROT,5`,
   `ST GPS`, `MAC BOOM,ON` → `CMD,1059x,MEC,BOOM,ON`. `SIMP 101325` sends that one
   value. `CAM_SET` with no value = one 10° step; `MEC` and `ST` need a value.
3. Anything else uses the original free-text path (`wire_command(text)`).
