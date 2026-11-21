# HANDOFF.md - Estado Actual, Anomalía Pendiente y Próximos Pasos

Este archivo resume el estado técnico del proyecto, las modificaciones aplicadas, el problema no resuelto y los pasos exactos a seguir por el siguiente agente.

---

## 1. Objetivo Actual

Hacer que los cambios de perfil de Motorola Dolby Atmos (Dynamic, Movie, Music, Game, Voice, Custom) y los deslizadores del ecualizador gráfico de 10 bandas generen una alteración física y audible en el sonido real del altavoz y auriculares del Redmi Note 13 4G (`sapphire`).

---

## 2. Lo que Ya Está Completado y Verificado

1. **Estabilidad del HAL y Daemon DMS (v2.6):**
   - Se resolvió el fallo crítico anterior (`SIGSEGV fault addr 0x1` en `android.hardware.audio.service_64`).
   - `vendor.dolby.hardware.dms@2.0-service` y `daxService` corren estables sin caídas.
   - `daxService` tiene `Ctrl=yes` exclusivo sobre el UUID `9d4921da-8225-4f29-aefa-39537a04bcaa`.
2. **Audio Limpio sin "Pops" ni Explosiones:**
   - La reproducción de audio PCM fluye de forma continua sin corrupciones ni underruns.
3. **Bypass de `deep_buffer` (v2.7):**
   - Se removió el mixPort `deep_buffer` en `vendor/etc/audio/audio_policy_configuration.xml` y `vendor/etc/audio_policy_configuration.xml`.
   - Android ahora enruta la música directamente a través del hilo primario `AudioOut_D` (`AUDIO_OUTPUT_FLAG_PRIMARY`).
4. **Procesamiento de DAP Activo en AudioFlinger:**
   - En `dumpsys media.audio_flinger`:
     - El hilo `AudioOut_D` tiene la cadena de efectos de Sesión 0 con `Effect ID DAP (libswdap.so)`.
     - `State: 003` (Activo), `Enabled: y`.
     - El contador de bucle `EffectHal::threadLoop n=...` incrementa continuamente en miles de ciclos mientras la música suena.
