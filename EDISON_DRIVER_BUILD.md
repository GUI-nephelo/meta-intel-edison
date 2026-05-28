# Intel Edison 外部内核模块编译指南

> **本文档是 Agent 操作手册**。阅读本文档的 AI Agent 应按照以下流程自动完成驱动搜索、分支创建、workflow 配置、CI 触发、错误修复、artifact 交付的完整闭环。

## 目标环境

| 项目 | 值 |
|---|---|
| 内核版本 | `6.12.3-edison-acpi-preempt-rt` |
| vermagic | `6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload` |
| 编译器 | `x86_64-poky-linux-gcc (GCC) 13.3.0` |
| 设备架构 | x86_64 |
| 构建系统 | Yocto/OpenEmbedded (meta-intel-edison) |
| 内核配置来源 | `meta-intel-edison-bsp/recipes-kernel/linux/files/*.cfg` (40+片段) |
| GitHub 仓库 | `GUI-nephelo/meta-intel-edison` |
| Edison SSH | `ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i ~/edison.pem root@192.168.1.57` |
| 串口调试 | `python -m serial.tools.miniterm COM5 115200` |

## Agent 驱动编译完整流程

### 第一步：搜索与确认驱动信息

1. **确认目标驱动的 Kconfig 信息**：
   - 在 Linux 内核源码中搜索：`grep -r "tristate\|bool" <子系统>/Kconfig | grep -i <驱动名>`
   - 或在线搜索：https://cateee.net/linuxkernel/lkddb/ 搜索 CONFIG 名

2. **确认驱动的依赖模块**：
   - 查看对应 `Makefile` 中的 `obj-$(CONFIG_XXX)` 行，确认实际 `.ko` 文件名
   - **注意**：CONFIG 名和 .ko 文件名可能不一致（如 `CONFIG_DRM_GEM_DMA_HELPER` → `drm_dma_helper.ko`，不是 `drm_gem_dma_helper.ko`）
   - 用 `modinfo` 或 `readelf` 查看模块的 `depends` 字段

3. **检查 Edison 内核是否已内置**：
   - SSH 到 Edison：`cat /lib/modules/6.12.3-edison-acpi-preempt-rt/modules.builtin | grep <模块名>`
   - 如果已内置，无需编译外部模块

4. **确认所有子依赖的 CONFIG**：
   - 目标驱动可能需要额外的 CONFIG（如 TINYDRM 需要 `CONFIG_BACKLIGHT_CLASS_DEVICE=m`）
   - 在内核源码中搜索 `depends on` 行确认完整依赖链

### 第二步：创建编译分支

```bash
# 从模板分支或 master 创建新分支
gh api repos/GUI-nephelo/meta-intel-edison/git/refs/heads/master --jq '.object.sha'
# 用上面得到的 SHA 创建新分支
gh api repos/GUI-nephelo/meta-intel-edison/git/refs -X POST \
  -f ref="refs/heads/build/<驱动名>" -f sha="<master的SHA>"
```

### 第三步：生成并上传 Workflow 文件

使用下方 **模板 workflow**，填写 `<驱动特异部分>` 后上传：

```bash
# 将 workflow 内容 base64 编码后上传
node -e "const fs=require('fs');const c=fs.readFileSync('workflow.yml','utf8');const b=Buffer.from(c).toString('base64');const body={message:'Add <驱动名> driver build workflow',branch:'build/<驱动名>',content:b};fs.writeFileSync('commit-body.json',JSON.stringify(body));"
gh api repos/GUI-nephelo/meta-intel-edison/contents/.github/workflows/build-<驱动名>.yml -X PUT --input commit-body.json
```

### 第四步：触发并监控 CI

```bash
# 方式1：workflow_dispatch 从 setup 开始完整跑
gh workflow run build-<驱动名>.yml --ref build/<驱动名>

# 方式2：仅重跑验证（如果只改了验证逻辑）
gh workflow run build-<驱动名>.yml --ref build/<驱动名> -f start_from=verify

# 方式3：重跑编译+验证（如果改了 CONFIG）
gh workflow run build-<驱动名>.yml --ref build/<驱动名> -f start_from=build

# 监控
gh run list --repo GUI-nephelo/meta-intel-edison --workflow=build-<驱动名>.yml --limit=3
gh run view <RUN_ID> --repo GUI-nephelo/meta-intel-edison
```

### 第五步：错误修复（快速迭代）

