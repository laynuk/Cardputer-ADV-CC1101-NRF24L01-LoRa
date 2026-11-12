# Changelog — laynuk fork (base: heyajohnny Bruce 1.16, HEAD 3bc53a7)

Changes on top of the heyajohnny 3-in-1 fork. Kept separate so they stay portable
if the fork is ever rebased onto a newer Bruce.

---

## [pure-jam-mode] — NRF24 jammer: G0 pure-jam toggle  ✅ tested on hardware

On the Cardputer ADV 3-in-1 module the NRF24 CE/CSN share the keyboard's I2C pins.
During jamming the keyboard polling keeps borrowing those pins, interrupting the carrier,
so a Bluetooth target leaks through. This adds a "pure jam" mode that parks the keyboard and
runs the carrier continuously.

### Added
- **G0 toggles pure-jam** in the jammer (falling edge, 250 ms debounce). Press to enter
  max-jam, press again to resume the keyboard. Exit the jammer with Esc (keyboard active).
- **Continuous-carrier fast path**: in pure-jam the GPIO11 / bus-switch checks are skipped.
- **Indicator**: on-screen hint + dim-red RGB LED (`#ifdef HAS_RGB_LED`).
- Flags `g_pureJam` / `g_jammerOwnsG0` (globals.h / nrf_common.cpp).

Files: interface.cpp, nrf_jammer.cpp, nrf_common.cpp, globals.h.

---

## [hw-profiles] — Hardware profiles (Standard ↔ ADV 3-in-1)  ⚠️ NOT yet tested

Lets one build switch between the 3-in-1 cap and a normally-wired module, instead of the
3-in-1 wiring being hardcoded. Delivered as two commits so Stage 2 can be reverted alone.

### Stage 1 — profile flag, menu, behaviour gating
- `bruceConfig.hwProfile` (0 = Standard, 1 = ADV 3-in-1). **Default 1** so existing users keep
  working after the update. Persisted in bruce.conf.
- All 3-in-1-specific behaviour is gated on the profile:
  - `_post_setup_gpio`: the clone pinout + G8/G9 parking only in 3-in-1.
  - `InputHandler`: the I2C bus-sharing "dance" only in 3-in-1; Standard reads the keyboard
    plainly (Wire1 stays I2C).
  - codec (ES8311) access: dance only in 3-in-1.
  - `nrf_jammer`: G0/pure-jam only in 3-in-1; in Standard G0 stays a normal button and the
    jammer runs stock.
- **Menu moved**: pin config left the Dev menu — now under `System Config → Advanced →
  Device Pins`, with a `HW Profile` selector. Switching a profile prompts a restart (pins are
  applied at boot).

### Stage 2 — per-profile editable pins
- Pins stored per profile, keyed `"<MAC>#<profile>"`, so switching never clobbers the other
  profile's pins.
- The 3-in-1 clone pinout becomes the **editable default** of profile 1 (applied once on a
  fresh profile, then taken from brucePins.conf). Standard defaults to unset — you configure
  it for your module.
- **Migration**: legacy flat pin entries (pre-profile) are moved into profile 1 on first load.
- **Reset pins to default** (per profile) in the Device Pins menu — lighter recovery than a
  full factory reset. Factory reset still clears both profiles.

Files: config.h/.cpp, configPins.h/.cpp, interface.cpp, nrf_jammer.cpp, ConfigMenu.h/.cpp.

### Test carefully (built blind — no PlatformIO/hardware in the authoring environment)
- **3-in-1 profile (default):** everything works exactly as before (keyboard, CC1101, LoRa,
  NRF scanner, G0 pure-jam). Existing pins survive the update (migration).
- **Standard profile:** switch in the menu → restart → the keyboard still works, G0 is a
  normal button, and a module on its own pins (set via Device Pins) works.
- **Per-profile pins:** edit pins in one profile, switch, switch back — each keeps its own.
- **Reset pins to default** and **Factory reset** behave as described.
- Highest-risk spots to watch: the ArduinoJson migration copy (`obj[p1].set(obj[mac])`), the
  Standard-profile keyboard read path, and the `needPinDefaults` first-boot populate.
