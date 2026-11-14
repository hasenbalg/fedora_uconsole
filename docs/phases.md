# Phase plan

A broken screen must never sit on the access path: each phase ends with the
box reachable over SSH before the next one starts.

## Phase 0 - reachable box (no card reader anywhere in the workflow)
Constraint: the management host has no SD reader. uConsole boots its own media.
- Download Fedora Server 45 aarch64 raw image (+ CHECKSUM verify) on mgmt host.
- Write image to a **USB flash drive** on mgmt host; seed the drive's
  filesystems (SSH pubkey + NetworkManager keyfile from
  `config/keyfile.template`, hostname `uconsole`) via loop-mount; resize rootfs.
- Boot uConsole from the stick with SD slot EMPTY (firmware default order
  SD->USB; confirmed by community on uConsole CM4, incl. CM5). If the external
  USB-A port shows no VBUS at firmware stage (one report), move the stick to
  the internal USB port (proven by user reports booting SSD there).
- SSH over seeded Wi-Fi. From inside the box, download the same seeded image
  and `dd` to `/dev/mmcblk0` (the uConsole's own SD slot; NOTE: if the CM4 has
  eMMC, the SD slot is hard-disabled and `/dev/mmcblk0` IS the eMMC - confirm
  which target before writing).
- Reboot from internal media. Keep the stick as rescue boot medium.
- **Acceptance: `ssh` works headless from Wi-Fi on internal media; stick-free
  boot reproducible.**

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
