# Intel Edison 外部内核模块编译指南

## 目标环境

| 项目 | 值 |
|---|---|
| 内核版本 | `6.12.3-edison-acpi-preempt-rt` |
| vermagic | `6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload` |
| 编译器 | `x86_64-poky-linux-gcc (GCC) 13.3.0` |
| 设备架构 | x86_64 |
| 构建系统 | Yocto/OpenEmbedded (meta-intel-edison) |
| 内核配置来源 | `meta-intel-edison-bsp/recipes-kernel/linux/files/*.cfg` (40+片段) |

## 关键约束

### 1. vermagic 必须精确匹配

Edison 已有模块的 vermagic 为：

```
6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload
```

vermagic 由以下内核配置项决定：

| vermagic 字段 | 对应 CONFIG | 说明 |
|---|---|---|
| `6.12.3-edison-acpi-preempt-rt` | `CONFIG_LOCALVERSION="-edison-acpi-preempt-rt"` + 关闭 `CONFIG_LOCALVERSION_AUTO` | 禁止 `+` 后缀 |
| `SMP` | `CONFIG_SMP=y` | 多核支持 |
| `preempt_rt` | `CONFIG_PREEMPT_RT=y`（不是 `CONFIG_PREEMPT_BUILD`） | RT内核，需要 `CONFIG_EXPERT=y` 前置 |
| `mod_unload` | `CONFIG_MODULE_UNLOAD=y` | 模块卸载支持 |
| 无 `modversions` | `CONFIG_MODVERSIONS` 未设置 | 无符号版本校验 |

**`CONFIG_PREEMPT_RT` 的隐藏依赖链**：

```
CONFIG_EXPERT=y          ← sof_nocodec.cfg 中包含此项
  └── CONFIG_PREEMPT_RT=y  ← preempt.cfg 中包含此项
```

如果只设 `CONFIG_PREEMPT_RT=y` 而不启用 `CONFIG_EXPERT=y`，`make oldconfig` 会静默丢弃 `PREEMPT_RT`，导致 vermagic 变成 `preempt` 而非 `preempt_rt`。

### 2. struct module 大小（readelf 解析陷阱）

内核加载模块时校验 `.gnu.linkonce.this_module` section 大小。Edison 的值为 **0x4c0 (1216 bytes)**。

影响 `struct module` 大小的因素：
- **GCC 版本**：不同版本对结构体 padding/alignment 不同
- **内核配置**：某些 CONFIG 会向 `struct module` 增加字段

| 编译环境 | GCC | struct module 大小 | 结果 |
|---|---|---|---|
| GitHub Actions ubuntu-24.04 (固定) | 13.x | 0x4c0 | ✅ 匹配 |
| Edison 片上 (poky) | 13.3.0 | 0x4c0 | ✅ 基准 |
| WSL Debian 12 | 12.2.0 | 0x480 | ❌ 不匹配 |

**⚠️ readelf 解析根因（2026-05-27 确认）**：

之前 3 次 DRM workflow 失败报告"struct module = 0x1640"是**误报**。根因是 readelf 字段偏移错误：

```
readelf -SW 输出格式：
[Nr] Name                          Type     Address           Offset   Size   ...
[31] .gnu.linkonce.this_module     PROGBITS 0000000000000000  001640   0004c0 ...
                                                            ↑ Offset  ↑ Size
```

- `$(i+2)` 取到的是 **Offset**（001640），不是 Size
- `$(i+3)` 才是 **Size**（0004c0 = 0x4c0 ✅）

**结论：ubuntu-24.04 默认 GCC 即可产出正确的 0x4c0 struct module，无需固定 GCC 版本。** 但必须固定 `ubuntu-24.04`（而非 `ubuntu-latest`）以保证环境一致性。

> 2026-05-27 经验：之前怀疑 ubuntu-latest GCC 升级导致 struct module 变大，实际是 readelf 解析 bug。GCC-13 pinning 不需要，但固定 runner image 版号是必要的。

### 3. 必须完整编译内核

仅 `make modules_prepare` 不够，缺少 `Module.symvers` 导致 modpost 阶段大量 undefined symbol 错误。

正确流程：
```
make defconfig → merge .cfg → make -j$(nproc) → make M=xxx
```

