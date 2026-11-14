# fedora_uconsole

Fedora 45 Server (aarch64, headless) on a ClockworkPi uConsole with the
Raspberry Pi CM4 core. No GUI: text login (fbcon) on the built-in 5" panel,
USB keyboard, genuine battery operation, SSH + Cockpit for management.

Status: **planning seeded, no hardware bringup yet.** See `docs/phases.md`.

## Requirements (fixed, grilled)
- Fedora **45 Server aarch64** (>= 45-1.4 image), rpi4 target, **stock Fedora kernel**.
- Built-in 720x1280 (portrait, mounted landscape) screen shows a **text login**;
  built-in keyboard works for local login.
- **Battery is a hard requirement**: capacity reporting + clean poweroff before
  cells die.
- Remote management: SSH, Cockpit. **No Tailscale** (owner ruling).
- First boot is pre-seeded SD **only** (see Constraints).

## Architecture decisions
| Area | Decision | Basis |
|---|---|---|
| Kernel | Stock Fedora kernel, NOT ClockworkPi's patched 6.12 | verified in-tree AXP221 support; see hardware-notes |
| Battery/PMU | In-tree: axp20x MFD + battery + AC + PEK power-key + poweroff handler | all mainline and enabled in Fedora kernel config |
| Panel | Out-of-tree DKMS: `panel-cwu50` (mainline-API port, JD9365DA ctrl) | not mainlined; precedent port exists for mainline 7.x |
| Backlight | Out-of-tree DKMS: `ocp8178` | not mainlined |
| Overlays | Clockwork `devterm-*` overlays **retargeted to mainline DT node labels** | vendor overlays target downstream labels |
| Console | fbcon + `fbcon=rotate` (panel native portrait) | standard KMS rotation |
| Keyboard | Zero work: USB HID composite (STM32, VID 1EAF) on internal hub | confirmed by others' dmesg on stock-style kernels |
| First boot | Pre-seed card: SSH pubkey + NetworkManager WPA-PSK keyfile | no second door exists |
| Low battery | custom systemd unit watching `/sys/class/power_supply/`, AXP hw cutoff = failsafe | headless Fedora has no upower by default |

## Constraints on record (base unit, CM4 core)
- **g_ether over USB-C is physically impossible**: USB-C data lines (USB0_DP/DM)
  do not exist on the CM4 SODIMM; USB_OTGID strapped to GND via 1K.
- **No Ethernet** (no expansion board fitted), **no external serial**. The
  keyboard's rear UART port flashes the keyboard's STM32 - it is not a Pi console.
- Consequence: a card that is not seeded cannot be reached at first boot.
  **Every rewrite of the card requires re-seeding.**
- Poweroff risk: mainline `i2c-bcm2835` lacks atomic I2C xfer; ClockworkPi still
  ships a candidate patch for shutdown hangs on the same PMU. If `poweroff`
  stalls at the AXP off-write, fallback is an `i2c-dev` register write in the
  shutdown path. To be tested in Phase 3.
- You own DKMS rebuilds on every Fedora kernel update; keep the previous
  kernel as boot fallback until the panel comes up.

## Layout
- `config/` - boot config dropins and seed templates (no real secrets here)
- `docs/`   - hardware evidence notes and the phase plan
- `overlays/` - retargeted devterm device-tree overlays (Phase 2+)
- `dkms/`   - panel-cwu50 and ocp8178 out-of-tree drivers (Phase 2)

## Upstream references
- ClockworkPi hardware: https://github.com/clockworkpi/uConsole
  (`Code/patch/cm4/20230630/`: config.txt + kernel patch; the patch contains
  overlay DTS sources and all downstream driver deltas)
- Mainline-API panel port precedent: https://github.com/tdeval/uconsole-radxa-cm5-mainline
- Fedora on RPi: https://docs.fedoraproject.org/en-US/quick-docs/raspberry-pi/
- Community CM4 config.txt discussion: https://forum.clockworkpi.com/t/raspberry-pis-config-txt/19360
