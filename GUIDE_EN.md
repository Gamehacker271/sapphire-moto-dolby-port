# Android Kitchen Integration Guide: Motorola Dolby Atmos & Spatial Audio (DAX3)
## Target Device: Redmi Note 13 4G / 4G NFC (`sapphire` / `sapphiren` - Qualcomm Snapdragon 685)
### Base: Android 14 / Android 15 (HyperOS 1, HyperOS 2, AOSP, LineageOS, PixelOS, etc.)
### Method: ROM Image Unpacking, Modding & Repacking (ROM Kitchen)

---

## 1. Overview & Methodology (Kitchen / Image Repacking)

This guide is designed exclusively for ROM developers and modders using **Android Kitchen tools (such as CRB Kitchen, SuperR's Kitchen, or manual shell scripts using `lpunpack`, `fsck.erofs`, `mkfs.erofs`, and `lpmake`)**.

By integrating Dolby Atmos directly into the partition images (`system.img`, `vendor.img`, `product.img`), you eliminate the need for Magisk/KernelSU modules, background startup scripts (`service.sh`), and runtime property injections. The entire audio subsystem initializes natively during Android's boot sequence.

---

## 2. Kitchen Workflow Overview

```
[ super.img / Payload.bin ]
           │
           ├── (Unpack via lpunpack / erofs-extract)
           ▼
 Extracted Partitions:
   ├── /system
   ├── /vendor
   └── /system_ext
           │
           ├── 1. Copy binaries, apps, libraries & configs
           ├── 2. Manually edit audio_effects.xml
           ├── 3. Inject SELinux CIL rules & file_contexts
           ├── 4. Configure fs_config (UID/GID permissions)
           └── 5. Add audio props to build.prop
           │
           ├── (Repack via mkfs.erofs / lpmake)
           ▼
[ New Flashed super.img ]
```

---

## 3. File Distribution in Extracted Partition Folders

### A. Vendor Partition (`/vendor`)

Place the following files in the extracted `vendor/` directory, respecting relative paths:

#### 1. HAL Daemon Binary & Init Script
* `vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service`
* `vendor/etc/init/vendor.dolby.hardware.dms@2.0-service.rc`
  * *File contents:*
    ```rc
    service dms-hal-2-0 /vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service
        class hal
        user media
        group audio media
        writepid /dev/cpuset/foreground/tasks

    on post-fs-data
        mkdir /data/vendor/dolby 0770 media media
    ```

#### 2. VINTF Manifest Declaration
* `vendor/etc/vintf/manifest/vendor.dolby.hardware.dms.xml`
  * *File contents:*
    ```xml
    <manifest version="1.0" type="device">
        <hal format="hidl">
            <name>vendor.dolby.hardware.dms</name>
            <transport>hwbinder</transport>
            <version>2.0</version>
            <interface>
                <name>IDms</name>
                <instance>default</instance>
            </interface>
        </hal>
    </manifest>
    ```

#### 3. Tuning & Calibration Profiles
* `vendor/etc/dolby/dax-default.xml` *(DSP tuning and equalizer profiles for SM6225)*
* `vendor/etc/aualgo/aualgo_sst.conf` *(Motorola Smart Sound tuning parameters)*

#### 4. Shared Libraries (`.so`)
* Inside `vendor/lib64/soundfx/` and `vendor/lib/soundfx/`:
  * `libdlbvol.so` (Dolby Volume Listener)
  * `libhwdap.so` (Dolby Hardware DAP Effect)
  * `libswgamedap.so` (Dolby Game DAP Effect)
  * `libswvqe.so` (Dolby Voice Quality Enhancement)
  * `libmsstwrapper.so` (Motorola Smart Sound Wrapper)
* Inside `vendor/lib64/` and `vendor/lib/`:
  * `vendor.dolby.hardware.dms@2.0.so`
  * `vendor.dolby.hardware.dms@2.0-impl.so` *(64-bit only)*
  * `libdlbdsservice.so`
  * `libdlbpreg.so`
  * `libstagefrightdolby.so`
  * `libstagefright_fdtn_dolby.so`
  * `libaualgo_sst_component.so`

---

### B. System Partition (`/system`)

#### 1. System Applications
* `system/priv-app/daxService/daxService.apk`
* `system/priv-app/MotoDolbyDax3/MotoDolbyDax3.apk`
* `system/priv-app/MotorolaSettingsProvider/MotorolaSettingsProvider.apk`
* `system/app/MotoSignatureApp/MotoSignatureApp.apk`

#### 2. Framework JARs
Place in `system/framework/`:
* `moto.jar`
* `moto-core_services.jar`
* `moto-settings.jar`
* `moto-checkin.jar`
* `com.motorola.motosignature.jar`
* `com.motorola.frameworks.core.addon.jar`

#### 3. Privileged Permissions Whitelist & Sysconfig
Place in `system/etc/permissions/`:
* `privapp-com.dolby.daxservice.xml`:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <permissions>
      <privapp-permissions package="com.dolby.daxservice">
          <permission name="android.permission.INTERACT_ACROSS_USERS"/>
          <permission name="android.permission.MANAGE_USERS"/>
          <permission name="android.permission.WRITE_SECURE_SETTINGS"/>
          <permission name="android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME"/>
          <permission name="android.permission.MODIFY_DEFAULT_AUDIO_EFFECTS"/>
          <permission name="com.motorola.gamemode.permission.LAUNCH_GAMEMODE"/>
      </privapp-permissions>
  </permissions>
  ```
* `privapp-com.motorola.dolby.dolbyui.xml`:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <permissions>
      <privapp-permissions package="com.motorola.dolby.dolbyui">
          <permission name="android.permission.BLUETOOTH_PRIVILEGED"/>
          <permission name="android.permission.BROADCAST_CLOSE_SYSTEM_DIALOGS"/>
          <permission name="android.permission.INTERACT_ACROSS_USERS"/>
          <permission name="com.motorola.permission.ACCESS_CHECKIN"/>
      </privapp-permissions>
  </permissions>
  ```
* `privapp-com.motorola.android.providers.settings.xml`
* `moto.xml`, `moto-core_services.xml`, `moto-settings.xml`, `moto-checkin.xml`
* `com.motorola.motosignature.xml`, `com.motorola.frameworks.core.addon.xml`
* `com.motorola.software.dolby.motodolbyui.xml`, `com.motorola.software.sstservice.xml`

Place in `system/etc/sysconfig/`:
* `config-com.dolby.daxservice.xml`
* `config-com.motorola.dolby.dolbyui.xml`
* `config-com.motorola.android.providers.settings.xml`
* `config-com.motorola.motosignature.app.xml`

Place in `system/etc/default-permissions/`:
* `default-com.dolby.daxservice.xml`
* `default-com.motorola.dolby.dolbyui.xml`

---

## 4. Step-by-Step Guide: Editing `audio_effects.xml`

> [!CAUTION]
> **NEVER OVERWRITE** your ROM's stock `audio_effects.xml` with a pre-made file. Every Android build contains target-specific audio effects (such as MiSound, Dirac, Qualcomm LPA, Reverb, etc.). Overwriting the file leads to audio flinger crashes or bootloops. Always open the ROM's stock file and apply the manual modifications detailed below.

The target file is located at `vendor/etc/audio_effects.xml` (or `system/etc/audio_effects.xml` if your ROM centralizes audio effects in system).

### Step 1: Declare Libraries in `<libraries>`
Add the 5 Dolby/Moto libraries inside the `<libraries>` block:

```xml
    <libraries>
        <!-- Preserve stock ROM libraries -->

        <!-- DOLBY ATMOS & MOTO AUDIO -->
        <library name="moto_sst" path="libmsstwrapper.so"/>
        <library name="dvl" path="libdlbvol.so"/>
        <library name="gamedap_mod" path="libswgamedap.so"/>
        <library name="vqe_mod" path="libswvqe.so"/>
        <library name="dap_mod" path="libswdap.so"/>
    </libraries>
```

### Step 2: Declare Effects and UUIDs in `<effects>`
Insert the effect definitions with their exact UUIDs inside the `<effects>` block:

```xml
    <effects>
        <!-- Preserve stock ROM effects -->

        <!-- DOLBY ATMOS & MOTO AUDIO EFFECTS -->
        <effect name="moto_sst" library="moto_sst" uuid="01325f27-2882-44f7-bdec-e7ce4ea3a581"/>
        <effect name="dvl_notification" library="dvl" uuid="1f0091e3-6ad8-40fe-9b09-5948f9a26e7e"/>
        <effect name="dvl_system" library="dvl" uuid="874db4d8-051d-4b7b-bd95-a3bebc837e9e"/>
        <effect name="dvl_alarm" library="dvl" uuid="6aff229c-30c6-4cc8-9957-dbfe5c1bd7f6"/>
        <effect name="dvl_ring" library="dvl" uuid="21d14087-558a-4f21-94a9-5002dce64bce"/>
        <effect name="dvl_music" library="dvl" uuid="40f66c8b-5aa5-4345-8919-53ec431aaa98"/>
        <effect name="gamedap_mod" library="gamedap_mod" uuid="3783c334-d3a0-4d13-874f-0032e5fb80e2"/>
        <effect name="vqe_mod" library="vqe_mod" uuid="64a0f614-7fa4-48b8-b081-d59dc954616f"/>
        <effect name="dap_mod" library="dap_mod" uuid="9d4921da-8225-4f29-aefa-39537a04bcaa"/>
    </effects>
```

### Step 3: Configure `<postprocess>` — THE CRITICAL CONTROL RULE

> [!IMPORTANT]
> **FUNDAMENTAL RULE:**
> **UNDER NO CIRCUMSTANCES** should you add `<apply effect="dap_mod"/>` inside the `<postprocess>` section.

#### Why?
When you add `dap_mod` to `<postprocess>`, `audioserver` automatically instantiates `DAP_offload` (UUID `9d4921da`) on Session 0 at boot. Because `audioserver` connects first, AudioFlinger grants it **`Ctrl=yes`** (exclusive control). When `daxService` later attempts to connect, it receives **`Ctrl=no`** and enters an endless retry loop:
```
waitForDolbyControl: daxservice waiting for control...
DolbyAudioEffects is not accessible!
```
The result is the classic bug: the UI opens and sliders move, but **audio output is completely unaffected**.

#### Correct `<postprocess>` Configuration:
Only apply `moto_sst`, volume listeners (`dvl_*`), and `gamedap_mod` on patch/rerouting streams. Keep `dap_mod` unassigned here so `daxService` acts as the sole client and is granted `Ctrl=yes`:

```xml
    <postprocess>
        <stream type="music">
            <apply effect="moto_sst"/>
            <apply effect="dvl_music"/>
        </stream>
        <stream type="ring">
            <apply effect="moto_sst"/>
            <apply effect="dvl_ring"/>
        </stream>
        <stream type="alarm">
            <apply effect="moto_sst"/>
            <apply effect="dvl_alarm"/>
        </stream>
        <stream type="system">
            <apply effect="moto_sst"/>
            <apply effect="dvl_system"/>
        </stream>
        <stream type="voice_call">
        </stream>
        <stream type="notification">
            <apply effect="moto_sst"/>
            <apply effect="dvl_notification"/>
        </stream>
        <stream type="bluetooth_sco">
        </stream>
        <stream type="dtmf">
        </stream>
        <stream type="enforced_audible">
        </stream>
        <stream type="accessibility">
        </stream>
        <stream type="tts">
        </stream>
        <stream type="assistant">
        </stream>
        <stream type="call_assistant">
        </stream>
        <stream type="patch">
            <apply effect="moto_sst"/>
            <apply effect="gamedap_mod"/>
        </stream>
        <stream type="rerouting">
            <apply effect="moto_sst"/>
            <apply effect="gamedap_mod"/>
        </stream>
    </postprocess>
```

---

## 5. SELinux in Kitchen: CIL Rules, Precompiled Sepolicy & File Contexts

In Android 14/15, SELinux policies are split into plain-text CIL (Common Intermediate Language) files.

Without proper rules, `daxService` (`priv_app`) is blocked from communicating with the HAL `hal_dms_default`, triggering:
`RemoteException: HwBinder Error: (-2147483646)`

### Step 1: Inject Rules into `vendor_sepolicy.cil`
Open `vendor/etc/selinux/vendor_sepolicy.cil` (or `system_ext/etc/selinux/system_ext_sepolicy.cil`) with a text editor and append the following CIL rules:

```cil
; ============================================================
; MOTOROLA DOLBY ATMOS & MOTO CORE SELINUX RULES
; ============================================================

; HwService Manager Find
(allow priv_app hal_dms_hwservice (hwservice_manager (find)))
(allow platform_app hal_dms_hwservice (hwservice_manager (find)))
(allow system_app hal_dms_hwservice (hwservice_manager (find)))
(allow audioserver hal_dms_hwservice (hwservice_manager (find)))
(allow hal_audio_default hal_dms_hwservice (hwservice_manager (find)))

; Bidirectional Binder Call and Transfer (CRITICAL)
(allow priv_app hal_dms_default (binder (call transfer)))
(allow hal_dms_default priv_app (binder (call transfer)))
(allow priv_app hal_dms_default (fd (use)))
(allow hal_dms_default priv_app (fd (use)))

(allow system_app hal_dms_default (binder (call transfer)))
(allow hal_dms_default system_app (binder (call transfer)))
(allow audioserver hal_dms_default (binder (call transfer)))
(allow hal_dms_default audioserver (binder (call transfer)))
(allow hal_audio_default hal_dms_default (binder (call transfer)))
(allow hal_dms_default hal_audio_default (binder (call transfer)))

; File and Property Access
(allow priv_app hal_dms_default_exec (file (read getattr open execute)))
(allow priv_app vendor_displayfeature_prop (file (read open getattr map)))
(allow priv_app vendor_audio_prop (file (read open getattr map)))
(allow priv_app audio_config_prop (file (read open getattr map)))
(allow hal_audio_default vendor_dolby_loglevel_prop (file (read open getattr)))
(allow audioserver vendor_dolby_loglevel_prop (file (read open getattr)))
```

### Step 2: Handling `precompiled_sepolicy` (Preventing Bootloops)
Inside `vendor/etc/selinux/`, Android images include pre-compiled binary files:
* `precompiled_sepolicy`
* `precompiled_sepolicy.plat_sepolicy_and_mapping.sha256`
* `precompiled_sepolicy.vendor_sepolicy_and_mapping.sha256`

> [!WARNING]
> If you modify `vendor_sepolicy.cil` while leaving the original `precompiled_sepolicy` file intact, `init` will detect a SHA256 checksum mismatch and abort booting into Android.

**Two Kitchen Solutions:**

* **Method A (Recommended - Simplest):**
  Delete all `precompiled_sepolicy*` files from `vendor/etc/selinux/`:
  ```bash
  rm -f vendor/etc/selinux/precompiled_sepolicy*
  ```
  On first boot, Android's `init` sees that no pre-compiled binary is present and compiles the `.cil` files (`plat_sepolicy.cil`, `vendor_sepolicy.cil`, etc.) directly into memory in milliseconds.

* **Method B (Advanced):**
  Use `secilc` in your kitchen environment to recompile the combined policy binary and update the `.sha256` files.

### Step 3: Assign Contexts in `vendor_file_contexts`
Open `vendor/etc/selinux/vendor_file_contexts` and append the label for the HAL binary:

```text
/vendor/bin/hw/vendor\.dolby\.hardware\.dms@2\.0-service    u:object_r:hal_dms_default_exec:s0
```

---

## 6. File Permissions and `fs_config`

In tools like **CRB Kitchen** or **SuperR's Kitchen**, UNIX file ownership and modes are configured in `vendor_fs_config` and `system_fs_config`.

### In `vendor_fs_config`:
```text
vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service 0 2000 0755
vendor/etc/init/vendor.dolby.hardware.dms@2.0-service.rc 0 0 0644
vendor/etc/vintf/manifest/vendor.dolby.hardware.dms.xml 0 0 0644
vendor/etc/dolby/dax-default.xml 0 0 0644
vendor/etc/aualgo/aualgo_sst.conf 0 0 0644
vendor/lib/soundfx/lib.*\.so 0 0 0644
vendor/lib64/soundfx/lib.*\.so 0 0 0644
```

### In `system_fs_config`:
```text
system/priv-app/daxService/.* 0 0 0644
system/priv-app/MotoDolbyDax3/.* 0 0 0644
system/priv-app/MotorolaSettingsProvider/.* 0 0 0644
system/app/MotoSignatureApp/.* 0 0 0644
system/framework/moto.*\.jar 0 0 0644
system/framework/com\.motorola.*\.jar 0 0 0644
system/etc/permissions/.*\.xml 0 0 0644
system/etc/sysconfig/.*\.xml 0 0 0644
```

---

## 7. System Properties in `build.prop`

Open `vendor/build.prop` (or `system/build.prop`) in your kitchen and append the following properties:

```properties
# --- MOTOROLA DOLBY ATMOS (DAX3) PROPS ---
persist.vendor.audio_fx.current=dolby
ro.vendor.dolby.dax.version=DAX3_3.8.5.20_r1
ro.vendor.audio.moto_sst_supported=true
ro.audio.ignore_effects=false
ro.dolby.mod_uuid=false
ro.dolby.music_stream=false
ro.spatial.music_stream=false

# --- QUALCOMM SM6225 DSP OFFLOAD ROUTING ---
audio.deep_buffer.media=false
media.stagefright.audio.deep=false
vendor.audio.dolby.ds2.enabled=false
vendor.audio.dolby.ds2.hardbypass=false
audio.offload.disable=1
vendor.audio.offload.track.enable=false
vendor.audio.offload.multiple.enabled=false
```

---

## 8. Repacking & Post-Flash Diagnostics

1. Repack modified partitions into **EROFS** or **EXT4** images using your kitchen tool.
2. Build `super.img` with `lpmake`.
3. Flash the ROM to the device via Fastboot or Custom Recovery.
4. Connect via USB with ADB debugging enabled and execute the diagnostic commands:

### Diagnostic 1: Verify HAL Daemon Process
```bash
adb shell pidof vendor.dolby.hardware.dms@2.0-service
```
*Expected: Returns an active PID owned by user `media`.*

### Diagnostic 2: Verify `daxservice` has `Ctrl=yes`
```bash
adb shell "dumpsys media.audio_flinger | grep -A 10 'UUID: 9d4921da'"
```
*Expected Output:*
```text
3 Clients:
  Pid Priority Ctrl Locked
 4811        0  yes    yes   <-- daxservice MUST HAVE Ctrl=yes
11806        0   no    yes   <-- audioserver MUST HAVE Ctrl=no
```

### Diagnostic 3: Verify Real-Time Equalizer Processing
Open Moto Dolby UI and adjust an equalizer slider:
```bash
adb shell "logcat -d | grep -iE 'fillDax|setGraphic|DolbyAudio'"
```
*Expected: Clean execution of `setGraphicEqualizerBandGains` without any `DolbyAudioEffects is not accessible` errors.*

---

## 9. Credits & Acknowledgements

* **[Rei Ryuki (reiryuki)](https://github.com/reiryuki)** — Special thanks and major credit for creating the original Magisk modules that served as the upstream source for the base binaries and libraries:
  * [Moto-Core-Magisk-Module](https://github.com/reiryuki/Moto-Core-Magisk-Module)
  * [Dolby-Atmos-Spatial-Sound-Moto-Edge-30-Ultra-Magisk-Module](https://github.com/reiryuki/Dolby-Atmos-Spatial-Sound-Moto-Edge-30-Ultra-Magisk-Module)
* **Dolby Laboratories** & **Motorola Mobility LLC** (Proprietary audio algorithms and intellectual property).
* **Gamehac_RA** — Native ROM integration, hardware testing, and porting to Redmi Note 13 4G.
* **Google DeepMind Antigravity** — AudioFlinger pipeline diagnostics, SELinux policy engineering, and solution design.