5. **Persistencia y Git:**
   - Módulo [Sapphire_HyperOS2_Moto_Dolby_Consolidated_v2.7.zip](file:///home/gamehacra/.gemini/antigravity/scratch/sapphire_hyperos2_mod/Sapphire_HyperOS2_Moto_Dolby_Consolidated_v2.7.zip) generado en el host y desplegado en `/data/adb/modules/sapphire_mod_consolidated/`.
   - Repositorio [sapphire-moto-dolby-port](https://github.com/Gamehacker271/sapphire-moto-dolby-port) sincronizado hasta `commit f331536`.
   - Bootloop protector `.boot_count` en `0`.

---

## 3. Archivos Creados o Modificados

### En el Repositorio (`github_repo/`)
- `vendor/etc/audio/audio_policy_configuration.xml` *(Nuevo - sin deep_buffer)*
- `vendor/etc/audio_policy_configuration.xml` *(Nuevo - sin deep_buffer)*
- `vendor/etc/audio_effects.xml` *(Referencia funcional con moto_sst y dlb_* en postprocess)*
- `system_ext/lib64/libaudiopolicyserviceimpl.so` *(Parche HyperOS Sesión 0)*
- `GUIDE_ES.md` y `GUIDE_EN.md` *(Documentación completa de kitchen, postprocess y audio_policy)*
- `README.md` *(Arquitectura de 3 problemas resueltos)*
- `AGENTS.md` *(Reglas e invariantes del sistema)*
- `HANDOFF.md` *(Este archivo)*

### En el Módulo del Dispositivo (`/data/adb/modules/sapphire_mod_consolidated/`)
- `system/vendor/etc/audio_policy_configuration.xml`
- `system/vendor/etc/audio/audio_policy_configuration.xml`
- `system/vendor/etc/audio_effects.xml`
- `system/vendor/etc/dolby/dax-default.xml`
- `system/vendor/lib64/soundfx/libswdap.so` (y librerías vendor Motorola asociadas)
- `module.prop` *(v2.7)*

---

## 4. Anomalía Pendiente ("Sigue igual")

A pesar de que:
- La música pasa por `AudioOut_D`.
- `daxService` recibe las pulsaciones de perfil y envía parámetros `gebs` (Graphic Equalizer Bands Set).
- `libswdap.so` procesa los buffers (`EffectHal::threadLoop n=20000+`).

El usuario reporta que **físicamente el audio no cambia** al cambiar de perfil ni al mover los ecualizadores.

### Hipótesis Técnicas del Problema

1. **Los perfiles para Altavoz (`endpoint_type id="speaker"`) en `dax-default.xml` tienen GEQ plano (0 dB):**
   - En `dax-default.xml`, todos los valores de `band_geq` para el endpoint `speaker` están configurados en `gain="0"` para todos los perfiles por defecto.
2. **Master Enable del Ecualizador Gráfico no se activa:**
   - En el desensamblado de `libswdap.so`, la función `dap_cpdp_graphic_equalizer_bands_set` (tag `gebs`) solo procesa audio si el ecualizador gráfico maestro está habilitado (`dap_cpdp_graphic_equalizer_enable_set`). Si `daxService` solo envía `gebs` sin activar el switch maestro del GEQ para esa sesión/endpoint, las ganancias se almacenan pero no se aplican.
3. **Discrepancia de Endpoint Activo (Speaker vs Headphone):**
   - En `AudioOut_D`, el dispositivo es `0x2 (AUDIO_DEVICE_OUT_SPEAKER)`.
   - En logcat se observó anteriormente `headphone-virtualizer-mode value="1"`, lo que podría indicar que Dolby está aplicando los parámetros al endpoint `headphone` en lugar de `speaker`.
4. **Interferencia de `libmsstwrapper.so` (Moto SST Sound):**
   - `Moto SST Sound` (Efecto ID 91) está ubicado al final de la cadena de efectos (`insert pref: last`) en la Sesión 0. Si `libmsstwrapper.so` sobrescribe el buffer de salida o procesa un buffer no ecualizado, podría estar anulando la salida de `libswdap.so`.
5. **Procesamiento Float vs Int16 en el Pipeline:**
   - `AudioOut_D` procesa internamente en `AUDIO_FORMAT_PCM_FLOAT` (5), pero la salida al HAL de Qualcomm es `AUDIO_FORMAT_PCM_16_BIT` (1). Verificar si el conversor o el mixer están tomando el buffer antes de la etapa de efectos.

---

## 5. Próximos Pasos Exactos

### Paso 1: Prueba de Aislamiento con Auriculares (3.5mm o Bluetooth)
- Pedir o verificar con auriculares conectados si los cambios de perfil y ecualizador son audibles allí (el endpoint `headphone` en `dax-default.xml` tiene virtualización de 8 canales y ecualización no plana activa por defecto).

### Paso 2: Desactivar Temporalmente `Moto SST` de la Sesión 0
- En `vendor/etc/audio_effects.xml`, remover o comentar temporalmente `<apply effect="moto_sst"/>` de `<postprocess>`.
- Reiniciar audioserver (`su -c 'killall -9 audioserver android.hardware.audio.service_64'`).
- Verificar si al dejar a `libswdap.so` como único efecto de la cadena, las modificaciones de sonido se vuelven audibles de inmediato.

### Paso 3: Probar Booster Forzado en `dax-default.xml`
- En `/data/adb/modules/sapphire_mod_consolidated/system/vendor/etc/dolby/dax-default.xml`:
  - Para el perfil `Movie` (`id="1"`), bajo `<endpoint_type id="speaker">`, modificar temporalmente `calibration-boost` de `0` a `120` (o `volmax-boost` a un valor notablemente superior).
  - Reiniciar `dms@2.0-service` (`pkill -f dms`) y audioserver.
  - Al cambiar entre Music y Movie, si el tuning se está leyendo, deberá ocurrir un salto evidente de volumen.

### Paso 4: Inspeccionar la Llamada a `dap_cpdp_graphic_equalizer_enable_set`
- Ejecutar logcat filtrando por `DlbDap2Process`:
  ```bash
  adb logcat -c && adb logcat -s DlbDap2Process:V DlbEffectDap:V
  ```
- Mientras se mueve un deslizador del ecualizador en la UI de Moto Dolby, verificar qué parámetros exactos llegan al HAL (`gebs`, `gebo`, `gebe`, etc.).
