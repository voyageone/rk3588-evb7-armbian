# Armbian 镜像编译项目 — RK3588 EVB7

> 项目启动：2026-05-28
> 最后更新：2026-05-28
> 状态：🔄 首次编译进行中

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

- **eMMC**：正常工作（`status=okay`）
- **SD 卡**：支持（`sdmmc@fe2c0000`, SDHC）
- **GMAC0**：正常工作，1Gbps 链路
- **GMAC1**：存在但可能未连接物理网口
- **PCIe**：3 组均已启用
- **USB3**：2 路均已启用
- **WiFi/BT**：SDIO WiFi + UART9 蓝牙已配置

### 2.4 当前运行系统

- Debian 11（Linaro ALIP）
- 内核版本：供应商定制（基于 5.10）
- 用户：`linaro`

---

## 三、技术决策

### 3.1 为什么用 GitHub Actions 编译？

- Armbian 官方提供了 `armbian/build` GitHub Action
- GitHub Actions 免费账户每月 2000 分钟
- 编译 RK3588 镜像约需 30-60 分钟
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

---

## 四、编译配置

### 4.1 仓库结构

```
voyageone/rk3588-evb7-armbian/
├── .github/workflows/build.yml    # GitHub Actions workflow
├── README.md
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
```

### 6.3 关键对比点

- `compatible` 字符串 → 匹配 Armbian board name
- 串口控制台（`console=` in cmdline）→ 设置 `SERIALCON`
- Ethernet PHY 配置 → 确认网口正常工作
- eMMC/SD 节点 → 确认存储可用

---

## 七、编译监控

### 7.1 当前编译状态

| 项目 | 详情 |
|------|------|
| 首次编译触发时间 | 2026-05-28 04:13 UTC |
| GitHub Actions Run | [链接](https://github.com/voyageone/rk3588-evb7-armbian/actions/runs/26554251571) |
| 监控 Cron Job | 每 5 分钟检查一次，最多 15 次 |
| 预计完成时间 | 2026-05-28 05:00-05:30 UTC |

### 7.2 手动检查命令

```bash
# 查看运行状态
gh run view 26554251571 --repo voyageone/rk3588-evb7-armbian

# 查看实时日志
gh run view 26554251571 --repo voyageone/rk3588-evb7-armbian --log

# 下载编译产物
gh release download --repo voyageone/rk3588-evb7-armbian --pattern "*.img.xz"
```

### 7.3 可能的编译失败原因

1. **磁盘空间不足**：GitHub Actions runner 只有 ~14GB 可用，RK3588 编译需要大量空间
   - 已配置 `jlumbroso/free-disk-space` action 清理空间
2. **U-Boot 编译失败**：`rk3588_defconfig` 可能在 Radxa 仓库中不完整
   - 备选：改用 `rock-5b-rk3588_defconfig`
3. **内核编译失败**：vendor 内核 6.1 可能有兼容性问题
   - 备选：切换到 `current` 分支（主线内核）
4. **board config 未被识别**：userpatches 路径不对
   - 检查：`userpatches/config/boards/rk3588-evb7.conf` 路径是否正确

---

## 八、后续计划

### 8.1 首次编译完成后

- [ ] 下载镜像文件
- [ ] 烧录到 SD 卡测试启动
- [ ] 验证：串口输出、eMMC 识别、网口、USB
- [ ] 记录启动日志和任何问题

### 8.2 可能需要的调整

- 如果 U-Boot 启动失败 → 换 BOOTCONFIG（如 `rock-5b-rk3588_defconfig`）
- 如果网口不通 → 检查 GMAC0 PHY 配置，可能需要 DTB overlay
- 如果 eMMC 找不到 → 检查 dwcmshc 驱动是否正确加载
- 如果 WiFi 不工作 → 需要额外安装驱动（RTL8188EUS）

### 8.3 长期目标

- 切换到 `current` 或 `edge` 主线内核
- 启用 NPU 支持
- 配置 PCIe NVMe 启动
- 创建完整桌面镜像（xfce/gnome）

---

## 九、相关链接

- GitHub 仓库：https://github.com/voyageone/rk3588-evb7-armbian
- Armbian Build Framework：https://github.com/armbian/build
- Armbian Rockchip 内核：https://github.com/armbian/linux-rockchip
- Radxa U-Boot：https://github.com/radxa/u-boot
- Armbian Board Config 文档：https://github.com/armbian/build/blob/main/config/boards/README.md

---

## 十、变更日志

### 2026-05-28

- 项目启动
- 从开发板提取 DTB 并完成对比验证
- 创建 GitHub 仓库 `voyageone/rk3588-evb7-armbian`
- 创建自定义 board config（rk3588-evb7.conf）
- 解决 GitHub token workflow scope 问题
- 触发首次编译（Ubuntu Noble + vendor kernel + minimal）
- 设置自动监控 cron job
