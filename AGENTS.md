# AGENTS.md - Arquitectura, Convenciones y Reglas del Proyecto

Este documento define la arquitectura, pila tecnológica, convenciones técnicas y reglas permanentes para cualquier agente que trabaje en el port de Motorola Dolby Atmos (DAX3) + Moto Spatial Sound para Redmi Note 13 4G (`sapphire` / `sapphiren`).

---

## 1. Arquitectura del Proyecto

El proyecto proporciona integración de nivel de sistema (System/Vendor/ROM Kitchen o Módulo KernelSU/Magisk) del motor de audio Motorola Dolby Atmos DAX3 en HyperOS 2 / Android 15 para la plataforma Qualcomm Snapdragon 685 (SM6225).

### Componentes del Sistema

```
[UI / Aplicación]
  ├── com.motorola.dolby.dolbyui (MotoDolbyDax3.apk)
  └── com.motorola.dolby.dolbyui.ui.sst.SSTActivity (Moto Spatial Sound)
         │  (IPC / Intent / ContentProvider)
         ▼
[Middleware Service]
  └── com.dolby.daxservice (daxService.apk, UID 10305, priv-app)
         │  (HwBinder: vendor.dolby.hardware.dms@2.0)
         ▼
[Hardware Abstraction Layer (HAL)]
  └── vendor.dolby.hardware.dms@2.0-service (PID daemon, user: media)
         │  (dlopen / ioctl / parameter storage)
         ▼
[AudioFlinger & Audio Server]
  ├── audioserver (Android Framework / C++)
  ├── libaudiopolicyserviceimpl.so (Parche de lista blanca de efectos en HyperOS)
  └── audio_effects.xml / audio_policy_configuration.xml
         │  (Session 0 Global Effects Chain)
         ▼
[SoundFX DSP Libraries]
  ├── libswdap.so (Dolby Audio Processing Engine - DAP)
  ├── libswgamedap.so (Game DAP Engine)
  ├── libswvqe.so (Voice Quality Enhancement)
  ├── libdlbvol.so (Dolby Volume Listener)
  └── libmsstwrapper.so (Moto SST Spatial Sound Wrapper)
         │
         ▼
[Qualcomm Audio HAL & ALSA Kernel]
  └── android.hardware.audio.service_64 -> /dev/snd/pcmC0D* -> Speaker / Headphones
```

---

## 2. Stack Tecnológico y Entorno

- **Dispositivo de pruebas:** Xiaomi Redmi Note 13 4G (`sapphire`/`sapphiren`).
- **SoC:** Qualcomm SM6225 Snapdragon 685 (8 cores Kryo 265, Adreno 610).
- **Sistema Operativo:** Xiaomi HyperOS 2 Global (Android 15 / API 35).
- **Acceso Root:** KernelSU / Magisk.
- **Identificador ADB:** `cbd15166`.
- **Ruta del repositorio Git:** `/home/gamehacra/.gemini/antigravity/scratch/sapphire_hyperos2_mod/github_repo/`
- **Ruta de staging del módulo:** `/home/gamehacra/.gemini/antigravity/scratch/sapphire_hyperos2_mod/mods/consolidated_module/`
- **Ruta activa en dispositivo:** `/data/adb/modules/sapphire_mod_consolidated/`

---

## 3. Comandos Clave de Operación y Diagnóstico

### Diagnóstico de AudioFlinger y Cadena de Efectos
```bash
# Volcado de hilos y cadenas de efectos de AudioFlinger
adb shell dumpsys media.audio_flinger | grep -E "(Output thread|Effect chain|Effect ID|session)"

# Inspección de clientes de DAP (Verificar Ctrl=yes de daxservice)
adb shell "dumpsys media.audio_flinger | grep -A 10 'UUID: 9d4921da'"

# Estado de procesamiento en tiempo real del HAL de DAP
adb shell "dumpsys media.audio_flinger | grep -A 5 'Effect ID.*HAL dump:'"

# Inspección del hilo primario (AudioOut_D) y tracks activos
adb shell dumpsys media.audio_flinger | sed -n '/Output thread.*AudioOut_D/,/Output thread/p'
```

