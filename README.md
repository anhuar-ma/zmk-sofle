## Wikis 

Building with pointing to an specific zmk repo
  https://zmk.dev/docs/features/modules#beta-testing

Building zmk locally 

  https://zmk.dev/docs/development/local-toolchain/setup


# Install docker

pacman -S docker

# Follow the docker installation:

https://zmk.dev/docs/development/local-toolchain/setup/container

In your case the zmk-config in the west command is refering to your config file in your repo

make sure that zmk-config contains your conifg files(sofle keymap eg), current working command:

```
west build -p always -s app -d build/left -b nice_nano -S studio-rpc-usb-uart -- -DZMK_CONFIG=/workspaces/zmk-config -DSHIELD=sofle_left -DCONFIG_ZMK_STUDIO=y -DCONFIG_ZMK_STUDIO_LOCKING=n

If


Copilot Make it Work:

Compacted conversationExcellent! Congratulations on the successful build! Let me give you a detailed explanation of what we did and how to set up ZMK locally from scratch. 

Created memory file zmk-hwmv2-migration.md

## What We Fixed - Detailed Breakdown

### **The Core Problem: Zephyr Hardware Model Incompatibility**

Your custom board definition was written for **Zephyr Hardware-Model-v1 (HWMv1)**, but ZMK's main branch uses **Zephyr 4.1**, which requires **Hardware-Model-v2 (HWMv2)**. These are fundamentally different hardware abstraction systems:

| Aspect | HWMv1 (Old) | HWMv2 (Zephyr 4.1) |
|--------|-----------|------------------|
| Board Definition | Single `Kconfig.board` file | `board.yml` metadata file |
| Kconfig Logic | `depends on BOARD_NAME` | `select SOC_NAME` (per-board file) |
| SoC Selection | In `*_defconfig` files | In per-board `Kconfig` files |
| Board Discovery | Manual west board registry | Automatic via board.yml + CMake |

### **Step 1: Created board.yml (HWMv2 Metadata)**

**Location**: board.yml

This file replaces the implicit board discovery of HWMv1. It explicitly tells Zephyr 4.1 that two boards exist with their vendor and SoC information:

```yaml
boards:
  - name: eyelash_sofle_left
    vendor: eyelashperipherals
    socs:
      - name: nrf52840
  - name: eyelash_sofle_right
    vendor: eyelashperipherals
    socs:
      - name: nrf52840
```

**Why this matters**: Without this, Zephyr 4.1's CMake-based board discovery can't find your custom boards.

### **Step 2: Created Per-Board Kconfig Files**

**Files Created**:
- `Kconfig.eyelash_sofle_left`
- `Kconfig.eyelash_sofle_right`

**Before (HWMv1 - Single File)**:
```kconfig
if BOARD_EYELASH_SOFLE_LEFT
  config SOC_NRF52840_QIAA
    default y
endif

if BOARD_EYELASH_SOFLE_RIGHT
  config SOC_NRF52840_QIAA
    default y
endif
```

**After (HWMv2 - Per-Board)**:
```kconfig
# Kconfig.eyelash_sofle_left
config BOARD_EYELASH_SOFLE_LEFT
  select SOC_NRF52840_QIAA
```

```kconfig
# Kconfig.eyelash_sofle_right
config BOARD_EYELASH_SOFLE_RIGHT
  select SOC_NRF52840_QIAA
