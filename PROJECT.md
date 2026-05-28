# Armbian 镜像编译项目 — RK3588 EVB7

> 项目启动：2026-05-28
> 最后更新：2026-05-28 05:30 UTC
> 状态：🔄 第二次编译进行中（第一次镜像编译成功但 Release 上传失败）

---

## 一、项目目标

为 Rockchip RK3588 EVB7 LP4 V10 开发板编译 Armbian 镜像，利用 GitHub Actions CI 在云端完成编译，无需本地构建环境。

---

## 二、硬件信息

### 2.1 基本规格

| 项目 | 详情 |
|------|------|
| SoC | Rockchip RK3588（4×A76 + 4×A55 + Mali-G610 + NPU） |
| RAM | 16GB LPDDR4 |
| 存储 | 116.5GB eMMC（HS400, 200MHz） |
| 网络 | 双千兆以太网（GMAC0 + GMAC1，RGMII） |
| WiFi | USB Realtek RTL8188EUS（外置 USB 适配器） |
| 蓝牙 | UART9 BT |
| 显示 | 2×MIPI-DSI, 2×HDMI, 2×EDP, 2×DP |
| 摄像头 | 6×MIPI CSI |
| PCIe | 3 组（可用于 NVMe、WiFi 等） |
| USB | 2 路 USB3（DWC3） |
| SATA | 3 路 |

### 2.2 设备树信息

| 项目 | 值 |
|------|-----|
| compatible | `rockchip,rk3588-evb7-lp4-v10`, `rockchip,rk3588` |
| model | Rockchip RK3588 EVB7 LP4 V10 Board |
| DTB 文件 | `rk3588-evb7-lp4-v10.dtb` |
| 串口控制台 | `ttyFIQ0`（注意：非标准 `ttyS2`） |
| Boot args | `storagemedia=emmc androidboot.mode=normal console=ttyFIQ0 root=PARTUUID=614e0000-0000` |

### 2.3 已确认的硬件状态

- **eMMC**：正常工作（`status=okay`，HS400 模式，200MHz）
- **SD 卡**：支持（`sdmmc@fe2c0000`, SDHC）
- **GMAC0**：正常工作，1Gbps 链路（RGMII，PHY at reg=1）
- **GMAC1**：存在但可能未连接物理网口
- **PCIe**：3 组均已启用（bus 0x30-0x3f, 0x40-0x4f, 0x00-0x0f）
- **USB3**：2 路均已启用（DWC3）
- **WiFi/BT**：SDIO WiFi + UART9 蓝牙已配置
- **NPU**：3 核 NPU 已启用（`status=okay`），内核配置 `CONFIG_ROCKCHIP_RKNPU=y`

### 2.4 当前运行系统

- Debian 11（Linaro ALIP）
- 内核版本：供应商定制（基于 5.10）
- 用户：`linaro`

---

## 三、技术决策

### 3.1 为什么用 GitHub Actions 编译？

- Armbian 官方提供了 `armbian/build` GitHub Action
- GitHub Actions 免费账户每月 2000 分钟
- 编译 RK3588 镜像约需 60-90 分钟
- 无需本地 ARM 构建环境

### 3.2 为什么用自定义 board config？

Armbian **没有** `rk3588-evb7` 的板级配置。可用的选项：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **自定义 userpatches（✅ 采用）** | 精确匹配 EVB7，包含 ttyFIQ0 配置 | 需要维护 |
| 直接用 `rock-5b` | 开箱即用 | DTB 不对，串口不对 |
| 直接用 `orangepi5` | 开箱即用 | DTB 不对，串口不对 |

### 3.3 为什么用 Radxa 的 U-Boot？

Armbian 的 `rockchip-rk3588` 家族配置写死了：

```bash
BOOTSOURCE='https://github.com/radxa/u-boot.git'
BOOTBRANCH='branch:next-dev-v2024.10'
```

Radxa 的 U-Boot 基于 Rockchip SDK，包含 `rk3588_defconfig`，与 EVB7 完全兼容。比主线 U-Boot 更稳定。

