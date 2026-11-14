# Hardware support notes (verified 2026-11-14)

Method: vendor patch read first, then mainline `of_device_id` check via
git.kernel.org, then Fedora kernel config
(`https://src.fedoraproject.org/rpms/kernel/raw/f45/f/kernel-aarch64-fedora.config`).

## PMU - AXP221 @ i2c0 0x34, INT on GPIO2
Fully mainline. Vendor's axp patches are quirk polish, not the driver:
- MFD: `CONFIG_MFD_AXP20X=y` / `MFD_AXP20X_I2C=y`; `x-powers,axp221` AND
  `axp223` both match (`drivers/mfd/axp20x-i2c.c`).
- Battery gauge: `CONFIG_AXP20X_POWER=m`, compatible
  `x-powers,axp221-battery-power-supply` present in mainline.
- AC: `x-powers,axp221-ac-power-supply` present.
- Power key: `CONFIG_INPUT_AXP20X_PEK=m` -> logind power-key action.
- Soft poweroff: `axp20x_power_off()` registered via
  `devm_register_power_off_handler`; `AXP221_ID` is in the supported switch.
- NOTE the vendor overlay declares `x-powers,axp223` while the child supplies
  use `axp221-*` compatibles; mainline accepts either chip id - decide once in
  our retargeted overlay and note it here.
- Vendor quirk deltas worth porting behaviorally (userspace or patch later):
  VBUS re-plug IRQ kick, PEK press-timing reg, charger-ctrl reg at probe.

## Panel - 5" 720x1280 DSI (JD9365DA-H3 controller, TXW500170B0 assembly)
- `panel-cwu50` driver: NOT in mainline (vendor tree only, downstream panel API).
  Mainline `panel-jadard-jd9365da-h3` does NOT contain uConsole init tables
  (renders garbage per community reports).
- Use tdeval's mainline-API port of `panel-cwu50` as base for DKMS; it also
  covers the Dec-2025 TXW500170B0-BL revision.
- Backlight `ocp8178`: NOT mainline; vendor source is small; DKMS.
- Vendor also patched `vc4_dsi.c` (6.12 downstream) - investigate hunk in
  Phase 2; if the ported panel works against mainline vc4 without it, skip.
- Rotation: `fbcon=rotate:1` (verify direction) - native portrait, mounted landscape.

## Keyboard / trackball
USB HID composite on internal mainboard hub: keyboard (RID 2), mouse (1),
joystick (20), consumer control (3), plus CDC-ACM. Kernel: `usbhid`, works
out of the box. Fn-layer combos arrive as ordinary keycodes/consumer events;
brightness Fn-combos need a small listener (Phase 5, optional).

## Boot / config.txt (baseline from vendor CM4 config.txt)
```
ignore_lcd=1
dtoverlay=vc4-kms-v3d-pi4,cma-384
dtoverlay=devterm-pmu
dtoverlay=devterm-panel-uc
dtoverlay=devterm-misc
dtoverlay=audremap,pins_12_13
dtoverlay=dwc2,dr_mode=host
dtparam=spi=on
gpio=10=ip,np
dtparam=ant2
```
Fedora caveat: boot files live on the vfat `system-boot` partition
(config.txt, overlays/, extlinux/). Vendor overlays use DOWNSTREAM node labels
(`&i2c0if`, `&i2c1`) - retarget for mainline bcm2711 dtbs in `overlays/`.
`ignore_lcd=1` matters because the firmware DSI/LCD path is unused with
upstream vc4; verify semantics on Fedora's firmware blob in Phase 2.

## Expansion / IO
- Base unit: one external USB-A (host, via mainboard hub), USB-C power only
  (data lines absent on CM4 core - see README constraints), no Ethernet,
  no external UART header.
- HackerGadgets AIO v2 / Upgrade Kit would add RJ45 + NVMe; not fitted.

## Power-on sequencing (from vendor pi-gen notes)
- GPIO9 must be driven high at boot or the screen stays off
  (vendor: `gpio=9=op` style at firmware stage; adopt in config dropin).
- Vendor sets default backlight level at boot via service; replicate in Phase 2.
