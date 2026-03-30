# PRSLab_FC 开发与维护指南

本文档记录了基于 STM32H7 的 PRSLab_FC 飞控板在 ArduPilot 环境下的日常开发、编译烧录、Git 同步以及核心参数配置流程。

---

## 1. Bootloader 编译与烧录

所有硬件定义文件均位于 `libraries/AP_HAL_ChibiOS/hwdef/PRSLab_FC/` 目录下（包含 `hwdef.dat` 和 `hwdef-bl.dat`）。

### 1.1 编译 Bootloader
推荐使用完整的 Waf 流程进行清理和编译，方便后续拷贝与管理：

```bash
# 清理旧的编译缓存
./waf clean
rm -rf build/PRSLab_FC
rm -rf Tools/bootloaders/PRSLab_FC_bl.bin

# 配置并编译 Bootloader
./waf configure --board PRSLab_FC --bootloader
./waf bootloader

# 提取生成的 Bootloader 固件
cp build/PRSLab_FC/bin/AP_Bootloader.bin Tools/bootloaders/PRSLab_FC_bl.bin
```
*(备用快捷命令：`Tools/scripts/build_bootloaders.py PRSLab_FC`)*

### 1.2 首次擦除与烧录 (依赖 DAPLink + OpenOCD)
对于白板芯片，首次必须使用硬件调试器进行双 Bank 扇区擦除和烧录：

```bash
# 擦除 Bank1 的前两个扇区
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg \
  -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" \
  -c "init" -c "reset halt" \
  -c "flash erase_sector 0 0 last" \
  -c "flash erase_sector 1 0 last" \
  -c "shutdown"

# 烧录 Bootloader 到 0x08000000
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg \
  -c "program build/PRSLab_FC/bin/AP_Bootloader.bin verify reset exit 0x08000000"
```

### 1.3 日常更新 Bootloader
如果飞控已经刷入过 Bootloader，后续更新只需通过 Type-C 数据线直连即可：
```bash
./waf bootloader --upload
```

---

## 2. App 固件编译与调试

### 2.1 Waf 环境配置
根据开发需求，选择相应的配置指令：

```bash
# 1. 默认配置（不允许 Debug）
./waf configure --board PRSLab_FC

# 2. 开启 Debug 模式（允许 DAPLink 介入调试）
./waf configure --board PRSLab_FC --debug

# 3. 开启 EKF 双精度（通常情况默认不用）
./waf configure --board PRSLab_FC --ekf-double
```

### 2.2 编译与烧录固件
```bash
# 编译 Copter 固件
./waf copter

# 方式 A：使用 DAPLink 烧录到 0x08020000 (App 起始地址)
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg \
  -c "flash bank bank1 stm32h7x 0x08100000 0 0 0 stm32h7x.cpu0" \
  -c "program build/PRSLab_FC/bin/arducopter.bin verify reset exit 0x08020000"

# 方式 B：通过 Type-C 数据线直接烧录 (推荐)
./waf copter --upload
```

### 2.3 OpenOCD 实时调试
保持 OpenOCD 服务在后台运行，然后通过 Telnet 接入 GDB 进行底层调试：
```bash
openocd -f interface/cmsis-dap.cfg -f target/stm32h7x.cfg
telnet localhost 4444
```

---

## 3. WSL2 环境配置 (USB 穿透)

在 Windows + WSL2 环境下开发时，需要使用 `usbipd-win` 将 USB 设备（DAPLink 或飞控板）穿透到 Ubuntu 子系统。

**在 Windows PowerShell (管理员) 中执行：**
```powershell
# 1. 查看设备列表，记录目标设备的 BUSID
usbipd list

# 2. 绑定设备 (仅需执行一次)
usbipd bind --busid <BUSID>

# 3. 穿透挂载到 WSL (若使用 --upload 刷固件，建议加 --auto-attach 参数)
usbipd attach --wsl --busid <BUSID>

# 附：如果 WSL 出现互操作性异常，可彻底重启 WSL 虚拟机
wsl --shutdown
```

---
## 4. Git 协作与版本控制 (Fork & Rebase 工作流)