### 3.4 DTB 对比验证

从开发板提取的 DTB 与 Armbian vendor 内核中的 `rk3588-evb7-lp4-v10.dtb` **完全一致**，无需额外定制设备树。

### 3.5 NPU 支持

内核配置已包含 `CONFIG_ROCKCHIP_RKNPU=y`，NPU 驱动编译进内核。用户空间 RKNN 运行时库需烧录后从 Rockchip GitHub 安装。

### 3.6 TUN/网络支持

内核配置已包含：
- `CONFIG_TUN=m`（TUN/TAP 设备驱动）
- `CONFIG_WIREGUARD=m`（WireGuard VPN）
- `CONFIG_BRIDGE=m`（网桥）
- `CONFIG_VETH=m`（虚拟以太网对）
- `CONFIG_MACVLAN=m`（MAC VLAN）

OpenVPN、WireGuard、Clash 等工具可直接使用。

---

## 四、编译配置

### 4.1 仓库结构

```
voyageone/rk3588-evb7-armbian/
├── .github/workflows/build.yml    # GitHub Actions workflow
├── README.md
├── PROJECT.md                     # 本文档
└── userpatches/config/boards/
    └── rk3588-evb7.conf           # 自定义板级配置
```

### 4.2 Board Config 内容

```bash
# Rockchip RK3588 EVB7 LP4 V10
# Standard Rockchip evaluation board - vendor reference design
BOARD_NAME="RK3588 EVB7"
BOARD_VENDOR="rockchip"
BOARDFAMILY="rockchip-rk3588"
BOARD_MAINTAINER=""
INTRODUCED="2023"
BOOTCONFIG="rk3588_defconfig"
BOOT_SOC="rk3588"
KERNEL_TARGET="vendor"
BOOT_FDT_FILE="rockchip/rk3588-evb7-lp4-v10.dtb"
BOOT_SCENARIO="spl-blobs"
IMAGE_PARTITION_TABLE="gpt"
# EVB7 uses FIQ debug console (ttyFIQ0), not standard ttyS2
SERIALCON="ttyFIQ0"
```

### 4.3 Workflow 配置

```yaml
armbian_board: "rk3588-evb7"
armbian_target: "build"
armbian_release: "noble"          # Ubuntu 24.04
armbian_kernel_branch: "vendor"   # Rockchip 6.1 内核
armbian_ui: "minimal"
armbian_compress: "sha,img,xz"
permissions:
  contents: write                  # 允许创建 Release
```

### 4.4 源码版本

| 组件 | 仓库 | 分支/版本 |
|------|------|-----------|
| Armbian Build Framework | `armbian/build` | `main`（最新） |
| Linux 内核 | `armbian/linux-rockchip` | `rk-6.1-rkr5.1` |
| U-Boot | `radxa/u-boot` | `next-dev-v2024.10` |
| 用户空间 | Ubuntu Noble 24.04 | — |

---

## 五、遇到的问题与解决方案

### 问题 1：GitHub token 缺少 `workflow` scope

**现象**：推送 `.github/workflows/` 文件时被 GitHub 拒绝，报错 `refusing to allow a Personal Access Token to create or update workflow without 'workflow' scope`

**原因**：`gh auth login` 创建的 token 只有 `repo` scope，缺少 `workflow` scope

**解决**：用户提供新 token（带 `workflow` 权限），通过 `git remote set-url` 替换

**经验**：创建 GitHub 仓库时就需要确保 token 包含 `workflow` scope

### 问题 2：`gh auth refresh` 对 PAT 无效

**现象**：`gh auth refresh -h github.com -s workflow` 对 `ghp_` 开头的 PAT 不生效

**原因**：`gh auth refresh` 仅支持 OAuth token，不支持 Classic PAT

**解决**：用户手动创建新 PAT 并提供

### 问题 3：GitHub Contents API 无法创建 workflow 文件

**现象**：通过 `gh api repos/.../contents/.github/workflows/build.yml` 创建文件返回 404

