# Armbian SK-AM62A-LP Board Support

Custom Armbian board support files for the **Texas Instruments SK-AM62A-LP** evaluation
board (AM62A7 SoC), enabling a complete Armbian build with working SD card boot.

---

## Hardware

| Field | Value |
|-------|-------|
| Board | SK-AM62A-LP |
| SoC | Texas Instruments AM62A7 |
| Vendor | Texas Instruments |
| Kernel | 6.18.13 (TI vendor branch, SDK 12.00.00.07) |
| U-Boot | 2026.01 (TI vendor branch) |
| Console | ttyS2 (115200 8N1) |

---

## What This Fixes

### 1. SD Card Boot Hang — `vddshv_sdio` Regulator (Kernel Patch)

The upstream `k3-am62a7-sk.dts` defines the `vddshv_sdio` SDIO voltage regulator as
`regulator-gpio` (switchable 1.8V/3.3V). On the **LP variant**, this rail is hardwired
to 3.3V — there is no GPIO switch.

The `regulator-gpio` binding creates an unresolvable dependency chain:
fa00000.mmc → vddshv_sdio → main_gpio0 → vin-supply → pinctrl

This causes the kernel to spin in a deferred-probe loop, hanging at:
Waiting for root device PARTUUID=... deferred probe pending: platform: supplier regulator-5 not ready

**Fix:** Replace `regulator-gpio` with `regulator-fixed` at a constant 3.3V.

---

### 2. U-Boot A53 Build Failure — Missing Defconfig (U-Boot Patch)

Armbian's K3 board family expects `BOOTCONFIG="sk_am62a_lp_a53_defconfig"`, which does
not exist in the upstream U-Boot source tree. The nearest equivalent is
`am62ax_evm_a53_defconfig`, but a direct copy is insufficient.

Two critical options are required to prevent an assembler error in `crt0_64.S`:
CONFIG_HAS_CUSTOM_SYS_INIT_SP_ADDR=y CONFIG_CUSTOM_SYS_INIT_SP_ADDR=0x80480000

Without these, `SYS_INIT_SP_ADDR` expands to a compound C preprocessor expression
that the GNU assembler rejects as a non-constant operand.

---

## Repository Structure
armbian-sk-am62a-lp/ ├── README.md ├── config/ │ ├── boards/ │ │ └── sk-am62a-lp.conf ← Armbian board config │ └── kernel/ │ └── linux-k3-vendor.config ← Kernel config └── patch/ ├── kernel/ │ └── archive/ │ └── k3-6.18/ │ ├── 0002-am62a7-sk-lp-fix-vddshv_sdio-regulator-fixed-3v3.patch ← regulator fix │ └── board_sk-am62a-lp/ │ └── 0001-arm64-dts-ti-Add-SK-AM62A-LP-board-support.patch ← DTS addition └── u-boot/ └── u-boot-k3/ └── board_sk-am62a-lp/ └── 0001-configs-add-sk_am62a_lp_a53_defconfig.patch ← U-Boot defconfig


---

## Prerequisites

### System Requirements

- Ubuntu 22.04 LTS (native or WSL2)
- ~50 GB free disk space
- Internet access for source downloads

### Toolchain — GCC 13 Required for R5 SPL

```bash
sudo apt-get install gcc-13-arm-linux-gnueabi g++-13-arm-linux-gnueabi
sudo update-alternatives --install /usr/bin/arm-linux-gnueabi-gcc \
    arm-linux-gnueabi-gcc /usr/bin/arm-linux-gnueabi-gcc-13 100

Build Instructions
1. Clone Armbian
git clone https://github.com/armbian/build ~/Armbian/build
cd ~/Armbian/build

2. Apply This Repo's Files
REPO=~/armbian-sk-am62a-lp
BUILD=~/Armbian/build

mkdir -p $BUILD/config/boards
mkdir -p $BUILD/config/kernel
mkdir -p $BUILD/patch/kernel/archive/k3-6.18/board_sk-am62a-lp
mkdir -p $BUILD/patch/u-boot/u-boot-k3/board_sk-am62a-lp

cp $REPO/config/boards/sk-am62a-lp.conf              $BUILD/config/boards/
cp $REPO/config/kernel/linux-k3-vendor.config        $BUILD/config/kernel/
cp $REPO/patch/kernel/archive/k3-6.18/0002-*.patch   $BUILD/patch/kernel/archive/k3-6.18/
cp $REPO/patch/kernel/archive/k3-6.18/board_sk-am62a-lp/0001-*.patch \
                                                      $BUILD/patch/kernel/archive/k3-6.18/board_sk-am62a-lp/
cp $REPO/patch/u-boot/u-boot-k3/board_sk-am62a-lp/0001-*.patch \
                                                      $BUILD/patch/u-boot/u-boot-k3/board_sk-am62a-lp/

3. Build
cd ~/Armbian/build
./compile.sh \
  BOARD=sk-am62a-lp \
  BRANCH=vendor \
  BUILD_MINIMAL=yes \
  RELEASE=jammy \
  CLEAN_LEVEL=make

4. Verify DTB Before Flashing
rm -rf /tmp/dtb_check && mkdir -p /tmp/dtb_check
dpkg-deb -x \
  $(find ~/Armbian/build/output/debs -name "linux-dtb-vendor-k3*.deb" | head -1) \
  /tmp/dtb_check
dtc -I dtb -O dts \
  $(find /tmp/dtb_check -name "k3-am62a7-sk*.dtb") 2>/dev/null \
  | grep -A 10 "vddshv_sdio"

5. Flash
sudo dd if=output/images/Armbian_*.img of=/dev/sdX bs=4M status=progress conv=fsync
sync


Validated Build Artifacts


Armbian Patching Architecture Notes
Kernel patches — flat placement required
Armbian's Python patching script (lib/functions/compilation/patch/patching.py) does not pass
$BOARD

to the kernel patching stage, so
board_${BOARD}/

subdirectories are not scanned for kernel patches. Kernel patches must be placed flat:
patch/kernel/archive/k3-6.18/          ← kernel patches go HERE (flat)
patch/kernel/archive/k3-6.18/board_*/  ← NOT scanned during kernel build

U-Boot patches — board subdirectories work
U-Boot patching does support board subdirectories:
patch/u-boot/u-boot-k3/board_sk-am62a-lp/   ← U-Boot patches go here
