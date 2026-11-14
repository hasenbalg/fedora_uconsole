# Retargeted devterm overlays

Phase 2. Start from the DTS embedded in
`clockworkpi/uConsole:Code/patch/cm4/20230630/0001-patch-cm4.patch`,
retarget `&i2c0if`-style downstream labels to mainline bcm2711 nodes,
build with `dtc -@ -I dts -O dtb -o x.dtbo x.dts` (dtc from device-tree-compiler).
