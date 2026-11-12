# Changelog — laynuk fork (base: heyajohnny Bruce 1.16, HEAD 3bc53a7)

Changes on top of the heyajohnny 3-in-1 fork. Kept separate so they stay portable
if the fork is ever rebased onto a newer Bruce.

---

## v1.16-profiles — Hardware profiles + ADV keyboard crash fix  ✅ tested on hardware, released

Release: https://github.com/laynuk/Cardputer-ADV-CC1101-NRF24L01-LoRa/releases/tag/v1.16-profiles

### NRF24 jammer: G0 pure-jam toggle
On the Cardputer ADV 3-in-1 module the NRF24 CE/CSN share the keyboard's I2C pins, so during
jamming the keyboard polling keeps borrowing those pins and interrupting the carrier. "Pure jam"
parks the keyboard and runs the carrier continuously.
- **G0 toggles pure-jam** in the jammer (falling edge, 250 ms debounce). Press to enter max-jam,
  press again to resume the keyboard. Exit the jammer with Esc (keyboard active).
- Continuous-carrier fast path: in pure-jam the GPIO11 / bus-switch checks are skipped.
- Indicator: on-screen hint + dim-red RGB LED (`#ifdef HAS_RGB_LED`).
- Flags `g_pureJam` / `g_jammerOwnsG0` (globals.h / nrf_common.cpp).
- Files: interface.cpp, nrf_jammer.cpp, nrf_common.cpp, globals.h.

### Hardware profiles (Standard ↔ ADV 3-in-1)
One build can switch between the 3-in-1 cap and a normally-wired module instead of hardcoding the
3-in-1 wiring.
- `bruceConfig.hwProfile` (0 = Standard, 1 = ADV 3-in-1). **Default 1** so existing users keep
  working after the update. Persisted in bruce.conf. Switching applies on an immediate restart.
- All 3-in-1-specific behaviour is gated on the profile: the clone pinout + G8/G9 parking in
  `_post_setup_gpio`; the I2C bus-sharing "dance" in `InputHandler`; codec (ES8311) access; and
  G0/pure-jam in `nrf_jammer` (Standard: G0 is a normal button, jammer runs stock).
- **Device Pins moved** out of Dev Mode to `System Config → Advanced → Device Pins`, with a
  profile selector (as in Bruce 1.16.1).

### Per-profile pins — one file per profile
- Each profile keeps its **own pins file**: `/brucePins_adv3in1.conf` (3-in-1) and the historic
  `/brucePins.conf` (Standard), so switching never clobbers the other profile's pins.
- The 3-in-1 clone pinout is the editable default of the 3-in-1 profile, seeded once on a fresh
  profile (`createFile`), then taken from the file.
- **Migration**: the legacy single `/brucePins.conf` is renamed to `/brucePins_adv3in1.conf` on
  first boot (LittleFS and SD).
- **Reset pins** (per profile) in Device Pins — lighter than a factory reset. Factory reset backs
  up and clears both profiles' files.
- Files: config.h/.cpp, configPins.h/.cpp, interface.cpp, nrf_jammer.cpp, ConfigMenu.h/.cpp.

### Pin persistence + SD (fix #5)
- **Pins read from LittleFS** — the same filesystem they are written to. Previously reads could
  come from a stale SD copy when a card was inserted, so edits appeared not to persist.
- **SD mirror**: on every save the active profile's pins file is mirrored to SD (via
  `setupSdCard()` + `copyToFs`, like `/bruce.conf`) as an inspectable backup. LittleFS stays the
  source of truth; a plain `sdcardMounted` check was not enough (the shared bus can leave it false),
  so the mirror actively (re)mounts the card.
- **Import pins from SD** (Device Pins) — pull a hand-edited pins file from the card on demand
  (SD → LittleFS, then restart). The SD copy is never read automatically at boot.

### Crash fixes (not 3-in-1 specific — affect any ADV user)
- **Stack-canary crash on the first boot after a profile/pin change** (keyboard init). The TCA8418
  INT pin installed a GPIO ISR, whose `esp_intr_alloc()` ran on the ~1 KB ESP-IDF IPC task and
  intermittently overflowed it. The keyboard is already serviced by polling (`taskInputHandler`),
  so the ISR was removed. Also removes the recurring "Forcing keyboard interrupt" churn.
- **Stack-canary reboot when saving device pins.** The deeply nested Device Pins menu plus the SD
  copy overflowed loopTask's default 8 KB stack (corrupted backtrace on save). loopTask now gets
  16 KB via `SET_LOOP_TASK_STACK_SIZE` (main.cpp).

### Notes
- The `-1` (pin-disable) behaviour is kept as upstream Bruce: an SPI bus whose `sck` is `-1`
  reverts to the compiled default, because an SPI peripheral has no clock without it. To silence a
  radio, use its module-select setting rather than nulling its pins.