## GitHub Actions 工作流模板

### 仓库结构(https://github.com/GUI-nephelo/meta-intel-edison)

```
meta-intel-edison/
├── .github/workflows/
│   ├── build-rt2800usb.yml      # rt2800usb 驱动 (成功验证)
│   ├── build-snd-usb-audio.yml  # snd-usb-audio 驱动 (成功验证)
│   └── build-cp210x.yml         # cp210x USB串口驱动 (成功验证)
└── meta-intel-edison-bsp/
    └── recipes-kernel/linux/files/
        ├── preempt.cfg          # CONFIG_PREEMPT_RT=y
        ├── sof_nocodec.cfg      # CONFIG_EXPERT=y (关键!)
        ├── audio.cfg
        └── ... (40+ .cfg 片段)
```

### 工作流核心步骤

```yaml
jobs:
  build:
    runs-on: ubuntu-latest    # 必须固定 GCC-13（见下方）
    timeout-minutes: 90

    steps:
      # 1. Checkout 自己的 fork
      - uses: actions/checkout@v4

      # 2. Checkout upstream 获取 .cfg 片段（关键！）
      - uses: actions/checkout@v4
        with:
          repository: edison-fw/meta-intel-edison
          ref: master
          path: upstream-edison
          sparse-checkout: |
            meta-intel-edison-bsp/recipes-kernel/linux/files

      # 3. Cache hit 策略：缓存完整编译产物（源码+对象文件）
      #    key 包含 workflow 文件 hash → 改 workflow 时自动重建
      #    只改验证逻辑不改 workflow → cache hit → 跳过编译
      - uses: actions/cache@v4
        id: cache-kernel-build
        with:
          path: ~/linux-source
          key: linux-build-v6.12.3-<module>-v1-${{ hashFiles('.github/workflows/<this-file>.yml') }}
          restore-keys: |
            linux-build-v6.12.3-<module>-v1-

      # 4. 安装依赖 + 固定 GCC-13（关键！ubuntu-latest 已升级 GCC-14）
      - name: Install dependencies and pin GCC-13
        if: steps.cache-kernel-build.outputs.cache-hit != 'true'
        run: |
          sudo apt-get update
          sudo apt-get install -y flex bison libelf-dev libssl-dev gcc-13
          sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100
          sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-13 100
          gcc --version

      # 4b. Cache hit 时只需 GCC（验证步骤要用）
      - name: Install GCC-13 (cache hit)
        if: steps.cache-kernel-build.outputs.cache-hit == 'true'
        run: |
          sudo apt-get update && sudo apt-get install -y gcc-13
          sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100
          gcc --version

      # 5. 克隆+配置+编译（cache miss 时执行）
      - name: Build kernel (clone + config + make)
        if: steps.cache-kernel-build.outputs.cache-hit != 'true'
        run: |
          git clone --depth 1 --branch v6.12.3 \
            https://github.com/gregkh/linux.git ~/linux-source || \
          git clone --depth 1 --branch v6.12.3 \
            https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git \
            ~/linux-source

          cd ~/linux-source
          make defconfig

          # 合并所有 .cfg 片段
          CFG_DIR="$GITHUB_WORKSPACE/upstream-edison/meta-intel-edison-bsp/recipes-kernel/linux/files"
          MERGE_CFG=/tmp/edison-all.cfg
          cat /dev/null > $MERGE_CFG

          for cfg in \
            0001-enable-to-build-a-netboot-image.cfg \
            /* ... 列出所有 .cfg ... */ \
            preempt.cfg \
            sof_nocodec.cfg audio.cfg /* 等 */; do
            cat "$CFG_DIR/$cfg" >> $MERGE_CFG
            echo "" >> $MERGE_CFG
          done

          scripts/kconfig/merge_config.sh -m .config $MERGE_CFG

          # 添加目标模块的 CONFIG
          scripts/config --module CONFIG_XXX_YOUR_MODULE
          scripts/config --set-str CONFIG_LOCALVERSION "-edison-acpi-preempt-rt"
          scripts/config --disable CONFIG_LOCALVERSION_AUTO
          yes '' | make oldconfig

          # 完整编译（约 60 分钟）
          make -j$(nproc) 2>&1 | tail -10

      # 6. 验证（每次都执行，cache hit 时跳过编译直接到这步）
      - name: Verify and collect modules
        run: |
          cd ~/linux-source
          # .ko 存在性检查
          # vermagic 完整匹配检查
          # struct module size 检查（GCC-13 → 0x4c0）
```

      # 10. 打包上传
      - uses: actions/upload-artifact@v4
        with:
          name: your-module-v6.12.3
          path: output/
