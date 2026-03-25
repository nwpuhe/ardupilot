libraries/AP_HAL_ChibiOS/hwdef/PRSLab_FC/目录下编写 hwdef.dat 和 hwdef-bl.dat

```bash
# 配置bootloader
./waf clean
rm -rf build/PRSLab_FC

# 编译 bootloader
./waf configure --board PRSLab_FC --bootloader
./waf bootloader
cp build/PRSLab_FC/bin/AP_Bootloader.bin Tools/bootloaders/PRSLab_FC_bl.bin
# 或者 使用脚本直接编译拷贝（建议）
Tools/scripts/build_bootloaders.py PRSLab_FC

# 擦除+烧录
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" -c "init" -c "reset halt" -c "flash erase_sector 0 0 last" -c "flash erase_sector 1 0 last" -c "shutdown"
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "program build/PRSLab_FC/bin/AP_Bootloader.bin verify reset exit 0x08000000"

# 调试
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg
telnet localhost 4444

```

```bash
# 1. 配置编译环境
./waf configure --board PRSLab_FC

# 2.1 编译 copter 固件，并直接通过 USB 烧录！
./waf copter --upload

# 2.2 编译并使用 daplink+openocd 刷入
./waf copter
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" -c "program build/PRSLab_FC/bin/arducopter.bin verify reset exit 0x08020000"

```






