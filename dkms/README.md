# Out-of-tree drivers (DKMS)

- `panel-cwu50/`: base = https://github.com/tdeval/uconsole-radxa-cm5-mainline `kernel/panel-cwu50.c` (mainline panel API; covers TXW500170B0 and -BL). Adapt Kconfig/Makefile for DKMS.
- `ocp8178/`: backlight, port from vendor patch (`drivers/video/backlight/ocp8178_bl.c`); small, GPIO-ish backlight driver.
Both target stock Fedora kernel headers (`kernel-devel`).