**原因**：GitHub 对 `.github/workflows/` 目录的任何操作都需要 `workflow` scope

**解决**：先推送非 workflow 文件，再用带 `workflow` scope 的 token 推送 workflow

### 问题 4：Armbian 没有 EVB7 的 board 配置

**现象**：`config/boards/` 中没有 `rk3588-evb7` 相关文件

**原因**：EVB7 是 Rockchip 参考板，不在 Armbian 支持列表中

**解决**：通过 `userpatches/config/boards/` 创建自定义 board config

### 问题 5：串口控制台不匹配

**现象**：开发板使用 `ttyFIQ0`，Armbian 默认用 `ttyS2`

**影响**：如果不对，启动后串口无输出

**解决**：在 board config 中设置 `SERIALCON="ttyFIQ0"`

### 问题 6：Release 上传失败（403）

**现象**：第一次编译镜像成功，但 `ncipollo/release-action` 报错 `Error 403: Resource not accessible by integration`

**原因**：GitHub Actions 的 `GITHUB_TOKEN` 默认只有 `contents: read` 权限，创建 Release 需要 `contents: write`

**影响**：编译产物随 runner 销毁而丢失（无 artifact 备份）

**解决**：
1. 在 workflow 中添加 `permissions: contents: write`
2. 添加 `actions/upload-artifact` 步骤作为备份（保留 7 天）

**经验**：workflow 中涉及 Release 操作时，必须显式声明 `permissions: contents: write`

### 问题 7：ORAS 镜像拉取失败

**现象**：日志中出现 `ORAS manifest fetch error: denied: requested access to the resource is denied`

**原因**：Armbian 尝试从 OCI registry 拉取预构建的 rootfs 缓存，但 registry 访问被拒绝

**影响**：无实质影响，Armbian 会自动回退到源码编译

---

## 六、从开发板提取信息的方法

### 6.1 必须提取的信息

```bash
# 板子型号和兼容字符串（最关键）
cat /sys/firmware/devicetree/base/compatible | tr '\0' '\n'
cat /sys/firmware/devicetree/base/model

# 硬件概况
free -h
lsblk -d -o NAME,SIZE,TYPE,TRAN
ip link show | grep -E "^[0-9]"

# 串口控制台
cat /proc/cmdline | grep -o "console=[^ ]*"

# WiFi/BT
lspci -k 2>/dev/null | grep -i -E "network|wireless"
lsusb 2>/dev/null | grep -i -E "wireless|bluetooth"
```

### 6.2 DTB 提取与反编译

```bash
# 从运行中的系统导出 DTB
cp /sys/firmware/fdt /tmp/board.dtb

# 反编译（需要 dtc）
dtc -I dtb -O dts /tmp/board.dtb -o /tmp/board.dts

# 如果板子无网络，可将 DTB 文件传到其他机器上反编译
# DTB 文件大小约 160-170KB
```

### 6.3 关键对比点

- `compatible` 字符串 → 匹配 Armbian board name
- 串口控制台（`console=` in cmdline）→ 设置 `SERIALCON`
- Ethernet PHY 配置 → 确认网口正常工作
- eMMC/SD 节点 → 确认存储可用

---

## 七、编译监控

### 7.1 编译历史

| 编号 | Run ID | 触发时间 | 状态 | 说明 |
|------|--------|----------|------|------|
| 1 | 26554251571 | 04:13 UTC | ❌ 失败 | 镜像编译成功，Release 上传 403 |
| 2 | 26556488825 | 05:25 UTC | 🔄 进行中 | 已修复权限 + artifact 备份 |

### 7.2 手动检查命令

```bash
# 查看运行状态
gh run view 26556488825 --repo voyageone/rk3588-evb7-armbian

# 查看实时日志
gh api repos/voyageone/rk3588-evb7-armbian/actions/jobs/{job_id}/logs

# 下载编译产物（Release）
gh release download --repo voyageone/rk3588-evb7-armbian --pattern "*.img.xz"

# 下载编译产物（Artifact，备用）
gh run download 26556488825 --repo voyageone/rk3588-evb7-armbian
```

