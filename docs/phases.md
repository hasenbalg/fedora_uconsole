# Phase plan

A broken screen must never sit on the access path: each phase ends with the
box reachable over SSH before the next one starts.

## Phase 0 - reachable box (no hardware work)
- Download Fedora Server 45 aarch64 raw image (+ CHECKSUM verify).
- Write to SD, resize rootfs.
- Seed: SSH pubkey (into initial user + root via first-boot script or
  cloud-init-less injection), NetworkManager keyfile (WPA-PSK, see
  `config/keyfile.template`), hostname `uconsole`.
- Boot, log in over SSH from LAN. **Acceptance: `ssh` works headless, Wi-Fi joins.**

## Phase 1 - baseline services
- dnf update; enable sshd, cockpit.socket; cockpit self-signed OK on LAN.
- Low-battery shutdown unit (watch `POWER_SUPPLY_CAPACITY` +
  `POWER_SUPPLY_STATUS=Discharging`; threshold e.g. 5% -> `systemctl poweroff`).
- logind HandlePowerKey = poweroff (uses axp20x-pek once Phase 2 overlay lands).
- **Acceptance: Cockpit reachable; battery unit tested with `bat=discharging` simulation.**

## Phase 2 - screen (the only engineering risk)
- Compile retargeted devterm overlays (in `overlays/`) on-device with dtc.
- DKMS: `panel-cwu50` (base: tdeval mainline port) + `ocp8178`.
- config.txt dropin + extlinux cmdline: `fbcon=map:10 fbcon=rotate:1`,
  console=tty1, GPIO9 screen-power line.
- Investigate vendor `vc4_dsi.c` hunk only if panel doesn't bind on mainline vc4.
- **Acceptance: text login on the panel, keyboard logs in locally, rotation correct.**
- Rollback: previous kernel + overlays kept; box still SSHable at every step.

## Phase 3 - power
- Retargeted devterm-pmu overlay live: gauge + AC + power key visible in Fedora.
- Test `systemctl poweroff` -> panel off, power LED off, board actually off.
- If hang at AXP off-write: add shutdown-path i2c-dev write (register + value
  to be measured on-device), keep AXP hardware low-voltage cutoff as failsafe.
- **Acceptance: full-charge -> discharge on screen-on console -> clean auto-off.**

## Phase 4 - console polish
- getty on tty1 autologin-or-login decision; blanking/DPMS timing for battery.
- Fn+brightness listener (consumer-key daemon) - optional.
- Firmware/RTC: vendor devterm-misc includes RTC - confirm mainline driver fit
  or drop.

## Phase 5 - docs & resilience
- README runbook: re-seed after any image rewrite (mandatory), kernel update
  procedure with DKMS rebuild + panel verification before reboot,
  factory-reset path.
- Backup: image of golden SD off-device.
