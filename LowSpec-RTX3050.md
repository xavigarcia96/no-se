<!--
SPDX-License-Identifier: GPL-3.0-or-later
-->

# Lichii en laptop RTX 3050 (4 GB) / 8 GB RAM / Core i5 de 12a gen

Guía de ajuste para este hardware. Los números de rendimiento no están medidos en esta máquina:
úsala como punto de partida y compara con el log de tu propia sesión.

## Qué esperar

| Tipo de juego | Viabilidad en este equipo | Por qué |
|---|---|---|
| Switch 1 (layout de 4 GB) | Buena en la mayoría de títulos a 720p | La memoria del invitado son 4 GiB y la VRAM alcanza con ASTC→BC3 |
| Switch 2 | Muy limitada, en la práctica no jugable con fluidez | El emulador usa el layout de 12 GiB (+ memoria del servidor gráfico: ~20 GiB de *commit* de respaldo) y los juegos llenan 6-7 GB de VRAM. Con 8 GB de RAM y 4 GB de VRAM habrá paginación constante |

Referencia del propio proyecto (`NX2_NOTES.md`, fila 128): un Ryzen 5 2600 + RTX **3050 de 6 GB**
daba 8 fps con los valores por defecto de Eden y ~20 fps irregulares con los ajustes de abajo en un
título de Switch 2. Esta laptop tiene menos VRAM y menos RAM que ese equipo.

## 1. Perfil de configuración

Copia `dist/lowspec/qt-config-rtx3050-4gb.ini` (instrucciones dentro del archivo) a
`%APPDATA%\eden\config\qt-config.ini`. Lo que cambia respecto a un arranque limpio:

- **Escalado `ScaleForce`** en lugar de `Fsr3` (el valor de primer arranque), que cuesta GPU y VRAM.
- **VSync `FifoRelaxed`**: si bajas de 60 fps no se cae a 30.
- **Reactive flushing desactivado**, **precisión de GPU baja** y **shaders asíncronos**: lo que el
  proyecto midió como mejor en una RTX 3050.
- **ASTC en GPU recomprimido a BC3** y **VRAM conservadora**: la RTX no soporta ASTC nativo; sin
  recompresión las texturas ocupan 4-16 veces más (`NX2_NOTES.md`, fila 99). BC3 ya usa 4 bits por
  píxel en las texturas opacas, así que BC1 solo aporta pérdida de alfa; no lo uses.
- **Modo portátil (720p)**. En dock (1080p) se renderizan ~2,25 veces más píxeles.

Si algún juego va justo de VRAM, baja primero el anti-aliasing a `None` (`anti_aliasing=0`) y
mantén `resolution_setup=3` (1X); no subas la resolución.

## 2. Memoria virtual de Windows (lo más importante con 8 GB de RAM)

El respaldo de memoria del invitado se reserva como *commit* al arrancar. Con 8 GB de RAM y el
archivo de paginación en automático puede fallar con `Failed to allocate ... MiB of backing memory`
(`NX2_NOTES.md`, fila 128).

1. `Configuración > Sistema > Acerca de > Configuración avanzada del sistema > Rendimiento >
   Configuración > Opciones avanzadas > Memoria virtual > Cambiar`.
2. Desmarca "Administrar automáticamente...", elige tu SSD, **Tamaño personalizado**:
   inicial y máximo **24576 MB** (24 GB). Acepta y reinicia.
3. Mantén ≥ 30 GB libres en esa unidad.

Es necesario para los juegos de Switch 2 y una red de seguridad para los de Switch 1.

## 3. GPU y energía (laptop híbrida Intel + NVIDIA)

- Con el cargador conectado y el modo de energía de Windows en "Mejor rendimiento".
- `Configuración > Sistema > Pantalla > Gráficos`: añade `lichii.exe` y pon **Alto rendimiento**
  (RTX 3050), para que Optimus no use la gráfica integrada.
- Panel de control de NVIDIA > Administrar configuración 3D > Programa `lichii.exe`:
  Modo de administración de energía = **Preferir máximo rendimiento**; Tamaño de caché de
  sombreador = **Ilimitado** (o ≥ 10 GB).
- Cierra navegadores y Discord antes de jugar: consumen RAM y VRAM que aquí no sobran
  (el emulador ya ajusta sus umbrales de texturas al presupuesto de VRAM que reporta el driver, y
  lo reduce si otros programas usan VRAM).
- Driver NVIDIA reciente. Si hay cortes o *stutter* tras actualizar, borra la caché de shaders de
  Lichii (`%APPDATA%\eden\cache\shader`) y la de NVIDIA (`%LOCALAPPDATA%\NVIDIA\DXCache`) y vuelve a jugar.

## 4. Compilar para el i5 de 12a gen (opcional)

Primero prueba el build publicado `windows-x86_64-pgo`: usa el perfil PGO del proyecto, que se
afinó con juegos de Switch 2 (con un perfil de Switch 1 el mismo juego iba a la mitad de velocidad,
ver `.github/workflows/release-binaries.yml`).

Para compilar tú mismo, este parche añade el preset `alderlake` (AVX2/FMA/BMI2, sin AVX-512, que es
lo que tienen los Core de 12a gen):

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DYUZU_BUILD_PRESET=alderlake ...
```

`-DYUZU_BUILD_PRESET=v3` es la alternativa portable. No he medido la combinación del preset con el
perfil PGO; compara fps contra el build oficial antes de quedarte con uno.

## 5. Cómo leer tu log

En `%APPDATA%\eden\log\eden_log.txt` busca:

- `Available VRAM:` debe indicar entre ~3 y 3,5 GiB (4 GB menos la reserva del emulador y lo que Windows use).
- `Texture cache thresholds:` y, en juegos de Switch 2, `Out of device memory: keeping ... MiB more
  free`: si esta última aparece a menudo, la VRAM está al límite; baja anti-aliasing y no uses dock.
- `GPU thread pinning skipped: 4 physical performance cores, needs 6`: es lo esperado en un i5 H de
  4 núcleos P + E-cores. El emulador ya envía los hilos críticos a los núcleos de rendimiento y los
  trabajos de fondo a los de eficiencia; el fijado dedicado solo se activa con ≥ 6 núcleos P.

## Qué cambia este parche en el código

- `CMakeLists.txt`, `docs/Options.md`: preset `YUZU_BUILD_PRESET=alderlake`.
- `src/video_core/buffer_cache/buffer_cache.h`: en títulos de Switch 2 el recolector de buffers
  empezaba a liberar a 512 MiB (agresivo a 1 GiB) sin importar la VRAM. En una tarjeta con menos de
  4 GiB utilizables eso es una fracción demasiado grande que la caché de texturas también necesita,
  así que ahora escala a 1/8 y 1/4 de la VRAM disponible (448 / 896 MiB con ~3,5 GiB). Sin cambios
  en tarjetas de 4 GiB o más.
