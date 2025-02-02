# AxionOS

## Getting Started

To get started with **AxionOS**, you'll need to be familiar with [Source Control Tools](https://source.android.com/setup/develop).

### Initializing the Source

Initialize your local repository using the AxionOS manifest:

```bash
repo init -u https://github.com/AxionAOSP/android.git -b lineage-22.2 --git-lfs
```

Then sync the source:

```bash
repo sync
```

## Build Environment Setup

Make sure your build environment is properly set up by following the [LineageOS build guide](https://wiki.lineageos.org/devices/).

Before building, configure the environment:

```bash
. build/envsetup.sh
```

## Device Flags

Modify your device trees to inherit **LineageOS** common settings and disable **EPPE** (if applicable):

```make
TARGET_DISABLE_EPPE := true
$(call inherit-product, vendor/lineage/config/common_full_phone.mk)
```

### AxionOS-Specific Flags

These flags are needed for About Phone section UI and GMS framework.

#### 📷 Camera Flags

```make
# Define rear camera specs (multiple sensors supported)
AXION_CAMERA_REAR_INFO := 50,48  # Example: 50MP + 48MP

# Define front camera specs
AXION_CAMERA_FRONT_INFO := 42  # Example: 42MP
```

#### 👤 Device Maintainer & Processor Info Flags

```make
# Maintainer name (use "_" for spaces, e.g., "rmp_22" → "rmp 22" in UI)
AXION_MAINTAINER := rmp

# Processor name (use "_" for spaces)
AXION_PROCESSOR := Snapdragon_CPU_1
```

### 🪟 Enabling blur

To enable blur, add the following flag in your device's `lineage_device.mk`:

```make
TARGET_ENABLE_BLUR := true
```

### 🎵 ViperFX Integration

To include **ViPER4AndroidFX**, enable the following flag in your device's `lineage_device.mk`:

```make
TARGET_INCLUDE_VIPERFX := true
```

By default, this flag is **disabled** (`false`). If enabled, make sure your device includes the necessary drivers and libraries.

For full setup instructions, follow the [ViPER4AndroidFX ReadMe](https://github.com/AxionAOSP/android_packages_apps_ViPER4AndroidFX/blob/v4a/README.md).

---

# ⚛️ AxionOS Flags

AxionOS introduces specific CPU affinity settings to optimize system performance. These flags allow builders to define small and big core groups for scheduling critical processes like **SurfaceFlinger**, **HwComposer**, and **RenderEngine** to big cores.

## ⚡ Defining GameSpace Bypass charge support in `lineage_device.mk`

Builders **must** define wether their device supports bypass charging in their device tree:

```make
# if device has the following nodes: 
# /sys/class/power_supply/battery/input_suspend 
# /sys/class/qcom-battery/input_suspend
# bypass charging can be supported
BYPASS_CHARGE_SUPPORTED := true (false by default)
```
Builders **must** also add bypass charging rules on their device tree:

```init.te
# input_suspend_label is the label assigned to /sys/class/power_supply/battery/input_suspend which varies on some device tree
allow init <input_suspend_label>:file rw_file_perms;
```

## 🔧 Defining CPU Core Groups in `lineage_device.mk`

Builders **must** define the CPU core groups in their device tree:

```make
# Define small and big core groups
AXION_CPU_SMALL_CORES := 0,1,2,3
AXION_CPU_BIG_CORES := 4,5,6,7 (builders can exclude prime clusters here)
# Used by cpu limiter and performance mode
# Background cores used for non-critical cpusets 
AXION_CPU_BG := 0-2
# Background cores used for foreground cpusets
AXION_CPU_FG := 0-5
# CPU cores that will be used when limiting other cpusets except top-app
AXION_CPU_LIMIT_BG := 0-1
# Wether to enable debugging for adb logcat purposes
AXION_DEBUGGING_ENABLED := true/false
```

**Do not use `?=` here**, to make sure that it overrides AxionOS defaults

## 🚀 AxionOS Defaults

AxionOS provides default values and assigns them to system properties:

```make
# Default core groups (if not overridden by the builder)
AXION_CPU_SMALL_CORES ?= 0,1,2,3
AXION_CPU_BIG_CORES ?= 4,5,6,7
AXION_CPU_BG ?= 0-2
AXION_CPU_FG ?= 0-5
AXION_CPU_LIMIT_BG ?= 0-1
AXION_DEBUGGING_ENABLED ?= false

# AxionOS scheduling properties
PRODUCT_SYSTEM_PROPERTIES += \
    persist.sys.axion_cpu_big=$(AXION_CPU_BIG_CORES) \
    persist.sys.axion_cpu_small=$(AXION_CPU_SMALL_CORES) \
    persist.sys.axion_cpu_bg=$(AXION_CPU_BG) \
    persist.sys.axion_cpu_limit_bg=$(AXION_CPU_LIMIT_BG) \
    persist.sys.axion_cpu_fg=$(AXION_CPU_FG) \
    ro.sys.axion_userdebug_enabled=$(AXION_DEBUGGING_ENABLED)
```

## 💡 Purpose

These properties are used for:
- **Determining wether to use FIFO or legacy boosting for scheduling**.
- **Affining SurfaceFlinger, HwComposer, and RenderEngine to big cores**.

If your device has a different core configuration, override the values in `lineage_device.mk`.

## 📦 Including LOS Prebuilts
Wether to include LineageOS prebuilt apps (false by default)

```make
TARGET_INCLUDES_LOS_PREBUILTS := true/false
```

---

### ⚙️ Optional: Enabling `SCHED_DEBUG` for Kernel Scheduler Tuning

For non-prebuilt/inline built kernels, you can optionally enable `CONFIG_SCHED_DEBUG` to allow AxionOS to tune scheduler behavior. This is particularly useful for AxionOS load balancing, task migration, and latency optimizations.

To enable it, add/set the following to your kernel's `.config` file:

```config
CONFIG_SCHED_DEBUG=y
```
---

## Resolving AxionOS Kernel Tuning/Performance Mode Denials

Some device trees may encounter kernel tuning denials when accessing certain sysfs nodes. This is often due to differences in OEM labeling that conflict with the labels defined in **device/lineage/sepolicy**. To resolve these issues, please follow one of the two approaches below:

### 1. Standard Labeling Rules

For devices where you can use the standard labels, add the following **genfscon** rules to your device tree:

```genfs_context
genfscon proc /sys/vm/dirty_writeback_centisecs     u:object_r:proc_dirty:s0
genfscon proc /sys/vm/vfs_cache_pressure            u:object_r:proc_drop_caches:s0
genfscon proc /sys/vm/dirty_ratio u:object_r:proc_dirty:s0
genfscon proc /sys/kernel/sched_migration_cost_ns u:object_r:proc_sched:s0
```

These rules ensure that the appropriate security contexts are applied to the sysfs nodes, allowing proper kernel tuning without triggering denials.

### 2.OEM Labeling Adjustments
If your device tree already uses different OEM labels (for example, on MediaTek devices where /sys/vm/dirty_writeback_centisecs is labeled as u:object_r:proc_vm_dirty:s0, while on Qualcomm devices, /sys/vm/dirty-ratio is labeled as u:object_r:proc_dirty_ratio:s0), do not reassign the label in device/sepolicy. Instead, add an allow rule in your device-specific policy to grant the necessary permissions. For instance, for MediaTek/Qualcomm devices, include the following:

```init.te
allow init proc_vm_dirty:file rw_file_perms;
allow init proc_dirty_ratio:file rw_file_perms;
```
This rule permits the init process to access the file with the required read/write permissions, thereby avoiding compilation breakage caused by conflicting label definitions.

Note: These is needed to assure that AxionOS kernel tunings were applied

---

## Building AxionOS

### 🔑 Generate Private Keys

Before building, generate private keys:

```bash
gk -s
```

### 📲 Lunch Command

To configure the build environment for your device, use:

```bash
axion <device_codename>
```

By default, the build system compiles a **vanilla** (non-GMS) build. If you want to include **Google Mobile Services (GMS)**, specify the variant:

```bash
axion <device_codename> <variant>
```

- **gms core** → Includes **Google Mobile Services with Google Telephony** (GApps) (enabled by default if gms variant is unspecified).
- **gms pico** → Includes **Minimal Google Mobile Services** (GApps).
- **va** → Vanilla (GMS-free) build.

**Example:**
To build for a device with codename `panther` and include GMS pico:

```bash
axion panther gms pico
```

To build for a device with codename `panther` with GMS removed:

```bash
axion panther va
```

### 🔄 Syncing Source

After setting up the build environment and syncing the whole source, easily sync the latest source changes with:

```bash
axionSync
```

### ⚙️ Build for device

Compile the ROM with brunch:

```bash
ax -br -j<count>
```

Replace `<count>` with the number of CPU threads for faster compilation (e.g., `ax -j16`).

---

## 📜 Credits

AxionOS is built upon the hard work of the **Android Open Source Project (AOSP)** and **LineageOS** teams. Special thanks to all contributors!

---

🚀 Happy Building!
