# Armbian Image for Rockchip RK3588 EVB7 LP4 V10

Custom Armbian build for the Rockchip RK3588 Evaluation Board 7.

## Board Specifications

- **SoC:** Rockchip RK3588 (4x Cortex-A76 + 4x Cortex-A55 + Mali-G610)
- **RAM:** 16GB LPDDR4
- **Storage:** eMMC 128GB
- **Network:** Dual Gigabit Ethernet
- **WiFi:** USB Realtek RTL8188EUS (external dongle)

## Build

Go to **Actions** → **Build Armbian for RK3588 EVB7** → **Run workflow**

Options:
- **Kernel branch:** `vendor` (recommended, Rockchip 6.1), `current` (mainline), `edge`
- **Release:** `noble` (Ubuntu 24.04), `bookworm` (Debian 12), `trixie`
- **UI:** `minimal`, `server`, `xfce`

Build takes ~30-60 minutes. The image will be uploaded as a GitHub Release.

## Flash

```bash
# Write to SD card (replace sdX with your device)
xzcat Armbian_*.img.xz | sudo dd of=/dev/sdX bs=1M status=progress

# Or write to eMMC via USB OTG (rockusb mode)
```

## Notes

This uses a custom board config (`userpatches/config/boards/rk3588-evb7.conf`) with:
- U-Boot defconfig: `rk3588_defconfig` (generic Rockchip reference)
- DTB: `rk3588-evb7-lp4-v10.dtb` (vendor kernel)
- Boot scenario: `spl-blobs`

If boot fails, try changing `BOOTCONFIG` to `rock-5b-rk3588_defconfig` in the board config.
