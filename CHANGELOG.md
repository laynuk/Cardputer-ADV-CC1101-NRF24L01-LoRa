# Changelog — laynuk fork (base: heyajohnny Bruce 1.16, HEAD 3bc53a7)

Changes on top of the heyajohnny 3-in-1 fork. Kept separate so they stay portable
if the fork is ever rebased onto a newer Bruce.

## [pure-jam-mode] — NRF24 jammer: G0 pure-jam toggle

On the Cardputer ADV 3-in-1 module the NRF24 CE/CSN share the keyboard's I2C pins.
During jamming the keyboard polling keeps borrowing those pins, which interrupts the
carrier and slows the hop loop, so a Bluetooth target leaks through in short bursts.
This adds a "pure jam" mode that parks the keyboard and runs the carrier continuously,
matching the effectiveness of a standalone NRF24.

### Added
- **G0 toggles pure-jam** inside the NRF jammer (`nrf_jammer.cpp`). Falling-edge detect
  with a 250 ms debounce; press to enter max-jam, press again to resume the keyboard.
- **Continuous-carrier fast path** in the jammer's SPI loop: when pure-jam is on, the
  GPIO11 / bus-switch-count checks are skipped and the radio just hops channels, so the
  carrier is never dropped.
- **State indicator**: the on-screen hint line shows `PURE JAM | G0 resumes KB` vs
  `Exit:Esc  G0:pure jam`, and the RGB LED turns dim red in pure-jam (`#ifdef HAS_RGB_LED`).
- Two shared flags in `globals.h` / `nrf_common.cpp`:
  - `g_pureJam` — InputHandler suspends keyboard polling while true.
  - `g_jammerOwnsG0` — InputHandler ignores the G0 shoulder button while the jammer owns it.

### Changed
- `boards/m5stack-cardputer/interface.cpp` — `InputHandler`:
  - G0 shoulder-button → Select is gated on `!g_jammerOwnsG0`.
  - the TCA8418 keyboard-poll block returns early when `g_pureJam` (no bus borrow).

### Behaviour notes
- Exit the jammer with **Esc** (keyboard must be active, i.e. pure-jam off). So to leave:
  G0 to resume the keyboard, then Esc.
- Scope is the **jammer only**; spectrum and mousejack are unchanged.
- Jamming on-air is illegal in most regions — intended for tests on your own hardware.

### Not tested
- Built blind (no hardware/PlatformIO in the authoring environment). Compiles are
  brace-balanced and anchored to the 1.16 source, but verify on-device:
  G0 toggle, LED/indicator, continuous carrier vs. a BT target, and clean exit.
