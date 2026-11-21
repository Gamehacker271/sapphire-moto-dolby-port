# Motorola Dolby Atmos Spatial Audio Port for Redmi Note 13 4G (`sapphire` / `sapphiren`)
### Android Kitchen & ROM Repacking Guide

[![Platform](https://img.shields.io/badge/Platform-Qualcomm%20Snapdragon%20685%20(SM6225)-blue.svg)](#)
[![Android](https://img.shields.io/badge/Android-14%20%2F%2015-green.svg)](#)
[![Method](https://img.shields.io/badge/Method-Android%20Kitchen%20%2F%20Image%20Repack-purple.svg)](#)
[![Status](https://img.shields.io/badge/Status-Fully%20Functional-brightgreen.svg)](#)

A complete, production-grade guide, pre-arranged files, and technical breakdown for integrating **Motorola Dolby Atmos (DAX3) & Spatial Sound** directly into custom ROMs via **Android Kitchen / Image Repacking tools** (`system.img`, `vendor.img`, `super.img`) for the **Xiaomi Redmi Note 13 4G / 4G NFC** (`sapphire` / `sapphiren`).

> 🚀 **100% Native Integration — Zero Modules, Zero Scripts.**  
> Built for ROM builders using tools like **CRB Kitchen, SuperR's Kitchen, or manual `lpunpack`/`mkfs.erofs` scripts**. No `service.sh`, no Magisk/KernelSU modules, and no runtime hacks.

---

## 📁 Repository Structure

All necessary proprietary binaries, apps, libraries, configs, and VINTF fragments are included in this repository and organized in their target paths:

```
├── README.md
├── GUIDE_ES.md                          # Guía de integración en Español
├── GUIDE_EN.md                          # Integration guide in English
├── system/                              # Files to copy into extracted /system
│   ├── app/MotoSignatureApp/
│   ├── priv-app/
│   │   ├── MotoDolbyDax3/
│   │   ├── MotorolaSettingsProvider/
│   │   └── daxService/
│   ├── framework/                       # Moto Core & Core Addon framework JARs
│   └── etc/                             # Permissions, sysconfig & default-permissions
├── system_ext/                          # Optional /system_ext overrides (HyperOS whitelist bypass)
│   └── lib64/                           # Patched libaudiopolicyserviceimpl.so
└── vendor/                              # Files to copy into extracted /vendor
    ├── bin/hw/                          # vendor.dolby.hardware.dms@2.0-service
    ├── etc/
    │   ├── aualgo/                      # aualgo_sst.conf
    │   ├── dolby/                       # dax-default.xml
    │   ├── init/                        # vendor.dolby.hardware.dms@2.0-service.rc
    │   ├── vintf/manifest/              # vendor.dolby.hardware.dms.xml
    │   └── audio_effects.xml            # Working reference audio effects configuration
    ├── lib/                             # 32-bit codecs & soundfx libraries
    └── lib64/                           # 64-bit codecs & soundfx libraries
```

---

## 📖 Integration Guides / Guías de Integración

* 🇪🇸 **[Guía Completa para Android Kitchen en Español (GUIDE_ES.md)](GUIDE_ES.md)**
* 🇬🇧 **[Complete Android Kitchen Guide in English (GUIDE_EN.md)](GUIDE_EN.md)**

---

## 🔍 The Breakthrough: Solving the Broken Equalizer Bug

Many Dolby Atmos ports integrated into ROMs suffer from a notorious issue: **the application opens, presets toggle, and sliders move, but the audio never changes**.

Through in-depth `AudioFlinger` diagnostics and SELinux trace analysis on Snapdragon 685, we resolved the two root causes:

### 1. The `<postprocess>` Audio Effects Trap (AudioFlinger Control Stealing)
* **The Cause:** Declaring `<apply effect="dap_mod"/>` in `<postprocess>` inside `audio_effects.xml` forces Android's early `audioserver` to bind `DAP_offload` (UUID `9d4921da`) first on Session 0, taking **`Ctrl=yes`**. When `daxservice` launches later, AudioFlinger assigns it **`Ctrl=no`**, entering an infinite `waitForDolbyControl` loop (`DolbyAudioEffects is not accessible`).
* **The Fix:** **Never** declare `dap_mod` in `<postprocess>`. `daxservice` becomes the sole client, granting it exclusive `Ctrl=yes` over real-time hardware DSP equalizer gains.

```
BEFORE (Broken - Audioserver steals control):
  Effect ID 27 (DAP_offload):
    Pid  Priority  Ctrl  Locked
   2445         0   yes     yes  <-- audioserver holds Ctrl
   4811         0    no     yes  <-- daxservice has NO control (Broken EQ)

AFTER (Fixed - DaxService has exclusive control):
  Effect ID 27 (DAP_offload):
    Pid  Priority  Ctrl  Locked
   4811         0   yes     yes  <-- daxservice has EXCLUSIVE control (Working EQ!)
  11806         0    no     yes  <-- audioserver does not interfere
```

### 2. SELinux CIL Injection in Modern Android Images
* **The Cause:** `daxservice` runs as `priv_app`, while `vendor.dolby.hardware.dms@2.0-service` runs as `hal_dms_default`. On Android 14/15, cross-domain HwBinder object transfers are denied by default (`avc: denied { transfer }`), causing `RemoteException: HwBinder Error: (-2147483646)`.
* **The Fix:** The guide details how to inject bidirectional Binder rules directly into `vendor_sepolicy.cil` and handle `precompiled_sepolicy` to prevent bootloops.

---

## 📱 Device Compatibility

| Device Codename | Commercial Name | SoC | Tested Bases |
| :--- | :--- | :--- | :--- |
| `sapphire` | Xiaomi Redmi Note 13 4G | Qualcomm Snapdragon 685 (SM6225) | HyperOS 1, HyperOS 2 (Android 14/15) |
| `sapphiren` | Xiaomi Redmi Note 13 4G NFC | Qualcomm Snapdragon 685 (SM6225) | HyperOS 1, HyperOS 2 (Android 14/15) |

---

## 🛠 Features

- **Full Motorola Dolby Atmos (DAX3)** UI with real-time spectrum and profiles.
- **Working 10-Band Graphic Equalizer** and audio enhancers.
- **Motorola Spatial Sound** (Virtualizer for headphones and speakers).
- **Smart Audio (Moto SST)** algorithm.
- **Native Init Service**: Managed by Android `init` under UID `media`, GID `audio media`.

---

## 🤝 Credits & Acknowledgements

* **[Rei Ryuki (reiryuki)](https://github.com/reiryuki)** — Special thanks and major credit for creating and maintaining the upstream Magisk modules that served as the base for these extracted binaries and configurations:
  * **[Moto Core Magisk Module](https://github.com/reiryuki/Moto-Core-Magisk-Module)**
  * **[Dolby Atmos Spatial Sound Moto Edge 30 Ultra Magisk Module](https://github.com/reiryuki/Dolby-Atmos-Spatial-Sound-Moto-Edge-30-Ultra-Magisk-Module)**
* **Dolby Laboratories** & **Motorola Mobility LLC** (Proprietary audio processing algorithms and binaries).
* **Gamehac_RA** — ROM integration, validation, device testing, and porting to Redmi Note 13 4G.
* **Google DeepMind Antigravity** — Deep diagnostics, SELinux policy engineering, and AudioFlinger pipeline debugging.

---

## ⚖️ License & Disclaimer

This repository is for educational and research purposes only. All Dolby Atmos and Motorola intellectual properties, trademarks, and binaries belong to their respective owners.