如果 CI 失败：

1. **查看日志**：`gh run view --job=<JOB_ID> --repo GUI-nephelo/meta-intel-edison --log`
2. **对照下方 [常见失败排查](#常见失败排查) 表**
3. **修复 workflow 文件**：本地修改后用 GitHub API PUT 更新
4. **用 `start_from` 选择从哪个阶段重跑**，避免浪费已成功的阶段

### 第六步：下载并部署 Artifact

```bash
# 下载
gh run download <RUN_ID> --repo GUI-nephelo/meta-intel-edison --name <artifact名> --dir /tmp/driver-artifact

# 上传到 Edison
scp /tmp/driver-artifact/modules/*.ko root@192.168.1.57:/lib/modules/6.12.3-edison-acpi-preempt-rt/extra/

# SSH 到 Edison 加载
ssh root@192.168.1.57 "depmod -a 6.12.3-edison-acpi-preempt-rt; modprobe <模块名>"
```

### 第七步：验证

```bash
ssh root@192.168.1.57 "lsmod | grep <模块名>; dmesg | tail -10"
```

---

## 模板 Workflow（workflow_dispatch + start_from 三段式 CI）

> **这是最新的、经验证成功的模板**。所有新驱动都应基于此模板。
> 成功验证案例：DRM TINYDRM (build/drm-tinydrm 分支)

```yaml
name: Build <驱动描述> for Intel Edison

on:
  push:
    branches: [build/<驱动名>]
    paths:
      - '.github/workflows/build-<驱动名>.yml'
  workflow_dispatch:
    inputs:
      start_from:
        description: 'Start pipeline from stage (setup=full run, build=skip setup, verify=skip setup+build)'
        type: choice
        options:
          - setup
          - build
          - verify
        default: setup

permissions:
  contents: read

env:
  KERNEL_VERSION: "6.12.3"
  KERNEL_LOCALVERSION: "-edison-acpi-preempt-rt"
  EDISON_VERMAGIC: "6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload"
  # ===== 驱动特异配置 =====
  DRIVER_NAME: "<驱动名>"
  CACHE_SUFFIX: "<驱动缓存后缀>"  # 如 drm-tinydrm
  ARTIFACT_NAME: "<artifact名>"   # 如 drm-tinydrm-modules-v6.12.3

jobs:
  # ============================================================
  # Stage 1: Environment Setup
  # ============================================================
  setup-environment:
    if: ${{ github.event_name == 'push' || inputs.start_from == 'setup' }}
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    outputs:
      cache-key: ${{ steps.env-info.outputs.cache-key }}
    steps:
      - uses: actions/checkout@v4

      - name: Checkout upstream .cfg files
        uses: actions/checkout@v4
        with:
          repository: edison-fw/meta-intel-edison
          ref: master
          path: upstream-edison
          sparse-checkout: |
            meta-intel-edison-bsp/recipes-kernel/linux/files

      - name: Cache kernel source
        uses: actions/cache@v4
        id: cache-source
        with:
          path: ~/linux-source
          key: linux-src-v${{ env.KERNEL_VERSION }}-v1

      - name: Clone kernel source
        if: steps.cache-source.outputs.cache-hit != 'true'
        run: |
          rm -rf ~/linux-source
          # GitHub 镜像优先（更可靠），kernel.org 备用
          git clone --depth 1 --branch v${{ env.KERNEL_VERSION }} \
            https://github.com/gregkh/linux.git \
            ~/linux-source || \
          git clone --depth 1 --branch v${{ env.KERNEL_VERSION }} \
            https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git \
            ~/linux-source

      - name: Install build dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y flex bison libelf-dev libssl-dev

      - name: Print environment info
        id: env-info
        run: |
          echo "=== Environment ==="
          echo "OS: $(lsb_release -ds 2>/dev/null || cat /etc/os-release | grep PRETTY_NAME)"
          echo "GCC: $(gcc --version | head -1)"
          echo "Kernel: ${{ env.KERNEL_VERSION }}"
          echo "cache-key=linux-src-v${{ env.KERNEL_VERSION }}-v1" >> $GITHUB_OUTPUT

  # ============================================================
  # Stage 2: Kernel Build (config + compile)
  # ============================================================
  build-kernel:
    needs: setup-environment
    if: ${{ always() && (github.event_name == 'push' || inputs.start_from == 'setup' || inputs.start_from == 'build') }}
    runs-on: ubuntu-24.04
    timeout-minutes: 90
    outputs:
      build-cache-hit: ${{ steps.cache-build.outputs.cache-hit }}
    steps:
      - uses: actions/checkout@v4

      - name: Checkout upstream .cfg files
        uses: actions/checkout@v4
        with:
          repository: edison-fw/meta-intel-edison
          ref: master
          path: upstream-edison
          sparse-checkout: |
            meta-intel-edison-bsp/recipes-kernel/linux/files

      - name: Restore kernel source
        uses: actions/cache@v4
        id: cache-source
        with:
          path: ~/linux-source
          key: linux-src-v${{ env.KERNEL_VERSION }}-v1

      - name: Cache built kernel
        uses: actions/cache@v4
        id: cache-build
        with:
          path: ~/linux-source
          key: linux-build-v${{ env.KERNEL_VERSION }}-${{ env.CACHE_SUFFIX }}-v1

      - name: Install build dependencies
        if: steps.cache-build.outputs.cache-hit != 'true'
        run: |
          sudo apt-get update
          sudo apt-get install -y flex bison libelf-dev libssl-dev

      - name: Configure and build kernel
        if: steps.cache-build.outputs.cache-hit != 'true'
        run: |
          cd ~/linux-source
          CFG_DIR="$GITHUB_WORKSPACE/upstream-edison/meta-intel-edison-bsp/recipes-kernel/linux/files"

          echo "=== Build environment ==="
          gcc --version | head -1

          make defconfig

          MERGE_CFG=/tmp/edison-all.cfg
          cat /dev/null > $MERGE_CFG

          for cfg in \
            0001-enable-to-build-a-netboot-image.cfg \
            0002-enable-x2APIC.cfg \
            0003-enable-NETCONSOLE_DYNAMIC.cfg \
            0004-enable-STMMAC.cfg \
            0005-enable-IGB.cfg \
            0006-enable-IXGBE.cfg \
            0007-enable-USB_RTL8152.cfg \
            0008-disable-HPET.cfg \
            0009-disable-i915-DRM.cfg \
            0010-enable-X86_INTEL_MID.cfg \
            0011-enable-INPUT_SOC_BUTTON_ARRAY.cfg \
            0012-enable-I2C_HID-and-HID_MULTITOUCH.cfg \
            0013-DEBUG_SHIRQ-DEBUG_LOCKDEP.cfg \
            0014-enable-ACPI_DEBUG-and-ACPI_PROCFS_POWER.cfg \
            0015-enable-ACPI_TABLE_UPGRADE.cfg \
            0016-enable-X86_INTEL_LPSS-and-LPSS-drivers.cfg \
            0017-enable-MFD_INTEL_LPSS-drivers.cfg \
            0018-enable-INTEL_IDMA64-iDMA-64-bit.cfg \
            0020-enable-SPI_DW.cfg \
            0021-disable-HDA-audio.cfg \
            0022-enable-Intel-Quark-devices.cfg \
            0023-enable-DEBUG_GPIO.cfg \
            0024-enable-GPIO_DWAPB.cfg \
            0025-enable-GPIO_PCA953X.cfg \
            0029-enable-INTEL_IDLE.cfg \
            0030-enable-PUNIT_ATOM_DEBUG.cfg \
            0031-enable-REGULATOR.cfg \
            0032-enable-BRCMFMAC.cfg \
            0033-enable-BT_HCIUART_BCM.cfg \
            0034-enable-ADS7950.cfg \
            0035-enable-ACPI_CONFIGFS.cfg \
            0036-enable-SND_SST_ATOM_HIFI2_PLATFORM.cfg \
            0037-enable-SND_SOC_SOF-nocodec.cfg \
            0038-enable-PHY_TUSB1210.cfg \
            0039-enable-USB_CONFIGFS.cfg \
            0040-enable-INTEL_MRFLD_ADC.cfg \
            0041-enable-EXTCON_INTEL_MRFLD.cfg \
            0042-disable-LOCALVERSION_AUTO.cfg \
            preempt.cfg \
            ftdi_sio.cfg ch341.cfg smsc95xx.cfg bt_more.cfg \
            i2c_chardev.cfg configfs.cfg bridge.cfg leds.cfg bpf.cfg \
            btrfs.cfg sof_nocodec.cfg audio.cfg tun.cfg iio.cfg \
            cdc_eem.cfg namespaces.cfg; do
            if [ -f "$CFG_DIR/$cfg" ]; then
              cat "$CFG_DIR/$cfg" >> $MERGE_CFG
              echo "" >> $MERGE_CFG
            else
              echo "WARNING: $cfg not found"
            fi
          done

          scripts/kconfig/merge_config.sh -m .config $MERGE_CFG 2>&1 | tail -30

          # ===== 驱动特异 CONFIG 设置 =====
          # 在此添加目标驱动的 CONFIG：
          # scripts/config --module CONFIG_XXX_YOUR_DRIVER
          # 以及其依赖的 CONFIG：
          # scripts/config --module CONFIG_XXX_DEPENDENCY

          scripts/config --set-str CONFIG_LOCALVERSION "${{ env.KERNEL_LOCALVERSION }}"
          scripts/config --disable CONFIG_LOCALVERSION_AUTO

          yes '' | make oldconfig 2>&1 | tail -10

          echo "=== Key configs ==="
          grep -E 'CONFIG_PREEMPT_RT=|CONFIG_LOCALVERSION=|<驱动CONFIG>' .config

          # ===== 驱动 CONFIG 验证 =====
          # if ! grep -q 'CONFIG_XXX=m' .config; then
          #   echo "FATAL: XXX not =m"
          #   exit 1
          # fi

          echo "=== Building kernel ($(nproc) jobs) ==="
          make -j$(nproc) 2>&1 | tail -10

      - name: Build summary
        if: steps.cache-build.outputs.cache-hit == 'true'
        run: echo "Cache hit - skipped kernel build"

  # ============================================================
  # Stage 3: Verify + Package
  # ============================================================
  verify-and-package:
    needs: build-kernel
    if: ${{ always() }}
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - name: Restore built kernel
        uses: actions/cache@v4
        with:
          path: ~/linux-source
          key: linux-build-v${{ env.KERNEL_VERSION }}-${{ env.CACHE_SUFFIX }}-v1

      - name: Install verification tools
        run: sudo apt-get update && sudo apt-get install -y binutils kmod

      - name: Verify modules
        run: |
          cd ~/linux-source

          # ===== 驱动特异：目标 .ko 列表 =====
          KOS="
            path/to/your/driver.ko
          "

          echo "=== 1. File existence ==="
          for ko in $KOS; do
            if [ ! -f "$ko" ]; then
              echo "FATAL: $ko not found"
              exit 1
            fi
            echo "  FOUND: $ko"
          done

          echo ""
          echo "=== 2. vermagic check ==="
          for ko in $KOS; do
            VER=$(modinfo "$ko" 2>/dev/null | grep vermagic)
            echo "  $ko: $VER"
            if ! echo "$VER" | grep -qF "${{ env.EDISON_VERMAGIC }}"; then
              echo "  ERROR: vermagic mismatch!"
              exit 1
            fi
          done

          echo ""
          echo "=== 3. struct module size check (expect 0x4c0) ==="
          for ko in $KOS; do
            # readelf -SW columns: [Nr] Name Type Address Offset Size ...
            # PROGBITS is field $i, Size is field $(i+3) — NOT $(i+2) which is Offset
            MOD_SIZE=$(readelf -SW "$ko" | grep -A1 -E '\.gnu\.linkonce\.this_module PROGBITS|\.gnu\.linkonce\.this_module$' | grep -v rela | head -2 | tr '\n' ' ' | awk '{for(i=1;i<=NF;i++) if($i=="PROGBITS"){print $(i+3);break}}')
            MOD_SIZE_NORM=$(printf '%x' "0x$MOD_SIZE")
            echo "  $ko struct_module=0x$MOD_SIZE_NORM"
            if [ "0x$MOD_SIZE_NORM" != "0x4c0" ]; then
              echo "  ERROR: struct module size mismatch! Expected 0x4c0, got 0x$MOD_SIZE_NORM"
              exit 1
            fi
          done

          echo ""
          echo "=== All verifications passed ==="

      - name: Package modules
        run: |
          mkdir -p output/modules

          # ===== 驱动特异：复制目标 .ko =====
          # cp ~/linux-source/path/to/your/driver.ko output/modules/

          # ===== 驱动特异：复制依赖 .ko =====
          # for dep_ko in \
          #   path/to/dependency1.ko \
          #   path/to/dependency2.ko; do
          #   if [ -f ~/linux-source/$dep_ko ]; then
          #     cp ~/linux-source/$dep_ko output/modules/
          #     echo "  Packaged dependency: $dep_ko"
          #   else
          #     echo "  SKIP (not found or built-in): $dep_ko"
          #   fi
          # done

          ls -la output/modules/

      - name: Create install script
        run: |
          cat > output/install_on_edison.sh << 'INSTALLSCRIPT'
          #!/bin/bash
          set -e
          KVER="6.12.3-edison-acpi-preempt-rt"
          SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
          MODDIR="/lib/modules/$KVER/extra"

          echo "=== Installing $KVER modules ==="
          mkdir -p "$MODDIR"
          cp -v "$SCRIPT_DIR/modules/"*.ko "$MODDIR/"
          depmod -a "$KVER"

          echo ""
          echo "=== Loading modules ==="
          # ===== 驱动特异：依赖加载顺序 =====
          # for dep in dep1 dep2 dep3; do
          #   if [ -f "$MODDIR/${dep}.ko" ]; then
          #     modprobe "$dep" 2>/dev/null || insmod "$MODDIR/${dep}.ko"
          #   fi
          # done
          # modprobe <目标驱动> 2>/dev/null || insmod "$MODDIR/<目标驱动>.ko"

          echo ""
          echo "=== Verification ==="
          lsmod | grep -E '<驱动关键字>'
          INSTALLSCRIPT
          chmod +x output/install_on_edison.sh

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.ARTIFACT_NAME }}
          path: output/
          retention-days: 30
```

### 模板填写说明

Agent 在生成 workflow 时需要填写以下 `<驱动特异部分>`：

| 填写项 | 说明 | 示例 |
|---|---|---|
| `DRIVER_NAME` | 分支名后缀 | `drm-tinydrm` |
| `CACHE_SUFFIX` | cache key 后缀 | `drm-tinydrm` |
| `ARTIFACT_NAME` | artifact 下载名 | `drm-tinydrm-modules-v6.12.3` |
| **驱动特异 CONFIG** | Stage 2 中的 `scripts/config` 行 | `scripts/config --module CONFIG_TINYDRM_HX8357D` |
| **驱动特异 CONFIG 验证** | Stage 2 中的 grep 检查 | `grep -q 'CONFIG_TINYDRM_HX8357D=m'` |
| **目标 .ko 列表** | Stage 3 中的 KOS 变量 | `drivers/gpu/drm/tiny/hx8357d.ko` |
| **依赖 .ko 列表** | Stage 3 Package 步骤 | `drivers/video/backlight/backlight.ko` |
| **加载顺序** | Stage 3 install script | `backlight drm_dma_helper hx8357d` |

### start_from 策略效果

| start_from | 跳过阶段 | 执行阶段 | 耗时 |
|---|---|---|---|
| `setup`（默认） | 无 | 全部 | ~14 min |
| `build` | setup（用缓存源码） | build + verify | ~13 min |
| `verify` | setup + build（用缓存编译产物） | verify only | **~30 sec** |

**关键**：即使跳过 setup/build，对应 job 仍需在 `needs` 链中（用 `if: false` 跳过），因为 verify 的 `always()` 保证它一定执行。但通过 `if` 条件，setup 和 build 的 job 直接被跳过，**不计费**。

---

## 关键约束

### 1. vermagic 必须精确匹配

| vermagic 字段 | 对应 CONFIG | 说明 |
|---|---|---|
| `6.12.3-edison-acpi-preempt-rt` | `CONFIG_LOCALVERSION="-edison-acpi-preempt-rt"` + 关闭 `CONFIG_LOCALVERSION_AUTO` | 禁止 `+` 后缀 |
| `SMP` | `CONFIG_SMP=y` | 多核支持 |
| `preempt_rt` | `CONFIG_PREEMPT_RT=y`（不是 `CONFIG_PREEMPT_BUILD`） | RT内核，需 `CONFIG_EXPERT=y` 前置 |
| `mod_unload` | `CONFIG_MODULE_UNLOAD=y` | 模块卸载支持 |

**`CONFIG_PREEMPT_RT` 隐藏依赖链**：`CONFIG_EXPERT=y`（在 `sof_nocodec.cfg` 中）→ `CONFIG_PREEMPT_RT=y`（在 `preempt.cfg` 中）

### 2. struct module 大小（readelf 解析陷阱）

Edison 的 `struct module` 大小为 **0x4c0 (1216 bytes)**。

| 编译环境 | GCC | struct module 大小 | 结果 |
|---|---|---|---|
| `ubuntu-24.04`（固定） | 13.x | 0x4c0 | ✅ 匹配 |
| Edison 片上 (poky) | 13.3.0 | 0x4c0 | ✅ 基准 |
| WSL Debian 12 | 12.2.0 | 0x480 | ❌ 不匹配 |

**必须**：固定 `ubuntu-24.04`（不是 `ubuntu-latest`）。不需要额外固定 GCC 版本。

**readelf 正确解析**（已验证）：
```bash
# 列顺序：[Nr] Name Type Address Offset Size ...
# PROGBITS 所在位置 $i，Size 在 $(i+3)，Offset 在 $(i+2)
# ⚠️ $(i+2) 是 Offset 不是 Size！这是 3 次构建失败的根因
MOD_SIZE=$(readelf -SW "$ko" \
  | grep -A1 -E '\.gnu\.linkonce\.this_module PROGBITS|\.gnu\.linkonce\.this_module$' \
  | grep -v rela | head -2 | tr '\n' ' ' \
  | awk '{for(i=1;i<=NF;i++) if($i=="PROGBITS"){print $(i+3);break}}')
MOD_SIZE_NORM=$(printf '%x' "0x$MOD_SIZE")
```

### 3. 必须完整编译内核

仅 `make modules_prepare` 不够，缺少 `Module.symvers` 导致 modpost 阶段大量 undefined symbol 错误。

---

## 已验证驱动记录

| 驱动 | CONFIG | 分支 | 状态 | 备注 |
|---|---|---|---|---|
| rt2800usb | `CONFIG_RT2800USB=m` | `build/rt2800usb` | ✅ 成功 | 4个 .ko 文件 |
| snd-usb-audio | `CONFIG_SND_USB_AUDIO=m` | `snd-usb-audio` | ✅ 成功 | |
| cp210x | `CONFIG_USB_SERIAL_CP210X=m` | `build/cp210x` | ✅ 成功 | 仅 cp210x.ko，usbserial 已内置 |
| **DRM TINYDRM** | `CONFIG_TINYDRM_HX8357D=m` + `CONFIG_TINYDRM_ILI9486=m` | `build/drm-tinydrm` | ✅ 成功 | 三段式 CI + start_from，5个 .ko 文件 |
| general-drivers (批量) | 见下表 | `general_driver` | 🔄 编译中 | 1-Wire + WireGuard + nftables + VLAN + PWM |

### DRM TINYDRM 面板驱动 (2026-05-28) — 三段式 CI 参考实现

**模块依赖链**：
```
backlight.ko ← (CONFIG_BACKLIGHT_CLASS_DEVICE=m)
drm_dma_helper.ko ← (CONFIG_DRM_GEM_DMA_HELPER=m)
drm_mipi_dbi.ko ← (CONFIG_DRM_MIPI_DBI=m)
├── hx8357d.ko ← (CONFIG_TINYDRM_HX8357D=m)
└── ili9486.ko ← (CONFIG_TINYDRM_ILI9486=m)
```

**Edison lsmod 验证**：
```
ili9486        12288  0
hx8357d        12288  0
backlight      24576  2 ili9486,hx8357d
drm_dma_helper 16384  2 ili9486,hx8357d
drm_mipi_dbi   36864  2 ili9486,hx8357d
```

### general_driver 批量驱动 (2026-05-23)

| 驱动类别 | CONFIG | 模块列表 |
|---|---|---|
| **1-Wire** | `CONFIG_W1=m` 等 | w1-gpio, w1-therm, w1-slave-ds2408 等 |
| **WireGuard** | `CONFIG_WIREGUARD=m` | ❌ 内核配置依赖链不满足 |
| **nftables** | `CONFIG_NF_TABLES=m` 等 | nf_tables, nft_compat, nft_ct 等 |
| **802.1Q VLAN** | `CONFIG_VLAN_8021Q=m` | 8021q |
| **PWM** | `CONFIG_PWM_GPIO=m`, `CONFIG_PWM_PCA9685=m` | pwm-gpio, pwm-pca9685 |

---

## 常见失败排查

| 错误 | 原因 | 解决 |
|---|---|---|
| `Exec format error` | struct module 大小不匹配 | 固定 `ubuntu-24.04`，验证 readelf 输出为 0x4c0 |
| `vermagic disagrees` | 版本字符串不匹配 | 检查 LOCALVERSION 和 LOCALVERSION_AUTO |
| struct module 报 0x1640 或 0x17c0 | readelf 取了 Offset(`$(i+2)`) 而非 Size(`$(i+3)`) | 用模板中的正确 readelf 解析 |
| `Unknown symbol xxx` | 缺少依赖 .ko 模块 | 查看内核 Makefile 确认依赖模块名，添加到 Package 步骤 |
| `Unknown symbol devm_of_find_backlight` | 缺少 `backlight.ko` | 启用 `CONFIG_BACKLIGHT_CLASS_DEVICE=m` |
| 模块名与 CONFIG 名不一致 | Makefile 中的 obj- 变量名 | 查看 `Makefile` 确认实际 .ko 文件名（如 `drm_dma_helper.ko`）|
| `undefined symbol` (modpost) | 缺少 Module.symvers | 必须先做完整 `make`，不能只 `modules_prepare` |
| `CONFIG_PREEMPT_RT` 被 oldconfig 丢弃 | 缺少 CONFIG_EXPERT=y | 确保 sof_nocodec.cfg 已合并 |
| vermagic 是 `preempt` 而非 `preempt_rt` | PREEMPT_BUILD 被选中 | 需要完整 .cfg 片段合并 |
| `exports duplicate symbol` | 依赖模块已内置 | 检查 `modules.builtin`，只加载非内置的 .ko |
| git clone 502 | git.kernel.org 故障 | GitHub 镜像优先（模板已实现） |
| artifact 为空 | upload-artifact 的 path 用了相对路径 | 使用绝对路径 `/home/runner/linux-source/output/` 或 `output/`（相对于 GITHUB_WORKSPACE）|
| WireGuard 无法编译 | 依赖 crypto 子系统 | 需启用完整 crypto 依赖链或用 wireguard-go 用户态替代 |

---

## 经验教训（Agent 必读）

### 编译错误模式总结

1. **readelf 偏移量 bug**（3 次构建失败）：`$(i+2)` 取 Offset 而非 Size。所有旧 workflow（rt2800usb/snd-usb-audio/cp210x/general_driver）都有此 bug。模板中已修正为 `$(i+3)`。

2. **模块名不一致**：`CONFIG_DRM_GEM_DMA_HELPER` → Makefile 中 `obj-m += drm_dma_helper.o` → 实际文件 `drm_dma_helper.ko`。**永远先查 Makefile 确认 .ko 文件名**。

3. **隐式依赖**：TINYDRM 驱动需要 `backlight.ko`，但 `CONFIG_TINYDRM_HX8357D` 的 Kconfig `depends on` 没有显式列出。**必须检查 `modinfo` 的 depends 字段和 `nm -D` 的 undefined symbols**。

4. **ubuntu-latest 不稳定**：runner image 会自动升级，导致环境不一致。**必须固定 `ubuntu-24.04`**。

5. **cache key 设计**：build cache key 不能包含 workflow 文件 hash（改验证脚本会导致 cache miss 重跑编译）。三个阶段各自独立 key。

6. **已有模块的内置问题**：`usbserial`、`drm_kms_helper` 等可能已 built-in 到 Edison 内核。加载外部 .ko 会 `exports duplicate symbol`。**先查 `modules.builtin`**。

### 调试时间优化策略

| 调试场景 | 使用 start_from | 预计耗时 |
|---|---|---|
| 改验证脚本（readelf/vermagic 检查） | `verify` | **~30s** |
| 改打包逻辑（依赖 .ko 列表） | `verify` | **~30s** |
| 改驱动 CONFIG（加依赖） | `build` | **~13min** |
| 改内核版本/环境 | `setup` | **~14min** |

---

## SSH 连接

```bash
ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i ~/edison.pem root@192.168.1.57
python -m serial.tools.miniterm COM5 115200
```

## 验证清单

- [ ] `modinfo module.ko | grep vermagic` 输出 `6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload`
- [ ] readelf struct module size 为 `0x4c0`（用正确的 `$(i+3)` 解析）
- [ ] 目标模块未在 `modules.builtin` 中列出
- [ ] `modprobe module` 无报错
- [ ] `lsmod | grep module` 已加载
- [ ] `dmesg` 无相关错误
