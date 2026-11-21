# Guía de Integración Vía Android Kitchen: Motorola Dolby Atmos & Spatial Audio (DAX3)
## Dispositivo Objetivo: Redmi Note 13 4G / 4G NFC (`sapphire` / `sapphiren` - Qualcomm Snapdragon 685)
### Base: Android 14 / Android 15 (HyperOS 1, HyperOS 2, AOSP, LineageOS, PixelOS, etc.)
### Método: Descompresión, modificación y reempaquetado de imágenes de la ROM (ROM Kitchen)

---

## 1. Introducción y Metodología (Kitchen / Image Repacking)

Esta guía está diseñada exclusivamente para desarrolladores y editores de ROMs que trabajan con **herramientas de cocina (Android Kitchen, CRB Kitchen, SuperR's Kitchen, o scripts manuales con `lpunpack`, `fsck.erofs`, `mkfs.erofs`, `lpmake`)**.

Al integrar Dolby directamente en las imágenes del sistema (`system.img`, `vendor.img`, `product.img`), se eliminan por completo los módulos de Magisk/KernelSU, los scripts de inicio en segundo plano (`service.sh`), y los parches volátiles. Todo el subsistema de audio se inicializa de manera nativa durante el arranque del kernel y de Android.

---

## 2. Flujo de Trabajo en la Cocina

```
[ super.img / Payload.bin ]
           │
           ├── (Desempaquetar con lpunpack / erofs-extract)
           ▼
 Carpetas extraídas:
   ├── /system
   ├── /vendor
   └── /system_ext
           │
           ├── 1. Copiar binarios, apps, libs y configs
           ├── 2. Editar manualmente audio_effects.xml
           ├── 3. Inyectar reglas CIL y file_contexts de SELinux
           ├── 4. Configurar fs_config (permisos UID/GID)
           └── 5. Añadir props a build.prop
           │
           ├── (Reempaquetar con mkfs.erofs / lpmake)
           ▼
[ Nueva super.img lista para flashear ]
```

---

## 3. Reparto de Archivos en las Carpetas de la ROM

### A. Partición Vendor (`/vendor`)

Copia los siguientes archivos respetando exactamente sus rutas relativas dentro de la carpeta extraída de `vendor/`:

#### 1. Binario del Daemon HAL e Init Script
* `vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service`
* `vendor/etc/init/vendor.dolby.hardware.dms@2.0-service.rc`
  * *Contenido del archivo `.rc`:*
    ```rc
    service dms-hal-2-0 /vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service
        class hal
        user media
        group audio media
        writepid /dev/cpuset/foreground/tasks

    on post-fs-data
        mkdir /data/vendor/dolby 0770 media media
    ```

#### 2. Declaración VINTF
* `vendor/etc/vintf/manifest/vendor.dolby.hardware.dms.xml`
  * *Contenido:*
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

#### 3. Configuraciones y Perfiles
* `vendor/etc/dolby/dax-default.xml` *(Perfiles de ecualización y calibración DSP para SM6225)*
* `vendor/etc/aualgo/aualgo_sst.conf` *(Configuración del algoritmo Motorola Smart Sound)*

> [!IMPORTANT]
> **Calibración DAX de 10 Perfiles vs 9 Perfiles:**
> El archivo `dax-default.xml` incluido en este repositorio proviene de Motorola y define los 10 perfiles requeridos por la app (Dynamic, Movie, Music, Custom, Mobility_default, Mobility_on_the_go, Mobility_commute, Mobility_travel, Game [8] y Voice [9]). Si utilizas el `dax-default.xml` stock de Xiaomi (que sólo define 9 perfiles del 0 al 8), la UI crasheará con `DolbyAudioEffect: Invalid profile index9` y se quedará congelada en "Loading" al intentar abrir los ajustes de cualquier perfil.

#### 4. Bibliotecas Compartidas (`.so`)
* En `vendor/lib64/soundfx/` y `vendor/lib/soundfx/`:
  * `libdlbvol.so` (Dolby Volume Listener)
  * `libhwdap.so` (Dolby Hardware DAP Effect)
  * `libswgamedap.so` (Dolby Game DAP Effect)
  * `libswvqe.so` (Dolby Voice Quality Enhancement)
  * `libmsstwrapper.so` (Motorola Smart Sound Wrapper)
* En `vendor/lib64/` y `vendor/lib/`:
  * `vendor.dolby.hardware.dms@2.0.so`
  * `vendor.dolby.hardware.dms@2.0-impl.so` *(únicamente en lib64)*
  * `libdlbdsservice.so`
  * `libdlbpreg.so`
  * `libstagefrightdolby.so`
  * `libstagefright_fdtn_dolby.so`
  * `libaualgo_foundation.so` *(Dependencia obligatoria de libmsstwrapper en namespace vendor)*
  * `libaualgo_sst_component.so`

> [!NOTE]
> **Aislamiento de Linker VNDK:**
> `libmsstwrapper.so` (Spatial Audio) depende directamente de `libaualgo_foundation.so`. En Android 14/15, debido a la separación de namespaces del linker de Treble, las librerías ubicadas en `vendor` no pueden cargar dependencias desde `/system/lib64/`. Por ello, `libaualgo_foundation.so` DEBE colocarse directamente en `/vendor/lib64/` y `/vendor/lib/`.

---

### B. Partición System (`/system`)

#### 1. Aplicaciones del Sistema
* `system/priv-app/daxService/daxService.apk`
* `system/priv-app/MotoDolbyDax3/MotoDolbyDax3.apk`
* `system/priv-app/MotorolaSettingsProvider/MotorolaSettingsProvider.apk`
* `system/app/MotoSignatureApp/MotoSignatureApp.apk`

#### 2. Framework JARs
Colocar en `system/framework/`:
* `moto.jar`
* `moto-core_services.jar`
* `moto-settings.jar`
* `moto-checkin.jar`
* `com.motorola.motosignature.jar`
* `com.motorola.frameworks.core.addon.jar`

#### 3. Whitelist de Permisos Privilegiados y Sysconfig
Colocar en `system/etc/permissions/`:
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

Colocar en `system/etc/sysconfig/`:
* `config-com.dolby.daxservice.xml`
* `config-com.motorola.dolby.dolbyui.xml`
* `config-com.motorola.android.providers.settings.xml`
* `config-com.motorola.motosignature.app.xml`

Colocar en `system/etc/default-permissions/`:
* `default-com.dolby.daxservice.xml`
* `default-com.motorola.dolby.dolbyui.xml`

---

## 4. Guía de Edición Manual de `audio_effects.xml`

> [!CAUTION]
> **NUNCA REEMPLACES** el archivo `audio_effects.xml` de tu ROM por uno pre-fabricado. Cada compilación contiene descriptores de hardware únicos (MiSound, Dirac, Qualcomm LPA, etc.). Si sobrescribes el archivo, romperás el audio o causarás un bootloop en `audioserver`. Abre el archivo original de la ROM y realiza las modificaciones manuales explicadas aquí.

El archivo a editar se encuentra en `vendor/etc/audio_effects.xml` (o en `system/etc/audio_effects.xml` si tu ROM centraliza los efectos en system).

### Paso 1: Declarar las Bibliotecas en `<libraries>`
Añade las 5 librerías de Dolby/Moto dentro de la etiqueta `<libraries>`:

```xml
    <libraries>
        <!-- Mantén las librerías originales de tu ROM intactas -->

        <!-- DOLBY ATMOS & MOTO AUDIO -->
        <library name="moto_sst" path="libmsstwrapper.so"/>
        <library name="dvl" path="libdlbvol.so"/>
        <library name="gamedap_mod" path="libswgamedap.so"/>
        <library name="vqe_mod" path="libswvqe.so"/>
        <library name="dap_mod" path="libswdap.so"/>
    </libraries>
```

### Paso 2: Declarar los Efectos y sus UUIDs en `<effects>`
Añade las siguientes líneas dentro de la etiqueta `<effects>`:

```xml
    <effects>
        <!-- Mantén los efectos originales de tu ROM intactos -->

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

### Paso 3: Configurar `<postprocess>` — EL ERROR CRÍTICO DEL CONTROL Y TIPOS DE STREAM VÁLIDOS

> [!IMPORTANT]
> **REGLAS FUNDAMENTALES DE POSTPROCESS:**
> 1. **NUNCA** agregues `<apply effect="dap_mod"/>` en `<postprocess>`.
> 2. **SOLO** usa tipos de stream compatibles con Qualcomm SM6225 (`music`, `ring`, `alarm`, `notification`). **NO** agregues tipos como `patch`, `rerouting`, `accessibility` o `call_assistant`, ya que el parser del HAL (`EffectsConfig`) fallará y rechazará cargar el archivo completo.

#### ¿Por qué no poner `dap_mod` en `<postprocess>`?
Cuando agregas `dap_mod` en `<postprocess>`, `audioserver` crea automáticamente una instancia del efecto en la Sesión 0 durante el inicio del sistema. Debido a que `audioserver` conecta primero, `AudioFlinger` le otorga **`Ctrl=yes`** (control exclusivo). Cuando la app `daxService` intenta inicializarse, recibe **`Ctrl=no`** y entra en un bucle infinito:
```
waitForDolbyControl: daxservice waiting for control...
DolbyAudioEffects is not accessible!
```
El resultado es el clásico fallo: la interfaz de Dolby abre, los deslizadores se mueven, pero el sonido no cambia.

#### Edición Correcta y Segura de `<postprocess>`:
Aplica únicamente `moto_sst` (Motorola Spatial Sound) y los niveladores `dvl_*` en los streams compatibles. `dap_mod` debe quedar libre para que `daxService` sea su único cliente y obtenga `Ctrl=yes`:

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
        <stream type="notification">
            <apply effect="moto_sst"/>
            <apply effect="dvl_notification"/>
        </stream>
    </postprocess>
```

---

## 5. SELinux en Kitchen: Reglas CIL, Precompiled Sepolicy y File Contexts

En Android 14/15, las políticas de SELinux ya no se compilan en un archivo monolítico, sino que se dividen en archivos de texto en formato CIL (Common Intermediate Language). 

Si no se agregan estas reglas, `daxService` (`priv_app`) sufrirá denegaciones al comunicarse con el HAL `hal_dms_default` arrojando:
`RemoteException: HwBinder Error: (-2147483646)`

### Paso 1: Inyectar las Reglas en `vendor_sepolicy.cil`
Abre el archivo `vendor/etc/selinux/vendor_sepolicy.cil` (o alternativamente `system_ext/etc/selinux/system_ext_sepolicy.cil`) con un editor de texto y añade al final las siguientes declaraciones CIL:

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

; Bidirectional Binder Call and Transfer (CRÍTICO)
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

### Paso 2: El Manejo de `precompiled_sepolicy` (Evitar Bootloops)
En las ROMs de Xiaomi / Android moderno, dentro de `vendor/etc/selinux/` existen archivos pre-compilados:
* `precompiled_sepolicy`
* `precompiled_sepolicy.plat_sepolicy_and_mapping.sha256`
* `precompiled_sepolicy.vendor_sepolicy_and_mapping.sha256`

> [!WARNING]
> Si editas `vendor_sepolicy.cil` y dejas el archivo binario `precompiled_sepolicy` intacto, Android detectará un desajuste en los hashes sha256 y rechazará cargar la política, provocando un reinicio al recovery.

**Tienes dos métodos limpios en Kitchen:**

* **Método A (Recomendado y más simple):**
  Elimina los archivos `precompiled_sepolicy*` de la carpeta `vendor/etc/selinux/`:
  ```bash
  rm -f vendor/etc/selinux/precompiled_sepolicy*
  ```
  Al arrancar, el binario `init` de Android detecta la ausencia del binario pre-compilado y compila automáticamente los archivos `.cil` (`plat_sepolicy.cil`, `vendor_sepolicy.cil`, etc.) en memoria sin ningún problema.

* **Método B (Para cocinas avanzadas con `secilc`):**
  Si tu cocina incluye `secilc`, recompila el binario `precompiled_sepolicy` con las políticas combinadas y regenera los archivos `.sha256`.

### Paso 3: Asignar Contextos en `vendor_file_contexts`
Abre `vendor/etc/selinux/vendor_file_contexts` y agrega la etiqueta para el ejecutable del HAL:

```text
/vendor/bin/hw/vendor\.dolby\.hardware\.dms@2\.0-service    u:object_r:hal_dms_default_exec:s0
```

---

## 6. Permisos de Archivos y `fs_config`

En herramientas como **CRB Kitchen** o **SuperR's Kitchen**, los permisos UNIX (UID, GID, Mode) se definen en los archivos de configuración de la cocina (`system_fs_config` y `vendor_fs_config`).

### En `vendor_fs_config`:
```text
vendor/bin/hw/vendor.dolby.hardware.dms@2.0-service 0 2000 0755
vendor/etc/init/vendor.dolby.hardware.dms@2.0-service.rc 0 0 0644
vendor/etc/vintf/manifest/vendor.dolby.hardware.dms.xml 0 0 0644
vendor/etc/dolby/dax-default.xml 0 0 0644
vendor/etc/aualgo/aualgo_sst.conf 0 0 0644
vendor/lib/soundfx/lib.*\.so 0 0 0644
vendor/lib64/soundfx/lib.*\.so 0 0 0644
```

### En `system_fs_config`:
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

*(Si empaquetas manualmente con `mkfs.erofs`, asegúrate de pasar `--fs-config-file` con estos parámetros o aplicar `chmod 755` al binario y `chmod 644` al resto).*

---

## 7. Propiedades en `build.prop`

Abre `vendor/build.prop` (o `system/build.prop`) de la cocina y añade las siguientes propiedades para forzar el enrutamiento DSP a Dolby:

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

## 8. Reempaquetado y Diagnóstico tras el Flasheo

1. Reempaqueta las particiones modificadas (`system.img`, `vendor.img`, etc.) en formato **EROFS** o **EXT4** con tu cocina.
2. Reconstruye `super.img` mediante `lpmake` o el generador de tu herramienta.
3. Flashea la ROM por Fastboot o Recovery.
4. Conecta el teléfono por USB con depuración ADB activada y ejecuta las siguientes verificaciones:

### Diagnóstico 1: Comprobar el Daemon HAL
```bash
adb shell pidof vendor.dolby.hardware.dms@2.0-service
```
*Debe devolver un número de proceso (PID) activo bajo el usuario `media`.*

### Diagnóstico 2: Comprobar que `daxservice` tiene `Ctrl=yes`
```bash
adb shell "dumpsys media.audio_flinger | grep -A 10 'UUID: 9d4921da'"
```
*Salida requerida:*
```text
3 Clients:
  Pid Priority Ctrl Locked
 4811        0  yes    yes   <-- daxservice DEBE TENER Ctrl=yes
11806        0   no    yes   <-- audioserver DEBE TENER Ctrl=no
```

### Diagnóstico 3: Comprobar respuesta del ecualizador
Abre la aplicación de Moto Dolby, cambia de modo (por ejemplo, activa Música o mueve cualquier deslizador del ecualizador):
```bash
adb shell "logcat -d | grep -iE 'fillDax|setGraphic|DolbyAudio'"
```
*Deberás ver la llamada `setGraphicEqualizerBandGains` procesada con éxito y sin advertencias de `DolbyAudioEffects is not accessible`.*

---

## 9. Créditos y Agradecimientos

* **[Rei Ryuki (reiryuki)](https://github.com/reiryuki)** — Agradecimiento especial por la creación de los módulos originales para Magisk que sirvieron como fuente de los binarios y librerías base:
  * [Moto-Core-Magisk-Module](https://github.com/reiryuki/Moto-Core-Magisk-Module)
  * [Dolby-Atmos-Spatial-Sound-Moto-Edge-30-Ultra-Magisk-Module](https://github.com/reiryuki/Dolby-Atmos-Spatial-Sound-Moto-Edge-30-Ultra-Magisk-Module)
* **Dolby Laboratories** & **Motorola Mobility LLC** (Algoritmos y propiedad intelectual de procesamiento de audio).
* **Gamehac_RA** — Integración nativa, pruebas en hardware real y port a Redmi Note 13 4G.
* **Google DeepMind Antigravity** — Diagnóstico de AudioFlinger, depuración de SELinux y desarrollo de la solución.