```

### .cfg 片段完整列表

以下是 Edison 内核配置的所有 .cfg 文件，**必须全部合并**：

```
0001-enable-to-build-a-netboot-image.cfg
0002-enable-x2APIC.cfg
0003-enable-NETCONSOLE_DYNAMIC.cfg
0004-enable-STMMAC.cfg
0005-enable-IGB.cfg
0006-enable-IXGBE.cfg
0007-enable-USB_RTL8152.cfg
0008-disable-HPET.cfg
0009-disable-i915-DRM.cfg
0010-enable-X86_INTEL_MID.cfg
0011-enable-INPUT_SOC_BUTTON_ARRAY.cfg
0012-enable-I2C_HID-and-HID_MULTITOUCH.cfg
0013-DEBUG_SHIRQ-DEBUG_LOCKDEP.cfg
0014-enable-ACPI_DEBUG-and-ACPI_PROCFS_POWER.cfg
0015-enable-ACPI_TABLE_UPGRADE.cfg
0016-enable-X86_INTEL_LPSS-and-LPSS-drivers.cfg
0017-enable-MFD_INTEL_LPSS-drivers.cfg
0018-enable-INTEL_IDMA64-iDMA-64-bit.cfg
0020-enable-SPI_DW.cfg
0021-disable-HDA-audio.cfg
0022-enable-Intel-Quark-devices.cfg
0023-enable-DEBUG_GPIO.cfg
0024-enable-GPIO_DWAPB.cfg
0025-enable-GPIO_PCA953X.cfg
0029-enable-INTEL_IDLE.cfg
0030-enable-PUNIT_ATOM_DEBUG.cfg
0031-enable-REGULATOR.cfg
0032-enable-BRCMFMAC.cfg
0033-enable-BT_HCIUART_BCM.cfg
0034-enable-ADS7950.cfg
0035-enable-ACPI_CONFIGFS.cfg
0036-enable-SND_SST_ATOM_HIFI2_PLATFORM.cfg
0037-enable-SND_SOC_SOF-nocodec.cfg   ← 含 CONFIG_EXPERT=y
0038-enable-PHY_TUSB1210.cfg
0039-enable-USB_CONFIGFS.cfg
0040-enable-INTEL_MRFLD_ADC.cfg
0041-enable-EXTCON_INTEL_MRFLD.cfg
0042-disable-LOCALVERSION_AUTO.cfg
preempt.cfg                           ← 含 CONFIG_PREEMPT_RT=y
ftdi_sio.cfg ch341.cfg smsc95xx.cfg bt_more.cfg
i2c_chardev.cfg configfs.cfg bridge.cfg leds.cfg bpf.cfg
btrfs.cfg sof_nocodec.cfg audio.cfg tun.cfg iio.cfg
cdc_eem.cfg namespaces.cfg
```

## 新驱动编写流程

### 1. 确认目标模块的 CONFIG 依赖

```bash
# 在内核源码中查找模块对应的 CONFIG
grep -r "tristate\|bool" sound/usb/Kconfig | grep -i "usb.audio"
```

### 2. 创建 workflow

复制成功的 rt2800usb 或 snd-usb-audio workflow文件，修改：
- `CONFIG_XXX_YOUR_MODULE` 为目标模块
- `.ko` 收集的正则表达式
- 依赖模块列表
- artifact 名称

也可参考 `build-general-drivers.yml`（分支 `general_driver`）的**批量编译**模式：一次 `make -j$(nproc)` 内核后，用多个 `make M=xxx` 编译不同驱动，共享同一个内核源码树。

### 3. 上传到专有 branch

使用master分支创建为新的专有驱动编译分支build/xxx
并将生成好的workflow文件以github api（mcp、gh cli均可）的方式上传至该分支

### 4. 下载 artifact 并安装

```bash
# SCP 到 Edison
scp modules/*.ko root@192.168.1.57:/tmp/

# 在 Edison 上安装
mkdir -p /lib/modules/6.12.3-edison-acpi-preempt-rt/extra/
cp /tmp/*.ko /lib/modules/6.12.3-edison-acpi-preempt-rt/extra/
depmod -a 6.12.3-edison-acpi-preempt-rt
modprobe your-module
```

## 已验证驱动记录

| 驱动 | CONFIG | 分支 | 状态 | 备注 |
|---|---|---|---|---|
| rt2800usb | `CONFIG_RT2800USB=m` | `build/rt2800usb` | ✅ 成功 | 4个 .ko 文件 |
| snd-usb-audio | `CONFIG_SND_USB_AUDIO=m` | `snd-usb-audio` | ✅ 成功 | |
| cp210x | `CONFIG_USB_SERIAL_CP210X=m` | `build/cp210x` | ✅ 成功 | 仅 cp210x.ko，usbserial 已内置 |
| **DRM TINYDRM** | `CONFIG_TINYDRM_HX8357D=m` + `CONFIG_TINYDRM_ILI9486=m` | `build/drm-tinydrm` | ✅ 成功 | 三段式 CI，5个 .ko 文件 |
| general-drivers (批量) | 见下表 | `general_driver` | 🔄 编译中 | 1-Wire + WireGuard + nftables + VLAN + PWM |

### DRM TINYDRM 面板驱动 (2026-05-28)

使用三段式 CI（setup → build → verify+package），首次成功验证了该架构。

**模块依赖链**：
```
backlight.ko ← (CONFIG_BACKLIGHT_CLASS_DEVICE=m)
drm_dma_helper.ko ← (CONFIG_DRM_GEM_DMA_HELPER=m，实际文件名不是 drm_gem_dma_helper.ko!)
drm_mipi_dbi.ko ← (CONFIG_DRM_MIPI_DBI=m)
└── hx8357d.ko ← (CONFIG_TINYDRM_HX8357D=m)
└── ili9486.ko ← (CONFIG_TINYDRM_ILI9486=m)
```

**踩坑记录**：
1. **readelf 解析 bug**：`$(i+2)` 取的是 Offset 不是 Size，应为 `$(i+3)`。导致 3 次构建误报 struct module size 不匹配
2. **模块名陷阱**：`CONFIG_DRM_GEM_DMA_HELPER` 产出的模块叫 `drm_dma_helper.ko`（不是 `drm_gem_dma_helper.ko`）。Makefile 中 `obj-$(CONFIG_DRM_GEM_DMA_HELPER) += drm_dma_helper.o`
3. **backlight 依赖**：TINYDRM 驱动需要 `devm_of_find_backlight`，来自 `CONFIG_BACKLIGHT_CLASS_DEVICE=m`。Edison 原始内核未启用，需显式设置
4. **drm_kms_helper**：在 Edison 配置中为 =y（built-in），不需要外部 .ko

### general_driver 批量驱动 (2026-05-23)

使用单个 workflow 批量编译多个驱动，一次内核编译 + 多个 `make M=xxx`。

| 驱动类别 | CONFIG | 模块列表 |
|---|---|---|
| **1-Wire** | `CONFIG_W1=m` 等 | `w1-gpio`, `w1-therm`, `w1-slave-ds2408`, `w1-slave-ds2413`, `w1-slave-ds2423`, `w1-slave-eeprom`, `w1-slave-smem`, `w1-master-ds2490` |
| **WireGuard** | `CONFIG_WIREGUARD=m` | ~~wireguard~~ ❌ 内核配置依赖链不满足，无法在此编译 |
| **nftables** | `CONFIG_NF_TABLES=m` 等 | `nf_tables`, `nft_compat`, `nft_ct`, `nft_counter`, `nft_limit`, `nft_nat`, `nft_masq`, `nft_queue`, `nft_log`, `nft_reject`, `nft_hash`, `nft_fib`, `nft_objref`, `nft_bridge`, `nft_arp` |
| **802.1Q VLAN** | `CONFIG_VLAN_8021Q=m` | `8021q` |
| **PWM** | `CONFIG_PWM_GPIO=m`, `CONFIG_PWM_PCA9685=m` | `pwm-gpio`, `pwm-pca9685` |

**已知问题**：
- upload-artifact 的 `path` 必须用绝对路径 `/home/runner/linux-source/output/`，相对路径 `linux-source/output/` 找不到文件导致 artifact 为空
- **WireGuard 无法编译**：`CONFIG_WIREGUARD` 依赖 `CONFIG_CRYPTO_CHACHA20POLY1305`、`CONFIG_CRYPTO_BLAKE2S`、`CONFIG_CRYPTO_CURVE25519` 等加密子系统，Edison 内核配置未启用这些选项，`make oldconfig` 会静默丢弃。需用 `wg-quick` 用户态工具 + `wireguard-go` 用户态实现替代，或手动启用完整 crypto 依赖链后重新编译内核。

### cp210x 构建要点 (2026-05-22)

- **usbserial 是内置模块**：Edison 内核已将 `usbserial` 编译为 built-in（见 `modules.builtin`），加载外部 `usbserial.ko` 会报 `exports duplicate symbol` 错误。只需编译并加载 `cp210x.ko`。
- **内核克隆源**：`git.kernel.org` 偶尔返回 502，workflow 中应优先使用 GitHub 镜像 `https://github.com/gregkh/linux.git`，kernel.org 作为 fallback。
- **~~struct module size 检查可省略~~ → 已过时**：2026-05 ubuntu-latest 升级 GCC-14 后 struct module 变为 0x1640，必须固定 GCC-13。

## 常见失败排查

| 错误 | 原因 | 解决 |
|---|---|---|
| `Exec format error` | struct module 大小不匹配 | 确认 GCC 版本，固定 GCC-13，验证 readelf 输出为 0x4c0 |
| `vermagic disagrees` | 版本字符串不匹配 | 检查 LOCALVERSION 和 LOCALVERSION_AUTO |
| `section size must match` | GCC 版本与内核编译器不一致 | 固定 GCC-13（`apt install gcc-13` + `update-alternatives`）|
| struct module 报 0x1640 或 0x17c0 | readelf 解析取了 Offset 而非 Size（`$(i+2)` → 应为 `$(i+3)`） | 修正 awk 字段偏移 |
| `Unknown symbol drm_gem_dma_dumb_create` | 缺少 `drm_dma_helper.ko` 依赖模块 | 安装 `drm_dma_helper.ko`（注意文件名不是 `drm_gem_dma_helper.ko`）|
| `Unknown symbol devm_of_find_backlight` | 缺少 `backlight.ko` 依赖模块 | 启用 `CONFIG_BACKLIGHT_CLASS_DEVICE=m` 并安装 `backlight.ko` |
| 依赖 .ko 打包缺失 | Package 步骤中模块路径/文件名错误 | 查看内核 Makefile 确认实际 .ko 文件名（如 `drm_dma_helper.ko` 而非 `drm_gem_dma_helper.ko`）|
| `undefined symbol` | 缺少 Module.symvers | 必须先做完整 `make`，不能只 `modules_prepare` |
| `CONFIG_PREEMPT_RT` 被 oldconfig 丢弃 | 缺少 CONFIG_EXPERT=y | 确保 sof_nocodec.cfg 已合并 |
| vermagic 是 `preempt` 而非 `preempt_rt` | CONFIG_PREEMPT_BUILD 被选中而非 PREEMPT_RT | 需要完整 .cfg 片段合并 |
| `exports duplicate symbol` | 依赖模块已内置到内核 | 检查 `modules.builtin`，只加载非内置的 .ko |
| git clone 502 | git.kernel.org 临时故障 | 使用 GitHub 镜像作为首选源 |
| struct module size 检查误报 `0x0004c0 != 0x4c0` | `readelf -SW` 输出带前导零 + 长段名跨行，导致字段提取错误 | 用 `grep -A1` + `tr '\n' ' '` + `awk` 按关键字定位 + `printf '%x'` 归一化（见下方说明） |

### struct module size 检查的 readelf 输出解析陷阱

`readelf -SW` 解析 `.gnu.linkonce.this_module` 段的 Size 字段时有 **两个陷阱**：

1. **前导零**：Size 输出为 `0004c0` 而非 `4c0`，字符串 `"0x0004c0" != "0x4c0"` 永远为 true
2. **段名跨行**：`.gnu.linkonce.this_module` 名称较长时，readelf 将一行拆成两行输出，`grep | awk` 只拿到第一行，Size 字段丢失（或混入换行导致 `printf: invalid hex number`）

实际 `readelf -SW` 输出：
```
[23] .gnu.linkonce.this_module
                                        PROGBITS        0000000000000000 0004c0 000030 00  WA  0   0  40
```

**错误写法**（两次构建均因此失败）：
```bash
# 错误1: grep 只拿到第一行，$6 为空
MOD_SIZE=$(readelf -SW "$ko" | grep '.gnu.linkonce.this_module' | awk '{print $6}')
# 错误2: 段名跨行时 $MOD_SIZE 含换行符，printf 报 invalid hex number
MOD_SIZE_NORM=$(printf '%x' "0x$MOD_SIZE")
```

**正确写法**（`grep -A1` 取下一行 + `grep -v rela` 排除重定位段 + `tr` 合并行 + `awk` 按 PROGBITS 关键字定位 Size + `printf` 去前导零）：
```bash
MOD_SIZE=$(readelf -SW "$ko" \
  | grep -A1 -E '\.gnu\.linkonce\.this_module PROGBITS|\.gnu\.linkonce\.this_module$' \
  | grep -v rela | head -2 | tr '\n' ' ' \
  | awk '{for(i=1;i<=NF;i++) if($i=="PROGBITS"){print $(i+2);break}}')
MOD_SIZE_NORM=$(printf '%x' "0x$MOD_SIZE")
if [ "0x$MOD_SIZE_NORM" != "0x4c0" ]; then
  exit 1
fi
```

## SSH 连接

```bash
# 标准连接
ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i ~/edison.pem root@192.168.1.57

# 串口调试 (COM5, 115200)
python -m serial.tools.miniterm COM5 115200
```

## 验证清单

- [ ] `modinfo module.ko | grep vermagic` 输出 `6.12.3-edison-acpi-preempt-rt SMP preempt_rt mod_unload`
- [ ] `readelf -SW module.ko | grep this_module` size 为 `0x4c0`（必须用 GCC-13 编译）
- [ ] 目标模块未在 `modules.builtin` 中列出（否则无需编译外部模块）
- [ ] `modprobe module` 或 `insmod module.ko` 无报错
- [ ] `lsmod | grep module` 已加载
- [ ] `dmesg` 无相关错误

## Cache Hit 策略（避免重复编译）

### 背景

完整内核编译约 60 分钟。如果验证步骤失败（vermagic/struct module 检查），修改后重跑需要重新编译。GitHub Actions 不支持"从某步断点续跑"。

### 方案

缓存整个 `~/linux-source` 目录（含源码+编译产物）。Cache key 包含 workflow 文件的 hash — 修改 workflow 时自动重建，只改验证脚本时 cache hit 跳过编译。

### 实现要点

```yaml
# cache key 包含 workflow 文件 hash → 改 workflow 自动 cache miss
- uses: actions/cache@v4
  id: cache-kernel-build
  with:
    path: ~/linux-source
    key: linux-build-v6.12.3-<module>-v1-${{ hashFiles('.github/workflows/<this-file>.yml') }}
    restore-keys: |
      linux-build-v6.12.3-<module>-v1-

# clone + config + make 合并为一步（cache miss 时执行）
- name: Build kernel
  if: steps.cache-kernel-build.outputs.cache-hit != 'true'
  run: |
    git clone ... && cd ~/linux-source
    make defconfig && merge configs && make -j$(nproc)

# 验证步骤永远执行（cache hit 时直接用缓存结果）
- name: Verify
  run: cd ~/linux-source && ...
```

### 效果

| 场景 | cache 结果 | 耗时 |
|---|---|---|
| 首次运行 / 修改了 workflow | cache miss | ~13 分钟（完整编译）|
| 只改验证脚本，重跑 | cache hit | ~1 分钟（跳过编译）|
| GCC 版本问题需要调试 | cache hit | ~1 分钟 |

### 注意事项

- GitHub Actions cache 单条目上限 10GB，内核编译产物约 2-3GB，在范围内
- Cache 7 天未被任何 workflow 访问会自动清理
- `restore-keys` 前缀匹配作为降级策略：即使 key 不完全匹配，也能恢复最近一次编译结果

## 三段式 CI 架构

### 设计目标

构建一个可复用的、无错误的驱动编译 GitHub Actions 仓库。每次新增驱动时只需修改驱动相关的 CONFIG 和 .ko 路径，不关心环境和编译细节。

### 架构

```
┌─────────────────────────────────────────────────────────┐
│  Stage 1: setup-environment                             │
│  ┌─────────────────────────────────────────────────────┐│
│  │ 固定 ubuntu-24.04 + 安装依赖 + 缓存内核源码          ││
│  │ 输出：cached ~/linux-source (源码)                    ││
│  │ 触发条件：手动 / 源码缓存 miss                        ││
│  └─────────────────────────────────────────────────────┘│
│                          ↓                               │
│  Stage 2: build-kernel                                  │
│  ┌─────────────────────────────────────────────────────┐│
│  │ 恢复源码 → merge .cfg → 设置驱动 CONFIG → make      ││
│  │ 输出：cached ~/linux-source (含编译产物)              ││
│  │ 触发条件：构建缓存 miss（驱动配置变化）                ││
│  │ ★ 每个驱动的差异化配置在此完成                        ││
│  └─────────────────────────────────────────────────────┘│
│                          ↓                               │
│  Stage 3: verify-and-package                            │
│  ┌─────────────────────────────────────────────────────┐│
│  │ 恢复编译产物 → vermagic + struct module 验证          ││
│  │ → 收集 .ko → 生成安装脚本 → 上传 artifact            ││
│  │ 输出：最终 Artifact（可下载）                         ││
│  │ 触发条件：每次都执行                                  ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 关键设计决策

| 决策 | 理由 |
|---|---|
| 固定 `ubuntu-24.04`（非 `ubuntu-latest`） | 避免 runner image 升级导致环境不一致 |
| 三段式 jobs（非单 job 多 step） | 每段可独立重跑，失败的阶段无需从头再来 |
| Stage 2 缓存 key 不含 workflow hash | 改验证脚本不影响编译缓存，Stage 3 失败可秒级重试 |
| 驱动差异化集中在 Stage 2 | 新增驱动只需改 CONFIG + .ko 路径 |
| env 全局变量 | `KERNEL_VERSION`、`KERNEL_LOCALVERSION`、`EDISON_VERMAGIC` 集中管理 |

### 缓存策略（修正版）

```
Stage 1 → cache key: linux-src-v{KERNEL_VERSION}-v1
  ↓ (不变则 hit)
Stage 2 → cache key: linux-build-v{KERNEL_VERSION}-{driver}-v1
  ↓ (不变则 hit → 跳过编译)
Stage 3 → 无缓存，每次执行验证
```

- **改验证脚本** → 只重跑 Stage 3（~1 分钟，cache hit Stage 2）
- **改驱动 CONFIG** → Stage 2 cache miss → 重新编译（~13 分钟）
- **升级内核版本** → Stage 1 cache miss → 全部重建

### 新增驱动模板

复制一个现有 workflow 文件，只需修改：

1. **on.push.branches** → 新分支名 `build/xxx`
2. **env 驱动相关变量** → 目标 CONFIG 名
3. **Stage 2 中的 `scripts/config`** → 目标驱动的 CONFIG
4. **Stage 3 中的 KOS 列表** → 目标 .ko 路径
5. **Stage 3 中的依赖 .ko 列表** → 目标驱动依赖

其余环境搭建、内核编译、验证逻辑完全复用。

### 已知问题（待完善）

- [ ] Stage 1 和 Stage 2 的 checkout 步骤重复（可通过 artifact 或 reusable workflow 消除）
- [ ] 缓存 ~3GB 编译产物，GitHub 10GB 限制下同时只能存 3 个驱动的缓存
- [ ] 三段式之间通过 cache 传递数据，cache miss 时 Stage 3 会因缺少 `~/linux-source` 而失败
- [ ] 需要一个通用 workflow（参数化驱动配置），而非每个驱动一个 workflow 文件
