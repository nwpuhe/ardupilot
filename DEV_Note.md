libraries/AP_HAL_ChibiOS/hwdef/PRSLab_FC/目录下编写 hwdef.dat 和 hwdef-bl.dat

```bash
# 编译bootloader
Tools/scripts/build_bootloaders.py PRSLab_FC

# 或者如下方便编译拷贝
./waf clean
rm -rf build/PRSLab_FC
rm -rf Tools/bootloaders/PRSLab_FC_bl.bin
./waf configure --board PRSLab_FC --bootloader
./waf bootloader
cp build/PRSLab_FC/bin/AP_Bootloader.bin Tools/bootloaders/PRSLab_FC_bl.bin

# 擦除+烧录（第一次刷bootloader需要使用daplink）
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" -c "init" -c "reset halt" -c "flash erase_sector 0 0 last" -c "flash erase_sector 1 0 last" -c "shutdown"
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "program build/PRSLab_FC/bin/AP_Bootloader.bin verify reset exit 0x08000000"

# 已经刷过bootloader，更新bootloader可以通过typec直接更新bootloader固件
./waf bootloader --upload

# 调试
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg
telnet localhost 4444
```

```bash
# 1. 配置编译环境（不允许debug）
./waf configure --board PRSLab_FC
# 允许 daplink 进行 debug
./waf configure --board PRSLab_FC --debug
# 软件匹配双进度（默认不用）
./waf configure --board PRSLab_FC --ekf-double

# 2. 编译并使用 daplink+openocd 刷入
./waf copter
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" -c "program build/PRSLab_FC/bin/arducopter.bin verify reset exit 0x08020000"
# 或者编译 copter 固件，并直接通过 USB 烧录！
./waf copter --upload
```


**Github同步修改操作步骤：**
1. **网页端 Fork**：登录你的 GitHub，打开 [ArduPilot 官方仓库](https://github.com/ArduPilot/ardupilot)，点击右上角的 **Fork**，把它复刻到你自己的账号下。
2. **本地绑定你的远程仓库**：
   在你的 Ubuntu 终端里，给现有的本地代码库添加你自己的远程地址：
   ```bash
   git remote add myfork git@github.com:你的GitHub用户名/ardupilot.git
   ```
3. **创建专属分支并提交**：
   千万别在 `master` 分支上直接改，建一个新分支：
   ```bash
   git checkout -b prslab_fc_minimal_port
   ```
   然后把你新建的板子文件夹加进去：
   ```bash
   git add .
   git commit -m "hwdef: add PRSLab_FC minimal port with Bootloader"
   ```
4. **推送到你的 GitHub**：
   ```bash
   git push myfork prslab_fc_minimal_port
   # 或者
   git push -u myfork prslab_fc_minimal_port
   ```
在第二条命令里，加上 -u (即 --set-upstream) 是个一劳永逸的好习惯。它会把你本地的这个分支和你 GitHub 上的分支绑定起来。这意味着，等你以后给板子加上了 IMU 驱动、罗盘驱动，再次修改了 hwdef.dat 时，你只需要简单地敲：
```bash
git add .
git commit -m "add IMU drivers"
git push
```


既然你决定现在就把它同步到最新，那我们就来掌握 Git 里面最优雅、也是 ArduPilot 官方最推荐的代码同步技能：**变基（Rebase）**。

这个操作的逻辑非常形象：这就好比官方写了 42 页新内容，你写了 1 页新内容。我们现在要把你写的这 1 页先“拿起来”，把官方的 42 页垫在下面，然后再把你这 1 页原封不动地“放回最顶上”。这样既同步了官方最新代码，又保证了你的修改记录依然是最新的。

请在终端依次执行以下 4 步：

### 1. 绑定官方的“源头”仓库 (Upstream)
你现在只绑定了你自己的 Fork（`myfork`），我们需要告诉你的电脑，ArduPilot 官方的仓库在哪里：
```bash
git remote add upstream https://github.com/ArduPilot/ardupilot.git
```

### 2. 把官方最新的这 42 个改动“下载”到本地缓存
这一步只是下载，还不会影响你当前的代码：
```bash
git fetch upstream
```

### 3. 执行“变基”（Rebase），把你的代码放到最顶上
告诉 Git，把当前分支的基础，变成官方 `master` 分支的最新状态：
```bash
git rebase upstream/master
```
*(执行完这句，你会看到提示说 `Successfully rebased and updated...`，这就说明你的那 1 页代码已经完美盖在官方的最新代码之上了！)*

### 4. 强制推送到你的 GitHub
因为我们刚才“篡改”了历史的时间线（把你的提交往后挪了 42 个身位），所以向 GitHub 上传时，需要加一个 `-f`（force 强制覆盖）参数，告诉 GitHub 接受这个新的时间线：
```bash
git push -f myfork prslab_fc_minimal_port
```

如果变基的时候，有未暂存的变更，则不能变基
```bash
git stash
git rebase upstream/master
git stash pop
```

直接提交+变基
```bash
git add .
git commit -m "update DEV_Note.md"
# 或者
git commit --amend --no-edit
git rebase upstream/master
```

# 安全芯片的使用
I2C4 地址 0x60
编译开启功能：在未来编译时，可能需要通过类似 ./waf configure --board PRSLab_FC --enable-opendroneid 的标志来把安全加密库（CryptoAuthLib）编译进你的固件。
地面站参数激活：在 QGC 里配置与 DID_ (Drone ID) 相关的参数，告诉飞控“去 I2C4 总线上用加密芯片进行签名”。


# [参数配置]

# 设为 7 开启 3 IMU
EK3_IMU_MASK 7

# 温度校准
INS_TCAL_OPTIONS 1
INS_TCAL1_ENABLE Enable
INS_TCAL2_ENABLE Enable
INS_TCAL3_ENABLE Enable

# INS 位置
INS_POS1_X
INS_POS1_Y
INS_POS1_Z
INS_POS2_X
INS_POS2_Y
INS_POS2_Z
INS_POS3_X
INS_POS3_Y
INS_POS3_Z

# 罗盘
COMPASS_AUTO_ROT Disable
COMPASS_CAL_FIT Relaxed
