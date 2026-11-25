# rkdeveloptool（TL3588/RK3588 增强版）

基于 [rockchip-linux/rkdeveloptool](https://github.com/rockchip-linux/rkdeveloptool) 官方 master（v1.32）的增强分支，版本号 **1.33**（`rkdeveloptool -v` 可区分）。新增 `uf` / `afp` 两条命令，使 Linux 主机具备与 Windows RKDevTool"升级固件"页对等的一键烧写 RKFW `update.img` 能力；并针对 RK3588 修复了设备识别与权限诊断问题。

> 关联文档：Obsidian 库 `rk3588/tl3588/rkdeveloptool-uf-烧写updateimg指引.md`、`rk3588/tl3588/TL3588-无SD卡MaskROM烧写eMMC指引.md`（本 README 为两者的整合版）

## 1. 相对官方 master 的改动

| 改动 | 说明 |
|---|---|
| **新增 `uf <update.img>`** | 一键烧写整包 RKFW 升级固件（自动解 loader、按固件自带分区表逐分区写入、写 parameter/idblock、复位），对应 Windows RKDevTool"升级固件"页 |
| **新增 `afp <update.img>`** | 只解析固件包并打印分区项，**不需要插板子**，用于烧前预检（对应 Windows"解包"功能） |
| **udev 规则补齐 RK3588** | `99-rk-rockusb.rules` 原来只收录到 RK3566（`350a`），已补 `2207:350b`（RK3588/RK3588S，MaskROM 与 Loader 共用该 PID） |
| **MaskROM/Loader 判别修复（RK3588）** | 官方代码用 `bcdUSB bit0` 区分两种模式，但 RK3588 的 MaskROM 和 Loader 都报 `bcdUSB=2.00`，导致 Loader 态被误判为 Maskrom。现改为：bit0=0 且**无字符串描述符**（maskrom ROM 无 `Manufacturer/Product` 字符串）才是 MaskROM，否则按 Loader 处理 |
| **失败原因可视化** | USB 打开/断言失败时，控制台直接打印具体 libusb 错误码与名称；权限不足（`LIBUSB_ERROR_ACCESS`）时提示安装 udev 规则或用 sudo |

## 2. 编译与安装（Linux 主机）

```bash
# 依赖（Ubuntu/Debian；apt install rkdeveloptool 无效，官方仓库没有该包）
sudo apt install -y build-essential autoconf automake libtool pkg-config libusb-1.0-0-dev

cd ~/projects/rkdeveloptool
./autogen.sh && ./configure && make -j$(nproc)
sudo install -m755 ./rkdeveloptool /usr/local/bin/
rkdeveloptool -v    # 应显示 ver 1.33

# 安装 udev 规则免 sudo（已含 2207:350b）
sudo cp 99-rk-rockusb.rules /etc/udev/rules.d/
sudo udevadm control --reload && sudo udevadm trigger
```

> ⚠️ **不要用 SDK `rkbin/tools/` 自带的 rkdeveloptool**：其校验逻辑为旧版，对老格式（`LDR ` tag）loader 会误报 `Opening loader failed`（文件本身完好）。本分支基于官方 master，已修正该判断。

## 3. 让板子进入烧写模式（TL3588-EVM）

任选其一，插入 **OTG 口**（usbdrd30，与调试串口共用 Type-C；普通 USB host 口不能烧写）后由 PC 侧确认：

1. **eMMC 空片 / 引导损坏（最常见）**：插 USB 线上电，BootROM 发现 eMMC 无有效引导，**自动进入 MaskROM**，无需任何按键。
2. **eMMC 里有旧固件**：按住底板 **RECOVERY/更新按键**（以底板丝印为准）再上电或复位 → 进入 Loader 下载模式（同样可直接烧写）。
3. **兜底方法**（板上无按键且旧系统能启动）：从旧系统里擦掉引导块后重启，即自动进 MaskROM：
   ```bash
   dd if=/dev/zero of=/dev/mmcblk0 bs=512 seek=64 count=800 conv=fsync && sync && reboot
   ```

**PC 侧确认**：

```bash
lsusb | grep 2207          # 应看到 2207:350b（RK3588 MaskROM/Loader 共用 PID）
rkdeveloptool ld           # 显示 Maskrom 或 Loader
```

> 判别依据：MaskROM 无字符串描述符（`lsusb -v` 中 `iManufacturer=0, iProduct=0`）；Loader 重枚举后带 `Manufacturer: RockChip / Product: USB-MSC / SerialNumber: rockchip` 字符串。两种模式 `lsusb` 显示的 PID 均为 `350b`。

## 4. 烧写方式

### 4.1 方式一：`uf` 一键烧写 update.img（本分支新增）

固件形态为 RKFW 升级固件（出厂固件 / 官方发布包 / `img_maker` 打包的 update.img）时使用：

```bash
# 0.（可选）afp 预检固件包，不需要插板子
rkdeveloptool afp update_v1.1.img

# 1. 确认设备已进 MaskROM/Loader
rkdeveloptool ld

# 2. 一键烧写
rkdeveloptool uf relinux-image-std-rockchip-rk3588.rootfs-*.update.img

# 3. 完成后工具自动复位，拔线重新上电即可
```

`afp` 预检输出示例（出厂 update_v1.1.img 实测）：

```
RKFW header OK, firmware data at 0x76226 (3909214212 bytes)
RKAF package parsed, 10 item(s):
  parameter    path=parameter.txt        flash LBA 0x0       partition 0x4000 sectors, data=552 bytes
  CMDLINE: mtdparts=:0x00002000@0x00004000(uboot),...,-@0x01cb8000(userdata:grow)
  bootloader   path=MiniLoaderAll.bin    data=483776 bytes, not for flash
  uboot        path=uboot.img            flash LBA 0x4000    partition 0x2000 sectors, data=4194304 bytes
  ...
```

CMDLINE 中各分区起始扇区与条目 flash LBA 完全一致，即格式解析正确。

`uf` 内部自动依次执行：

1. 解析 RKFW 外壳 → 取出内嵌 loader（出厂固件为老格式 `LDR` loader，已实测可解析）；
2. 解析内层 RKAF 包（afptool 格式）→ 得到分区项与 parameter.txt；
3. 设备处于 **MaskROM** 时，先 `db`（loader 只下载进内存）；设备已在 **Loader** 模式时跳过；
4. 按 parameter 布局写入：
   - parameter 声明 `TYPE: GPT` → 生成 **GPT**（含 uuid 项）写 LBA 0 / 末尾备份；
   - 否则（老式 mtdparts 布局，自打包固件此类）→ 把 **parameter 缓冲写 LBA 0x2000**；
5. 逐分区把数据写入固件声明的 LBA（uboot@0x4000、boot@0x8000、recovery@0x28000、rootfs@0x78000、oem@0x1C78000、userdata@0x1CB8000 …），带百分比进度；
6. 用内嵌 loader 生成 **idblock 写 LBA 64**（等价 `ul`）；
7. 复位设备。

### 4.2 方式二：`db` + `wl` 整盘写 wic 镜像

固件形态为自研 relinux `.wic` 整盘镜像（已含 GPT 分区表、idblock@扇区64、uboot@8M、rootfs@16M）时使用——wic 本身就是整盘镜像，从 LBA 0 整体写入 eMMC 用户区，效果与 dd 写 SD 卡完全等价，全程不需要 SD 卡：

```bash
# 1. 确认设备已进入 MaskROM/Loader
rkdeveloptool ld

# 2. 下载 loader（只进内存，不写 eMMC；Loader 模式下可跳过）
rkdeveloptool db MiniLoaderAll-tronlong-v1.1.bin

# 3. 整盘写入（LBA 0 开始；会清空 eMMC 整个用户区）
rkdeveloptool wl 0 relinux-image-base-rockchip-rk3588.rootfs.wic

# 4. 复位
rkdeveloptool rd
```

### 4.3 两种方式对比

| | `uf update.img`（本分支新增） | `db` + `wl 0 xxx.wic`（原方式） |
|---|---|---|
| 固件形态 | RKFW 升级固件（出厂固件/官方发布包） | 整盘 wic 镜像 |
| 分区布局 | 固件自带（GPT 或 mtdparts），含 backup/oem/userdata | wic 的 GPT（idb@64/uboot@8M/rootfs@16M） |
| idblock | 从 RKFW 内嵌 loader 写入 | wic 内含 |
| 适用场景 | 客户拿到的是 update.img 时，Windows/Linux 体验一致 | 自研 relinux 固件 |

### 4.4 Windows 主机（对照，推荐给客户）

1. 安装驱动：SDK `4-软件资料/Tools/Windows/DriverAssitant_v5.12.zip`，管理员运行 `DriverInstall.exe` → "驱动安装"；
2. 运行 `RKDevTool_Release_v3.15.zip` 解压出的 `RKDevTool.exe`（免安装绿色版）；
3. `.wic` 整盘：切到**"下载镜像"**页 → 存储选 eMMC → 添加 `.wic` 项、**起始地址填 0** → 仅勾选该项 → "执行"；若提示需要 Loader，选 `MiniLoaderAll-tronlong-v1.1.bin`；
4. `update.img`：用**"升级固件"**页 → "固件"按钮选 update.img → "升级"；
5. 等进度 100%（2.6G 约 **10~30 分钟**）→ 拔线重新上电。

## 5. 烧写后验证

1. 串口（板卡调试口）**115200** 8N1（本固件全链 115200；出厂固件才是 1500000，勿混淆），重新上电应看到：DDR init 版本打印 → SPL/U-Boot → `Linux version 5.10.110-...` → rootfs 挂载 → login 提示符；
2. 进入系统后 `lsblk`：根分区应为 `/dev/mmcblk0p2`（rootfs@16M 布局）。

## 6. 常见问题

| 现象 | 处理 |
|---|---|
| `Creating Comm Object failed!` | USB 设备无访问权限：安装本仓库 `99-rk-rockusb.rules` 并 `udevadm control --reload && sudo udevadm trigger`，或直接 `sudo` 运行。新版会打印具体 libusb 错误（`LIBUSB_ERROR_ACCESS` = 权限） |
| `ld` 显示 Maskrom 但板子其实已跑 loader | 旧版 bug：RK3588 两种模式 `bcdUSB` 相同，bit0 判别失效。换本分支 1.33（按字符串描述符判别） |
| PC 识别不到设备 | 确认线插的是 **OTG 口**；换线/换口；重进 MaskROM（断电→按住 Recovery→上电） |
| Windows 提示找不到驱动 | 重装 DriverAssitant；设备管理器看是否有带感叹号的 Rockchip 设备 |
| `rkdeveloptool: no devices found` | 未装 udev 规则时需 `sudo`；`lsusb` 确认 2207 设备存在 |
| `Opening loader failed` | ① 路径写错（相对路径时先 cd 到文件所在目录）；② rkdeveloptool 版本过旧不认老格式 loader——换官方 master 编译版（见第 2 节）。该报错发生在 USB 通信之前，与设备端状态无关 |
| 烧写中途失败 | 多为线材/供电问题；重新 `db` + `wl`（或重跑 `uf`）即可 |
| 烧完不启动 | 串口看卡在哪一级（DDR/SPL/U-Boot/kernel），连同波特率一起反馈 |
| 烧写太慢 | 见附录 A 截断镜像尾部空白 |

## 7. 已知限制与待验证项

- **RKAF 条目包内偏移为 32 位**：update.img 固件数据超过 4GB 时无法支持（Windows 版同样受限）。
- **`uf` 不做整片擦除**（与 Windows"升级固件"行为一致）；如需彻底清空请先 `ef`。
- 若 eMMC 上此前烧过 wic（GPT 布局），`uf` 会覆盖固件覆盖到的区域；LBA 0 起的老 GPT 仅当 parameter 为 GPT 类型时才会被新 GPT 覆盖，老式 mtdparts 固件不受影响（uboot 只读 parameter）。
- **待实测（2026-11-25）**：`uf` 的 USB 下载全流程（db→分区→param/GPT→idblock→复位）逻辑复用自已验证的 db/ul/wl/prm/gpt 代码路径，出厂老格式 LDR loader 已通过 afp 解析验证，但**自研 relinux 新格式 MiniLoaderAll.bin 经 `uf` 对真 MaskROM 下载时出现挂死**（"Downloading bootloader..."后无 USB 活动、板子不重枚举，0x471/0x472 vendor 请求路径待排查，见第 8 节代码索引）。此问题解决前，自研固件建议用 4.2 节 `db`+`wl` 方式。
- 烧写耗时与 `wl` 相当：rootfs 3.5GB，USB2.0 下约 10~30 分钟。

## 8. 代码索引（本分支改动位置）

- `main.cpp`
  - `STRUCT_RKAF_HEAD` / `STRUCT_RKAF_ENTRY`：RKAF 头与条目结构（0x8C 头 + 112B/条目）
  - `parse_af_image()` / `parse_update_image()`：RKFW→RKAF 解析
  - `write_lba_offset()`：按包内偏移流式写分区（64KB 块 + 进度）
  - `write_param_data()`：parameter 写 LBA 0x2000（自动识别 PARM 包装）
  - `write_gpt_data()`：按 parameter 生成并写 GPT（TYPE: GPT 时）
  - `upgrade_fw()`：uf 主流程；`print_af_info()`：afp 输出
  - `upgrade_loader()`：设备类型检查放宽为 MaskROM|Loader（Loader 态也能补写 idblock）
  - `print_usb_comm_failure()`：CRKUsbComm 创建失败时打印 libusb 错误与权限提示（2026-11-25 新增）
- `RKScan.cpp`
  - `EnumerateUsbDevice()`：记录 `descriptor.iManufacturer/iProduct` 是否存在字符串描述符（`ucHasStrings`）
  - `Search()` / `Wait()`：MaskROM/Loader 判别改为"bit0=0 且无字符串描述符"（修复 RK3588 Loader 误判）
- `RKComm.cpp` / `RKComm.h`
  - `InitializeUsb()`：各失败路径记录 `m_usbError`；`GetUsbError()` 供控制台展示
- `99-rk-rockusb.rules`：补 `2207:350b`（RK3588/RK3588S）
- 回归测试：`docs/test/rkdeveloptool-uf/`（合成最小 update.img，裸 parameter 非 GPT 分支）

## 附录 A：烧写提速（截断 wic 尾部空白，可选）

wic 尾部常有稀疏空白，可先截断再提供/烧写（不影响分区数据）：

```bash
# 取最后一个分区的 start+size（单位=512B 扇区），截断到分区末尾
END=$(sfdisk -d x.wic | tail -1 | sed 's/.*start=\([0-9]*\), size=\([0-9]*\).*/\1+\2/')
truncate -s $(( $(echo "$END" | bc) * 512 )) x.wic
```

## 附录 B：打包 RKFW update.img 脚本（从 wic 重打包）

若客户只能走"升级固件"页，需把 wic 拆分重打包为 update.img。**注意 parameter.txt 必须按我们 wic 的 GPT 布局写（uboot@8M、rootfs@16M），不能沿用出厂 mtdparts（boot/recovery@80M/240M 那套）。此脚本未做过板级回环验证，首次使用需先烧一块验证板确认。**

依赖：`afptool`、`img_maker`（github.com/rockchip-linux/rk，Linux x64 二进制），`sfdisk`、`dd`。

```bash
#!/bin/bash
# make-updateimg-from-wic.sh <固件.wic> <MiniLoaderAll.bin> <输出update.img>
set -e
WIC=$1; LOADER=$2; OUT=$3
WORK=$(mktemp -d)

# 1. 按 GPT 布局切出分区镜像（uboot@8M=扇区16384，rootfs@16M=扇区32768）
dd if=$WIC of=$WORK/uboot.img  bs=512 skip=16384 count=8192 status=none   # 4M uboot 分区
dd if=$WIC of=$WORK/rootfs.img bs=512 skip=32768 status=none              # 到镜像尾

# 2. parameter.txt（分区地址与 wic GPT 严格对齐，单位=512B 扇区）
cat > $WORK/parameter.txt <<'EOF'
FIRMWARE_VER:1.0
MACHINE_MODEL:TL3588
MACHINE_ID:007
MACHINE_NAME:TL3588
DEFAULT_LOADER:ALL
CMDLINE:mtdparts=rk29xxnand:0x2000@0x0040(iddb),0x2000@0x4000(uboot),-@0x8000(rootfs)
EOF

# 3. package-file + afptool 打包 RKAF 层
cat > $WORK/package-file <<EOF
$WORK/parameter.txt	PARAMETER
$WORK/uboot.img	uboot
$WORK/rootfs.img	rootfs
EOF
afptool -pack $WORK/package-file $WORK/update_af.img

# 4. 套 RKFW 外壳（内含 loader）
img_maker -rk3588 $LOADER $WORK/update_af.img $OUT
echo "Done: $OUT"
```

> 说明：iddb 段地址仅占位（升级固件模式下 loader 由 RKFW 头携带写入扇区 64 区域）；打包完成后建议用 RKDevTool"解包"功能或本工具 `afp` 回读核对分区地址，并先在验证板上试烧一次。

---

*README 整合自 Obsidian 库两份指引（2026-11-04 MaskROM 烧写指引、2026-11-25 uf 指引），并纳入 2026-11-25 的 udev 规则、错误诊断与 MaskROM/Loader 判别修复。*