为了保持私人分支历史的纯净，并随时准备向 ArduPilot 官方提交 PR，我们采用标准的开源协作模式：**Fork 个人仓 + Rebase 官方主线**。

### 4.1 初始化与分支创建 (仅首次配置)

在 GitHub 网页端将 ArduPilot 官方仓库 Fork 到个人账号后，按以下步骤在本地建立开发环境：

```bash
# 1. 克隆你自己的 Fork 仓库到本地 (使用 SSH 或 HTTPS)
git clone git@github.com:nwpuhe/ardupilot.git
cd ardupilot

# 2. 绑定 ArduPilot 官方仓库作为“源头” (命名为 upstream)
git remote add upstream [https://github.com/ArduPilot/ardupilot.git](https://github.com/ArduPilot/ardupilot.git)

# 3. 基于当前代码，创建并切换到你的专属硬件适配分支
git checkout -b prslab_fc_minimal_port

# 4. 将新建的分支推送到你的个人云端，并建立追踪关系 (-u)
git push -u origin prslab_fc_minimal_port
```

### 4.2 日常同步与变基 (Rebase)

> **💡 Rebase 的逻辑理解：**
> 这就好比官方写了 42 页新内容，你写了 1 页新内容。变基就是把你写的这 1 页先“拿起来”，把官方的 42 页垫在下面，然后再把你这 1 页原封不动地“放回最顶上”。既同步了代码，又保证了你的修改记录永远处于时间线的最前端。

当官方主线 (`master`) 有了更新，你需要同步这些代码到你的板子分支时，执行以下操作：

```bash
# 1. 下载官方最新的代码变化到本地缓存
git fetch upstream

# 2. 执行变基，将你的专属修改“垫”在官方最新代码之上
git rebase upstream/master

# 3. 强制推送覆盖个人仓库 (因为变基修改了时间线，必须加 -f)
git push -f origin prslab_fc_minimal_port
```

### 4.3 常见场景处理

* **当前工作区有代码没写完，但急需同步官方代码：**
  ```bash
  git stash                   # 暂存当前未提交的修改
  git fetch upstream
  git rebase upstream/master  # 执行变基
  git stash pop               # 恢复刚才未写完的代码
  ```

* **发现刚提交的代码有拼写错误，想直接修改并变基，不想产生多余的提交记录：**
  ```bash
  git add .
  git commit --amend --no-edit  # 悄悄合并到上一次提交中
  git rebase upstream/master
  git push -f origin prslab_fc_minimal_port
  ```

---
## 5. 硬件特性与参数配置

### 5.1 安全加密芯片 (CryptoAuthLib)
* **硬件总线**：I2C4
* **设备地址**：`0x60`
* **编译激活**：未来编译时需通过 `./waf configure --board PRSLab_FC --enable-opendroneid` 开启安全加密库编译。
* **地面站配置**：在 QGC/MP 中配置与 `DID_` (Drone ID) 相关的参数，指示飞控调用 I2C4 上的加密芯片进行数字签名。

### 5.2 核心飞行参数 (QGC/MP)

| 功能模块 | 参数名 | 推荐值 | 说明 |
| :--- | :--- | :--- | :--- |
| **IMU 冗余** | `EK3_IMU_MASK` | `7` | 启用全部 3 颗 IMU 参与 EKF 解算 |
| **温度校准** | `INS_TCAL_OPTIONS` | `1` | 开启出厂温度校准补偿 |
| | `INS_TCAL1_ENABLE` | `Enable` | 启用 IMU1 温补 |
| | `INS_TCAL2_ENABLE` | `Enable` | 启用 IMU2 温补 |
| | `INS_TCAL3_ENABLE` | `Enable` | 启用 IMU3 温补 |
| **位置偏移** | `INS_POSx_X/Y/Z` | 视结构定 | 精确设定 3 颗 IMU 相对于重心的物理偏移量 |
| **罗盘配置** | `COMPASS_AUTO_ROT` | `Disable` | 关闭自动方向推断（硬件已严格对齐） |
| | `COMPASS_CAL_FIT` | `Relaxed` | 放宽罗盘校准的严格度 |
---