```

**Key difference**: HWMv2 uses `select` statements that proactively choose the SoC, rather than conditional `if/endif` blocks. This is how Zephyr 4.1's Kconfig system expects board definitions.

### **Step 3: Removed Legacy Kconfig.board**

The old `Kconfig.board` file mixed both board definitions with `depends on` logic. In HWMv2, this is handled automatically by CMake reading `board.yml` and loading the appropriate per-board Kconfig file. The old file would conflict.

### **Step 4: Cleaned Defconfig Files**

**Removed from both `eyelash_sofle_left_defconfig` and `eyelash_sofle_right_defconfig`**:
```
CONFIG_SOC_SERIES_NRF52X=y
CONFIG_SOC_NRF52840_QIAA=y
CONFIG_BOARD_EYELASH_SOFLE_LEFT=y    (or _RIGHT for right)
```

**Why**: In HWMv2, these are now selected via the Kconfig files, not hardcoded in defconfig. Having them in both places causes conflicts and redundancy.

**What stayed**: All the actual hardware driver configs, ZMK features, peripheral settings, RGB underglow, encoders, etc.

### **Step 5: Fixed Invalid Kconfig Symbol**

**Problem**: eyelash_sofle.conf contained `CONFIG_WS2812_STRIP=y`, but this symbol doesn't exist in either ZMK or Zephyr 4.1's Kconfig trees.

**Solution**: Removed that line. The actual RGB underglow is controlled by `CONFIG_LED_STRIP=y`, which is valid.

---

## Local ZMK Setup - Complete Guide

Here's how to set up ZMK locally on your machine (Mac/Linux):

### **Prerequisites**
- Git
- Docker (recommended for consistency with CI builds) OR native Zephyr SDK
- USB cable for programming boards

### **Option 1: Docker Setup (Recommended - Identical to CI)**

#### 1.1 Pull ZMK Docker Image
```bash
docker pull zmkfirmware/zmk-dev-arm:latest
```

#### 1.2 Clone ZMK Main Branch
```bash
git clone https://github.com/zmkfirmware/zmk.git
cd zmk
git switch main  # Use latest ZMK (Zephyr 4.1)
```

#### 1.3 Initialize ZMK Workspace
```bash
west init -l app/
west update
```

#### 1.4 Run Docker Container with Volumes
```bash
docker run --rm -it \
  -v $(pwd):/workspaces/zmk \
  -v /path/to/zmk-config:/workspaces/zmk-config \
  -v /path/to/zmk-modules:/workspaces/zmk-modules \
  zmkfirmware/zmk-dev-arm:latest bash
```

**Inside the container**, your directories are mounted:
- `/workspaces/zmk` → Your ZMK repo
- `/workspaces/zmk-config` → Your config folder (keymaps, layouts)
- `/workspaces/zmk-modules` → Your custom board definitions

#### 1.5 Build Left Half (Inside Container)
```bash
cd /workspaces/zmk
west build -p always -s app -d build/left \
  -b eyelash_sofle_left \
  -S studio-rpc-usb-uart \
  -- -DZMK_CONFIG=/workspaces/zmk-config \
  -DZMK_EXTRA_MODULES=/workspaces/zmk-modules \
  -DSHIELD=nice_view \
  -DCONFIG_ZMK_STUDIO=y \
  -DCONFIG_ZMK_STUDIO_LOCKING=n
```

**Output**: `build/left/zephyr/zmk.uf2` (ready to flash)

#### 1.6 Build Right Half (Inside Container)
```bash
west build -p always -s app -d build/right \
  -b eyelash_sofle_right \
  -- -DZMK_CONFIG=/workspaces/zmk-config \
  -DZMK_EXTRA_MODULES=/workspaces/zmk-modules \
  -DSHIELD=nice_view
```

**Output**: `build/right/zephyr/zmk.uf2`

---

### **Option 2: Native Setup (Mac/Linux)**

#### 2.1 Install Zephyr SDK
```bash
# Create a dedicated directory
mkdir ~/zephyr-workspace
cd ~/zephyr-workspace

# Download and install Zephyr SDK 0.16.9 (compatible with Zephyr 4.1)
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.9/zephyr-sdk-0.16.9_linux-x86_64.tar.xz
tar xvf zephyr-sdk-0.16.9_linux-x86_64.tar.xz
./zephyr-sdk-0.16.9/setup.sh
```

#### 2.2 Install Python Dependencies
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt  # From ZMK repo
```

#### 2.3 Clone and Initialize ZMK
```bash
git clone https://github.com/zmkfirmware/zmk.git
cd zmk
git switch main
west init -l app/
west update
```

#### 2.4 Link Your Config
```bash
# Create symlink to your config in ZMK's config folder
ln -s /path/to/zmk-config config
# OR set the CMake variable in builds
```