### 7.3 编译耗时参考

| 阶段 | 耗时 | 说明 |
|------|------|------|
| 安装依赖 + 下载源码 | ~10 min | |
| 编译 U-Boot | ~15 min | |
| 编译内核核心 | ~20 min | |
| 编译内核模块 | ~25-35 min | 2000+ 个模块 |
| 创建 rootfs + 打包 | ~10 min | |
| **总计** | **~60-90 min** | |

---

## 八、烧录与启动

### 8.1 烧录到 SD 卡（推荐先试）

```bash
# 解压并写入
xz -d Armbian_*.img.xz
sudo dd if=Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync

# 或一步到位
xzcat Armbian_*.img.xz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
```

### 8.2 烧录到 eMMC

**方法 A：通过 SD 卡中转（最安全）**
1. 先烧 SD 卡，从 SD 卡启动
2. 在 Armbian 内将系统克隆到 eMMC：
   ```bash
   sudo dd if=/dev/mmcblk1 of=/dev/mmcblk0 bs=4M status=progress conv=fsync
   ```

**方法 B：通过 USB OTG（RockUSB 模式）**
1. 按住 Maskrom 按钮上电
2. 使用 `rkdeveloptool` 烧录

### 8.3 串口连接

- 设备：`/dev/ttyUSB0`
- 波特率：`1500000`（Rockchip 默认）
- 数据位：8，停止位：1，无校验

```bash
screen /dev/ttyUSB0 1500000
```

### 8.4 注意事项

- `dd` 会**整卡覆盖**所有分区，原系统不可恢复
- 烧录前建议备份原 eMMC 数据
- 首次启动会自动扩展文件系统、设置 root 密码

---

## 九、后续计划

### 9.1 首次编译完成后

- [ ] 下载镜像文件
- [ ] 烧录到 SD 卡测试启动
- [ ] 验证：串口输出、eMMC 识别、网口、USB
- [ ] 记录启动日志和任何问题

### 9.2 可能需要的调整

- 如果 U-Boot 启动失败 → 换 BOOTCONFIG（如 `rock-5b-rk3588_defconfig`）
- 如果网口不通 → 检查 GMAC0 PHY 配置，可能需要 DTB overlay
- 如果 eMMC 找不到 → 检查 dwcmshc 驱动是否正确加载
- 如果 WiFi 不工作 → 需要额外安装驱动（RTL8188EUS）

### 9.3 长期目标

- 切换到 `current` 或 `edge` 主线内核
- 安装 RKNN 用户空间库启用 NPU
- 配置 PCIe NVMe 启动
- 创建完整桌面镜像（xfce/gnome）
- 定制精简内核配置以加速编译

---

## 十、相关链接

- GitHub 仓库：https://github.com/voyageone/rk3588-evb7-armbian
- Armbian Build Framework：https://github.com/armbian/build
- Armbian Rockchip 内核：https://github.com/armbian/linux-rockchip
- Radxa U-Boot：https://github.com/radxa/u-boot
- Armbian Board Config 文档：https://github.com/armbian/build/blob/main/config/boards/README.md
- Rockchip RKNN SDK：https://github.com/airockchip

---

## 十一、变更日志

### 2026-05-28

- 项目启动
- 从开发板提取 DTB 并完成对比验证
- 创建 GitHub 仓库 `voyageone/rk3588-evb7-armbian`
- 创建自定义 board config（rk3588-evb7.conf）
- 解决 GitHub token workflow scope 问题
- 触发首次编译（Ubuntu Noble + vendor kernel + minimal）
- 设置自动监控 cron job
- **首次编译镜像成功，但 Release 上传失败（403）**
- 修复 workflow：添加 `permissions: contents: write` + artifact 备份
- 触发第二次编译（Run 26556488825）
- 确认 NPU、TUN、WireGuard 等内核功能已启用