### Gestión de Servicios de Audio
```bash
# Reinicio en caliente de audioserver y HAL sin reiniciar el dispositivo
adb shell "su -c 'killall -9 audioserver android.hardware.audio.service_64'"

# Comprobar estado del daemon DMS HAL
adb shell pidof vendor.dolby.hardware.dms@2.0-service

# Verificación de parámetros enviados a libswdap
adb logcat -d -t 300 -s DlbDap2Process:V daxservice:V
```

### Verificación del Protector de Bootloop
```bash
# SIEMPRE debe devolver 0
adb shell "cat /data/adb/modules/sapphire_mod_consolidated/.boot_count"
```

---

## 4. Convenciones de Código y Modding

1. **Estructura XML de Efectos (`audio_effects.xml`):**
   - **NUNCA** declarar `dap` o `dap_mod` en `<postprocess>`. Declararlo allí causa que `audioserver` robe el control (`Ctrl=yes`), dejando a `daxService` con `Ctrl=no` en un bucle infinito `waitForDolbyControl`.
   - Únicamente declarar niveladores (`dvl_*`) y `moto_sst` en `<postprocess>`.
   - No usar tipos de stream no soportados en SM6225 (`patch`, `rerouting`, etc.).
2. **Políticas de Audio (`audio_policy_configuration.xml`):**
   - El mixPort `deep_buffer` debe permanecer eliminado o desactivado para que el audio multimedia se enrute por `primary output` (`AUDIO_OUTPUT_FLAG_PRIMARY`), donde reside la cadena de efectos de Sesión 0.
   - Cualquier archivo XML modificado debe validarse sintácticamente con `xml.etree.ElementTree` antes de desplegarse.
3. **SELinux (Formato CIL en Android 14/15):**
   - Las reglas deben inyectarse en `vendor_sepolicy.cil`.
   - Se deben eliminar los binarios `vendor/etc/selinux/precompiled_sepolicy*` si se recompila una ROM completa para evitar discrepancias de hash sha256 y bootloops.
4. **Permisos y Contextos:**
   - Binarios HAL (`vendor/bin/hw/*`): `0755`, contexto `u:object_r:hal_dms_default_exec:s0`.
   - Librerías y configuraciones: `0644`.

---

## 5. Reglas Permanentes e Invariantes

> [!CAUTION]
> **1. PROTECTOR DE BOOTLOOP OBLIGATORIO:**
> El archivo `/data/adb/modules/sapphire_mod_consolidated/.boot_count` implementa un fusible de seguridad en `post-fs-data.sh`. Si el dispositivo reinicia sin completar boot 3 veces consecutivas, el módulo se auto-desactiva.
> **Regla:** Verifica que `.boot_count` vuelva a `0` tras cualquier prueba o reinicio.

> [!IMPORTANT]
> **2. IDIOMA DE COMUNICACIÓN:**
> Toda la comunicación directa con el usuario (`Gamehac_RA`) debe ser en **Español**.

> [!IMPORTANT]
> **3. ENLACES CLICKABLES DE ARCHIVOS:**
> Toda referencia a archivos locales o generados debe incluir enlaces markdown clickables con esquema `file://` (ej. `[archivo.xml](file:///ruta/absoluta)`).

> [!WARNING]
> **4. NO SOBRESCRIBIR DESCRIPTORES NATIVOS DE AUDIO:**
> Nunca reemplazar ciegamente `audio_effects.xml` con archivos pre-fabricados de otros dispositivos. SM6225 posee descriptores de hardware únicos (MiSound, Fluence AEC/NS, etc.). Modificar únicamente los bloques de Dolby/Moto requeridos.
