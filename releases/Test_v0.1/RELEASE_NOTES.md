# Test_v0.1

**Source:** Startec-Dynamics/Athena-FW
**Date:** 2026-08-01

## Changes


| Area | Type | Change |
|---|---|---|
| **Watchdog & Fault Recovery** | Bug Fix | Independent watchdog was mis-configured — prescaler `/32` clamped a requested 45 s reload to the 12-bit maximum, giving an *effective* window of only ~4.1 s. Corrected to prescaler `/256` with a 30 s target reload (3750, below the clamp) → genuine **~30.0 s** window |
| | New Feature | 1 Hz timer (`wdgIgn`) refreshes the watchdog unconditionally while ignition is ON, so a rider is never reset mid-ride by a network stall. **Trade-off:** per-task heartbeat supervision is effectively disabled for the whole ignition-ON period |
| | Behavior Change | `ENABLE_CRASH_DUMP` is now `0` — fault handlers become `while(1)` instead of capturing a crash dump to RAM2/W25Q; recovery now depends entirely on the IWDG expiring |
| | Bug Fix | IWDG reset-cause flag was previously cleared in `main()` before it could be logged, hiding watchdog resets from the boot log. Clear removed — IWDG resets now correctly reported |
| **Memory Safety — AT/MQTT Parsing** | Bug Fix | `delimiterPos` was `uint8_t` over a 2048-byte receive buffer — any delimiter past byte 255 wrapped and produced wrong parse offsets (routinely hit by `+QMTRECV`). Widened to `uint16_t` |
| | Bug Fix | Three receive-fill paths (`modem_at_command` char/line mode, `qbGetLine`) had no bounds check against buffer/delimiter-array size, allowing RAM overwrites on long AT responses. New bounded append helper (`modem_append_recv_char`) enforces both limits |
| | New Feature | Structured `+QMTRECV` parser (`parse_qmtrecv_response`) replaces five copies of ad-hoc `strstr`/`strchr`/`memcpy` extraction, with explicit bounds on topic (150 B) and payload (2048 B) |
| | Behavior Change | Topic dispatch now matches the **exact topic suffix** instead of substring-anywhere search on the raw AT buffer — a payload that merely mentions another topic name can no longer misfire the wrong handler |
| | Bug Fix | `jsoneq()` matched key prefixes (`"lock"` matched `"lockdown"`) with no NULL guard; now requires exact-length match plus NULL/type guards |
| | Bug Fix | `humanTime2Epoch()` dereferenced `delimiterPos[0]` and a `strchr()` result without checking either — both now guarded |
| | Bug Fix | `modemReadPubSem` leaked on the MQTT-payload-truncation error path, permanently deadlocking the modem task after one truncated message. Semaphore now released on every exit path |
| **Modem & Connectivity** | New Feature | Bounded ignition-off offline modem policy: after 120 s without MQTT (no BLE, no active alert), the unit sleeps instead of retrying indefinitely; wakes for up to three 2-minute windows (1 h, then every 2 h), then stops until ignition or BLE wakes it |
| | Behavior Change | Modem init/network failure with ignition OFF previously called `HAL_NVIC_SystemReset()` after repeated failures — a dead-battery risk with no SIM coverage. Now rewinds the state machine and enters the offline sleep policy instead. Ignition ON still power-cycles the modem (unchanged) |
| | New Feature | MQTT credential validation and self-healing — writes are magic-stamped, NUL-terminated, ID-validated, and **read back and re-validated**. Corrupted credentials now trigger automatic re-provisioning instead of failing indefinitely |
| | Behavior Change | Provisioning success no longer reboots the MCU — publishes a status message and continues in place, closing the connection properly |
| | Bug Fix | `lineMode` left set after line-mode MQTT reads stripped the leading `+` from subsequent AT responses (e.g. `+QENG`/`+QSPN`). `MODEM_COMMAND` now clears `lineMode` after every call |
| | Bug Fix | `parse_qspn` advanced a fixed 6 characters after `"+QSPN:"`; now also accepts the stripped `"QSPN:"` form and locates the colon directly |
| | Behavior Change | Non-LTE cell registration now publishes partial/minimal cell info instead of aborting the publish entirely |
| | Bug Fix | Eleven raw digit-extraction sites replaced with a bounds-checked helper (`recv_digit_after_delim`) that validates the character is a digit before use |
| | Bug Fix | Three network-status macros (`NWREGROAM_STR`, `NWREGFAILED_STR`, `NWREGDENIED_STR`) were multi-character constants instead of string literals; corrected |
| **BLE Authorization & Anti-Theft** | Behavior Change (Security) | BLE unlock now requires a **secure** connection confirmed by the `BLE_INT` pin, not just a connection — an unauthenticated peer can no longer unlock. Rejected unlock forces and persists the lock state |
| | New Feature | Ride-authorization latches (`ble_authorized_for_ignition`, `ride_authorized`) — a rider authenticated before/at ignition-on stays authorized for the full ignition cycle even if BLE drops mid-ride |
| | Behavior Change | BLE secure-connection loss now debounced over 5 timer samples instead of acting immediately, reducing relay chatter from transient drops |
| | Behavior Change | BLE auto-shutoff now locks the bike only when connected **and not secure** (previously locked on any connection when ignition was off) |
| | Behavior Change | Theft evaluation now requires a 3 s lock-settle window before arming, plus 4 consecutive debounced-ignition samples over 2 s to confirm a hotwire event |
| | Behavior Change | Theft logic's ignition input now comes from a debounced value (`bike_state.debounced_ignition`) instead of a raw pin read |
| | Bug Fix | `ign_state` in the BLE event callback sat in a `switch` block where its initializer never ran, so lock/unlock decisions were made on an indeterminate value. Hoisted above the `switch` |
| | New Feature | Movement-theft detection now validates both current and baseline GPS coordinates (`theft_location_valid`) before triggering — removes false trips from partial fixes |
| **Alert Handling (Theft / Accident)** | New Feature | Ignition-off alert with no MQTT and no BLE peer for 120 s is now automatically cancelled and all alert state cleared |
| | New Feature | In-progress accident alert is cancelled (bike unlocked, subject to engine-disable) if righted before MQTT confirmation; wait-loop period dropped 5000 ms → 500 ms with active IMU polling to actually detect this |
| | New Feature | Theft alert is cancelled if a secure BLE connection appears while waiting for MQTT confirmation |
| | Bug Fix | Stale alert-timer notifications now dropped; an in-progress countdown aborts if the underlying alert flag was externally cleared |
| **GPS** | New Feature | GPS UART reception moved from per-byte RXNE interrupt capture to DMA1 Channel 5 circular reception with sentence-framed parsing |
| | New Feature | NMEA parsing rewritten — checksum validated **before** any field is read, fields tokenised in place, coordinates range-checked (±90°/±180°) and hemisphere-adjusted |
| | Behavior Change | Unused HDT/HDM/DPT/MTW (compass/sounder) decoding removed — verified no callers anywhere in the application |
| | Behavior Change | GPS restart interval extended 20 s → 90 s, with a soft reset attempted first if bytes arrived in the last 10 s |
| | Behavior Change | Ride location now saved on ignition ON→OFF (rides ≥ 20 s) instead of a movement-duration heuristic in the GPS task |
| | Bug Fix | Coordinate validation previously range-checked ±180° on **both** axes; latitude now correctly checked against ±90° |
| **Persistent Storage (NVM)** | New Feature | Persistent bike-state store — lock/unlock, EDS, theft-detect, accident-detect, BLE-enable journalled to two W25Q sectors (A/B, CRC-checked, sequence-numbered) and restored at boot; only restored when engine-disable is not set |
| | New Feature | All W25Q access serialized behind a recursive mutex (`w25q_lock`) that wakes the device and clears block-protect bits before each operation |
| | Bug Fix | `W25Q_WriteEnable()` previously asserted success without checking the WEL bit; now reads SR1 back and errors if the state wasn't reached |
| | Bug Fix | `W25Q_EraseSector()` previously returned before the erase completed, risking a race with a following write; now polls until busy clears |
| | Bug Fix | OTA metadata written with an incorrect double-word count (`sizeof/8 + sizeof%8`); corrected to `(sizeof+7)/8` with zero-padding |
| | New Feature | OTA metadata now carries a magic value, validated on read (erased-sector detection, range checks); records without the magic still accepted with a "legacy" warning for compatibility |
| **IMU** | New Feature | 6-hour maintenance reset re-initializes the IMU while parked and still, recovering DMP quaternion drift; both this and the existing 30 s stall-recovery restart enter a 30 s warm-up state suppressing movement/accident/theft checks |
| | Bug Fix | `i2c_scan()` never reset its found-flag at entry and kept scanning after the first match, risking a stale or wrong device address on re-scan. Now resets the flag and breaks on first match |
| **Power Management** | Behavior Change | Sleep ladder unified between test/production configs: ignition-off stage 3 210 s → 360 s; light→deep sleep entry unified at 36 h; deep-sleep hard-reset backstop unified at 72 h |
| | System Impact | Offline modem policy expected to substantially reduce parked power draw out of coverage — **not yet bench-measured**; a 72 h parked current profile is required before sign-off |
| **Code Size / Formatted Output** | New Feature | `printf`/`snprintf`/`sprintf` now backed by a minimal in-tree implementation; newlib float printf/scanf link options disabled for Debug config, reducing flash usage |
| | Known Limitation | `%e`/`%g` unimplemented (literal character emitted); `%f` capped at 6 decimal places; `vsnprintf`/`vsprintf`/`vprintf` not overridden and would lack float support if ever called (no current caller) |
| **Build / Flash Footprint** | Behavior Change | Application flash region reduced 206 KB → 198 KB (linker); `CONFIG_MAX_FW_SIZE` reduced 210 KB → 198 KB to match — larger OTA images now rejected |
| | **Critical Flag** | Shipped image measures 202,672 bytes against a 202,752-byte ceiling — **80 bytes of headroom remain.** Single largest release risk, independent of any functional change |


## Artifacts

- SI_ATH_FW_v0.8.8.9.bin
- SI_ATH_FW_v0.8.8.9.hex