#### 2.5 Build (Same Commands as Docker, No Container Prefix)
```bash
west build -p always -s app -d build/left \
  -b eyelash_sofle_left \
  -S studio-rpc-usb-uart \
  -- -DZMK_CONFIG=/path/to/zmk-config \
  -DZMK_EXTRA_MODULES=/path/to/zmk-modules \
  -DSHIELD=nice_view \
  -DCONFIG_ZMK_STUDIO=y \
  -DCONFIG_ZMK_STUDIO_LOCKING=n
```

---

## Your Custom Board Files Structure

For your setup to work, your `zmk-modules` repo needs this structure:

```
zmk-modules/
├── zephyr/
│   └── module.yml                          # References board_root
└── boards/
    └── arm/
        └── eyelash_sofle/
            ├── board.yml                   # NEW - HWMv2 metadata
            ├── board.cmake                 # Unchanged
            ├── Kconfig.board               # DELETED
            ├── Kconfig.eyelash_sofle_left  # NEW - Per-board
            ├── Kconfig.eyelash_sofle_right # NEW - Per-board
            ├── Kconfig.defconfig           # Unchanged (if exists)
            ├── eyelash_sofle.dtsi          # Hardware definitions
            ├── eyelash_sofle_left.dts      # Left specific
            ├── eyelash_sofle_right.dts     # Right specific
            ├── eyelash_sofle_left_defconfig  # Left defaults (cleaned)
            ├── eyelash_sofle_right_defconfig # Right defaults (cleaned)
            └── eyelash_sofle.keymap        # Keymap overlay
```

---

## Key Build Command Flags Explained

| Flag | Purpose |
|------|---------|
| `-p always` | Clean build (always) |
| `-s app` | Source directory (ZMK app) |
| `-d build/left` | Output directory |
| `-b eyelash_sofle_left` | Board target |
| `-S studio-rpc-usb-uart` | Snippet (enable ZMK Studio over USB/UART) |
| `-DZMK_CONFIG=...` | Path to your config folder (keymaps, layouts) |
| `-DZMK_EXTRA_MODULES=...` | Path to your custom boards |
| `-DSHIELD=nice_view` | Display shield selection |
| `-DCONFIG_ZMK_STUDIO=y` | Enable ZMK Studio |
| `-DCONFIG_ZMK_STUDIO_LOCKING=n` | Disable Studio locking (allow real-time edits) |

---

## Flashing Firmware to Hardware

Once you have `.uf2` files:

1. **Enter Bootloader** (Double-tap RESET on nice!nano v2)
2. **Mount as USB Drive** (appears as `NICENANO` or `NRF52840` drive)
3. **Copy UF2 File**:
   ```bash
   cp build/left/zephyr/zmk.uf2 /Volumes/NICENANO/  # Mac
   cp build/left/zephyr/zmk.uf2 /media/YOUR_USER/NICENANO/  # Linux
   ```
4. **Unmount** (board auto-resets and firmware loads)

---

## Troubleshooting Common Issues

| Issue | Solution |
|-------|----------|
| Board not found | Create `board.yml` with HWMv2 format |
| `cannot assign value to undefined symbol` | Check all config symbols exist in Zephyr/ZMK |
| DTC warnings | Non-fatal, related to deprecated syntax |
| Build cache issues | Use `-p always` to force clean build |
| Docker volume permission denied | Check mount paths exist and use full paths |

---

The migration from HWMv1 to HWMv2 was the critical fix. Zephyr 4.1 is stricter about board definitions, but once properly formatted, the build process is rock solid!


```

```
west build -p always -s app -d build/right -b eyelash_sofle_right \
  -- -DZMK_CONFIG=/workspaces/zmk-config \
  -DZMK_EXTRA_MODULES=/workspaces/zmk-modules \
  -DSHIELD=nice_view


Both Commands


 west build -p always -s app -d build/left -b eyelash_sofle_left -S studio-rpc-usb-uart -- -DZMK_CONFIG=/workspaces/zmk-config -DZMK_EXTRA_MODULES=/workspaces/zmk-modules -DSHIELD=nice_view -DCONFIG_ZMK_STUDIO=y -DCONFIG_ZMK_STUDIO_LOCKING=n  && 


 west build -p always -s app -d build/right -b eyelash_sofle_right   -- -DZMK_CONFIG=/workspaces/zmk-config   -DZMK_EXTRA_MODULES=/workspaces/zmk-modules   -DSHIELD=nice_view


