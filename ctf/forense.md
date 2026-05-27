# 📻 CTF — USB Radio SSTV

**Categoría:** Forense  
**Flag:** `flag{funni_audio_thingi}`  
**Herramientas:** Python, scipy, PIL, The Sleuth Kit / Autopsy

---

## 🗂️ Descripción del reto

> _"Recibes una pequeña instantánea de una memoria USB que parece estar prácticamente vacía cuando se monta. Los archivos visibles son aburridos e inútiles, pero el tamaño de la imagen sugiere que en algún momento hubo más datos en este dispositivo. Tu tarea consiste en indagar más allá de lo superficial, descubrir lo que quedó atrás y averiguar qué mensaje se pretendía transmitir."_

Archivos entregados:

- `usb-radio.img` — imagen de 64 MB de una USB FAT32

---

## 🔍 Fase 1: Reconocimiento del filesystem

La imagen monta como FAT32 con label `SDR_USB`. Análisis con Autopsy / The Sleuth Kit muestra:

|Archivo|Tamaño|Notas|
|---|---|---|
|`LAB-NOTES.TXT`|155 B|Señuelo|
|`README.TXT`|100 B|Señuelo|
|`RADIO.WAV`|0 B|Header WAV con `data size = 0`|

> [!tip] Pista clave La imagen ocupa 64 MB pero el filesystem apenas usa ~5 KB. Hay ~10.8 MB de datos en **espacio no asignado**.

Los contenidos de los archivos de texto confirman la narrativa del señuelo:

- _"Nothing important is supposed to remain on this device."_
- _"Old snapshots were archived elsewhere before this stick was reused."_

---

## 🕵️ Fase 2: Extracción del espacio no asignado

Análisis manual de la estructura FAT32:

```
Bytes/sector:      512
Sectores/cluster:  1
Sectores reservados: 32
FATs:              2
Sectores/FAT:      1009
Región de datos:   offset 1049600 (sector 2050)
```

Los clusters del filesystem:

- Cluster 2 → directorio raíz
- Cluster 3 → `LAB-NOTES.TXT`
- Cluster 4 → `README.TXT`
- Cluster 5 → header WAV (`RIFF....WAVEfmt` con `data size = 0`)
- Clusters 6+ → marcados como libres en la FAT, pero **contienen datos**

Búsqueda del primer byte no-nulo después del cluster 5:

```python
with open('usb-radio.img', 'rb') as f:
    data = f.read()

for i in range(1051648, len(data)):
    if data[i] != 0:
        print(f"Datos en byte {i}")  # → 1147182
        break
```

**Resultado:** datos reales comienzan en el byte `1147182`, no inmediatamente después del header WAV (que empieza en `1051136`). Hay ~95 KB de silencio entre el header y el audio.

Extracción como WAV válido:

```python
import struct

audio_start = 1147182
audio_size  = 10971854  # hasta último byte no-nulo

header = bytearray()
header += b'RIFF'
header += struct.pack('<I', 36 + audio_size)
header += b'WAVE'
header += b'fmt '
header += struct.pack('<I', 16)
header += struct.pack('<H', 1)      # PCM
header += struct.pack('<H', 1)      # mono
header += struct.pack('<I', 48000)  # 48 kHz
header += struct.pack('<I', 96000)
header += struct.pack('<H', 2)
header += struct.pack('<H', 16)
header += b'data'
header += struct.pack('<I', audio_size)

with open('hidden_audio.wav', 'wb') as f:
    f.write(header + data[audio_start:audio_start + audio_size])
```

**Duración extraída:** ~114.3 segundos a 48 kHz, 16-bit mono.

---

## 📡 Fase 3: Identificación del modo SSTV

Análisis del header VIS mediante frecuencia instantánea:

|Evento|Muestra|Tiempo|Descripción|
|---|---|---|---|
|Calibración|0|0 ms|1900 Hz durante ~1100 ms|
|Break|52799|1100 ms|1200 Hz, 10 ms|
|Start bit VIS|67679|1410 ms|1200 Hz, 30 ms|
|Primer sync de imagen|103509|2156 ms|inicio línea 0|

Espaciado entre syncs de imagen: **~21430 muestras = 446 ms/línea**.

Esto corresponde exactamente a **SSTV Martin M1**:

|Parámetro|Valor|
|---|---|
|Modo|Martin M1 (VIS code 44)|
|Resolución|320×256 píxeles|
|Canales|G → B → R (en ese orden)|
|Sync|1200 Hz, 4.862 ms|
|Gap|1500 Hz, 0.572 ms|
|Scan/canal|1500 Hz = negro, 2300 Hz = blanco, 146.432 ms|
|Duración total|~114 s|

---

## 🖼️ Fase 4: Decodificación SSTV

### Intento inicial — frecuencia instantánea directa

```python
from scipy.signal import hilbert
import numpy as np

analytic = hilbert(samples)
phase    = np.unwrap(np.angle(analytic))
freq     = np.diff(phase) * 48000 / (2 * np.pi)
```

**Problema:** la imagen resultaba con un patrón _checkerboard_ de ruido. La frecuencia instantánea fluctúa mucho entre muestras consecutivas, produciendo pixels alternados de blanco/negro.

### Solución — promediado por pixel

Cada pixel SSTV ocupa `scan_samples / WIDTH = 7029 / 320 ≈ 22 muestras`. Al suavizar la señal de frecuencia con una ventana del tamaño exacto de un pixel, el ruido de alta frecuencia desaparece:

```python
from scipy.ndimage import uniform_filter1d

# Suavizar con ventana del tamaño de 1 pixel SSTV (~22 muestras)
smooth_freq = uniform_filter1d(freq, size=22)
```

### Decoder completo

```python
from scipy.signal import hilbert
from scipy.ndimage import uniform_filter1d
from PIL import Image
import numpy as np

# Parámetros Martin M1
first_sync = 103509
line_gap   = 21430
sync_s     = int(0.004862 * 48000)   # 233 muestras
gap_s      = int(0.000572 * 48000)   # 27 muestras
scan_s     = int(0.146432 * 48000)   # 7029 muestras
WIDTH, HEIGHT = 320, 256
F_MIN, F_MAX  = 1500, 2300

analytic    = hilbert(samples)
phase       = np.unwrap(np.angle(analytic))
freq        = np.clip(np.diff(phase) * 48000 / (2*np.pi), 1000, 3000)
smooth_freq = uniform_filter1d(freq, size=22)

img = np.zeros((HEIGHT, WIDTH, 3), dtype=np.uint8)

for line in range(HEIGHT):
    base    = first_sync + line * line_gap
    g_start = base + sync_s + gap_s
    b_start = g_start + scan_s + gap_s
    r_start = b_start + scan_s + gap_s

    for ch, start in enumerate([g_start, b_start, r_start]):
        chunk = smooth_freq[start:start + scan_s]
        idx   = np.linspace(0, len(chunk)-1, WIDTH, dtype=int)
        img[line, :, ch] = np.clip(
            (chunk[idx] - F_MIN) / (F_MAX - F_MIN) * 255, 0, 255
        ).astype(np.uint8)

# Orden de canales en Martin M1: G, B, R → reordenar a R, G, B
rgb = np.stack([img[:,:,2], img[:,:,0], img[:,:,1]], axis=2)
Image.fromarray(rgb).save('sstv_decoded.png')
```

---

## 🏁 Resultado

La imagen decodificada muestra sobre fondo blanco:

```
flag{funni_audio_thingi}
```

---

## 📚 Lecciones aprendidas

> [!note] Takeaways
> 
> 1. **Espacio no asignado ≠ vacío.** Un filesystem FAT32 puede marcar clusters como libres en la FAT mientras retiene los datos físicos en el disco.
> 2. **El header señuelo.** El archivo `RADIO.WAV` con `data size = 0` es una trampa; los datos reales están en otra región de la imagen.
> 3. **Offset de datos.** El audio no empieza justo después del header WAV. Hay que buscar el primer byte no-nulo después del cluster 5.
> 4. **Checkerboard en SSTV.** La frecuencia instantánea por Hilbert fluctúa a nivel de muestra. Suavizar con una ventana del tamaño de 1 pixel (≈22 muestras para Martin M1 a 48 kHz) elimina el ruido completamente.
> 5. **Orden de canales Martin M1.** El orden de transmisión es **G → B → R**, no RGB.

---

## 🛠️ Herramientas usadas

- `Python 3` + `scipy`, `numpy`, `Pillow`
- `Autopsy 2.24` / `The Sleuth Kit 4.14.0`
- Análisis manual de estructura FAT32

---

_Resuelto: 2026-05-23_


# 📻 CTF — USB Radio SSTV

**Categoría:** Forense  
**Flag:** `flag{funni_audio_thingi}`  
**Dificultad:** Media  
**Fecha:** 2026-05-23

---

## 🧰 Herramientas utilizadas

### Forense de disco

|Herramienta|Uso en el reto|Descarga|
|---|---|---|
|[Autopsy](https://www.autopsy.com/download/)|Explorar filesystem, ver sectores, reporte de strings|Gratis, Windows/Linux|
|[The Sleuth Kit](https://www.sleuthkit.org/)|Backend de Autopsy, comandos `mmls`, `fls`, `icat`|Gratis, CLI|
|[FTK Imager](https://www.exterro.com/ftk-product-downloads/ftk-imager-4-7-1)|Montar imagen y exportar espacio no asignado|Gratis, Windows|
|[Active@ Disk Editor](https://www.disk-editor.com/)|Ver hex de sectores específicos|Gratis, Windows|

### Audio / SSTV

|Herramienta|Uso en el reto|Descarga|
|---|---|---|
|[QSSTV](https://users.telenet.be/on4qz/)|Decodificar SSTV directamente desde audio|Gratis, Linux|
|[RX-SSTV](https://www.qsl.net/on6mu/rxsstv.htm)|Decodificar SSTV en Windows|Gratis, Windows|
|[Audacity](https://www.audacityteam.org/)|Visualizar forma de onda, espectrograma|Gratis, multiplataforma|
|[Sonic Visualiser](https://www.sonicvisualiser.org/)|Análisis espectral detallado del audio SSTV|Gratis, multiplataforma|

### Análisis de imagen

|Herramienta|Uso en el reto|Descarga|
|---|---|---|
|[StegSolve](https://github.com/zardus/ctf-tools/blob/master/stegsolve/install)|Explorar canales de color, bit planes|Gratis, Java|
|[GIMP](https://www.gimp.org/)|Ajustar contraste, curvas, zoom en texto|Gratis, multiplataforma|
|[CyberChef](https://gchq.github.io/CyberChef/)|Análisis rápido online, magic bytes|Online, gratis|

---

## 🗂️ Descripción del reto

> _"Recibes una pequeña instantánea de una memoria USB que parece estar prácticamente vacía cuando se monta. Los archivos visibles son aburridos e inútiles, pero el tamaño de la imagen sugiere que en algún momento hubo más datos en este dispositivo."_

Archivo entregado: `usb-radio.img` — imagen de **64 MB**, filesystem **FAT32**, label `SDR_USB`.

---

## 🔍 Fase 1: Autopsy — Reconocimiento del filesystem

### Cargar la imagen

1. Abrir **Autopsy** → `New Case`
2. `Add Data Source` → `Disk Image or VM File` → seleccionar `usb-radio.img`
3. Dejar todos los módulos de ingest activados

### Lo que muestra Autopsy

En el árbol de archivos (`File Analysis`) aparecen solo 3 archivos:

```
SDR_USB/
├── LAB-NOTES.TXT   (155 bytes)  ← señuelo
├── README.TXT      (100 bytes)  ← señuelo
└── RADIO.WAV       (0 bytes)    ← header trampa, sin datos
```

> [!warning] Red flag inmediato La imagen pesa **64 MB** pero el filesystem ocupa **~5 KB**. Algo hay escondido.

### Revisar el sector 2050 (reporte de Autopsy)

Autopsy → `Image Details` → ir al sector `2050` → `String View`:

```
Sector: 2050
File System Type: fat32
CONTENT:
SDR_USB
LAB-NO~1TXT
README  TXT
ADIO    WAV       ← nombre corto de RADIO.WAV
```

Esto confirma que el directorio raíz está en el sector 2050 y los 3 archivos son todo lo que el FS conoce.

### Ver el espacio no asignado

Autopsy → `Unallocated Space` en el árbol izquierdo → botón derecho → `Extract File(s)`

Exportar como `unallocated.bin` (~53 MB de datos en espacio libre).

> [!tip] Alternativa con FTK Imager `File` → `Add Evidence Item` → `Image File` → seleccionar `.img`  
> En el árbol: `[unallocated space]` → botón derecho → `Export Files`

---

## 🔊 Fase 2: Audacity — Analizar el audio oculto

### Abrir el espacio no asignado

1. Abrir **Audacity**
2. `File` → `Import` → `Raw Data`
3. Seleccionar `unallocated.bin`
4. Configurar: **Signed 16-bit PCM**, **Little-endian**, **1 canal (mono)**, **48000 Hz**
5. Clic en `Import`

> [!note] ¿Por qué 48000 Hz? El header WAV falso (`RADIO.WAV`) tiene el sample rate codificado en el offset +24: `80 BB 00 00` = `0x0000BB80` = **48000**. Autopsy lo muestra en el Hex View del sector 5.

### Lo que se ve en Audacity

La forma de onda muestra:

- ~1.1 segundos de **silencio** al inicio (el header WAV falso en el cluster 5)
- ~2.1 segundos de **tonos puros** (calibración SSTV a 1900 Hz)
- ~112 segundos de **señal FM densa** → esto es la imagen SSTV

![[audacity_waveform.png]] _(La señal SSTV se ve como barras verticales densas y uniformes)_

### Confirmar con espectrograma

1. Clic en el nombre de la pista → `Spectrogram`
2. Ajustar: `Min Freq = 1000 Hz`, `Max Freq = 3000 Hz`
3. Se ven las **líneas horizontales del sync** cada ~446 ms y la variación de frecuencia de los pixeles

---

## 📡 Fase 3: QSSTV — Decodificar la imagen SSTV

### Preparar el archivo WAV

Primero hay que crear un WAV válido con el audio oculto. Opciones:

**Opción A — Audacity:**

1. Seleccionar desde el segundo 2.15 hasta el final
2. `File` → `Export` → `Export as WAV`
3. Formato: 48000 Hz, 16-bit, mono

**Opción B — Python (extracción precisa):**

```python
import struct

with open('usb-radio.img', 'rb') as f:
    data = f.read()

# El audio real empieza en byte 1147182 (primer byte no-nulo después del header)
audio_start = 1147182
audio_end   = 12119035  # último byte no-nulo

audio_size = audio_end - audio_start + 1

with open('hidden_audio.wav', 'wb') as f:
    # Header WAV correcto
    f.write(b'RIFF')
    f.write(struct.pack('<I', 36 + audio_size))
    f.write(b'WAVE')
    f.write(b'fmt ')
    f.write(struct.pack('<I', 16))
    f.write(struct.pack('<H', 1))       # PCM
    f.write(struct.pack('<H', 1))       # mono
    f.write(struct.pack('<I', 48000))   # sample rate
    f.write(struct.pack('<I', 96000))   # byte rate
    f.write(struct.pack('<H', 2))       # block align
    f.write(struct.pack('<H', 16))      # bits per sample
    f.write(b'data')
    f.write(struct.pack('<I', audio_size))
    f.write(data[audio_start:audio_start + audio_size])
```

### Decodificar en QSSTV (Linux)

1. Instalar: `sudo apt install qsstv`
2. Abrir QSSTV
3. `Options` → `Configuration` → `Sound` → cambiar input a **"from file"**
4. `Receive` → botón play → seleccionar `hidden_audio.wav`
5. QSSTV auto-detecta **Martin M1** y decodifica la imagen automáticamente

### Decodificar en RX-SSTV (Windows)

1. Descargar e instalar RX-SSTV
2. Configurar entrada como `Virtual Audio Cable` o directamente desde archivo con VLC
3. Reproducir el WAV con VLC mientras RX-SSTV escucha → imagen aparece en pantalla

> [!tip] Truco con VLC + RX-SSTV `VLC` → `Herramientas` → `Preferencias` → `Audio` → Output: **Windows Audio Session**  
> Luego en RX-SSTV seleccionar el mismo dispositivo como entrada.

---

## 🖼️ Fase 4: GIMP — Leer el flag en la imagen

Si la imagen decodificada tiene ruido (checkerboard), abrir en **GIMP**:

1. `Colors` → `Curves` → bajar las luces, subir las sombras
2. `Colors` → `Brightness-Contrast` → Contrast al máximo
3. `Filters` → `Enhance` → `Unsharp Mask` para enfocar el texto
4. Hacer zoom (`+`) en la esquina inferior derecha donde está el texto

O con la versión Python ya decodificada limpia, la imagen se ve directamente:

![[sstv_decoded_clean.png]]

---

## 🏁 Flag

```
flag{funni_audio_thingi}
```

---

## 🔬 Por qué el decoder Python manual fue necesario

QSSTV y RX-SSTV funcionan bien con audio limpio. En este reto el WAV tenía:

1. **Header con `data size = 0`** → los decoders comerciales rechazan el archivo
2. **Offset de inicio no estándar** → el audio empieza 95 KB después del header
3. **Checkerboard noise** → causado por leer la frecuencia instantánea sin promediar

La solución al checkerboard fue suavizar la señal de frecuencia con una ventana del tamaño exacto de **1 pixel SSTV** (~22 muestras):

```python
from scipy.ndimage import uniform_filter1d

# 1 pixel Martin M1 = 146.432ms / 320px * 48000Hz ≈ 22 muestras
smooth_freq = uniform_filter1d(freq, size=22)
```

---

## 📊 Estructura técnica completa

```
usb-radio.img (64 MB, FAT32)
│
├── [Filesystem — 5 KB]
│   ├── Sector 2050: Directorio raíz
│   ├── Cluster 3:   LAB-NOTES.TXT  (señuelo)
│   ├── Cluster 4:   README.TXT     (señuelo)
│   └── Cluster 5:   RADIO.WAV header (RIFF, data=0)
│
└── [Espacio no asignado — ~53 MB]
    ├── Bytes 1051648–1147181: ceros (padding)
    └── Bytes 1147182–12119035: audio SSTV Martin M1
        ├── 0–2156ms:   Header VIS (calibración 1900Hz + VIS code 44)
        └── 2156–114s:  256 líneas × 320px RGB (G→B→R)
                        → imagen con flag{funni_audio_thingi}
```

---

## 📚 Referencias

- [SSTV Martin M1 — especificación técnica](http://www.barberdsp.com/downloads/Dayton%20Paper.pdf)
- [FAT32 structure — OSDev Wiki](https://wiki.osdev.org/FAT)
- [QSSTV documentation](https://users.telenet.be/on4qz/qsstv/manual/)
- [Autopsy User Guide](https://sleuthkit.org/autopsy/docs/user-docs/4.19.3/)

---

_Resuelto: 2026-05-23 | Categoría: Forense | Flag: `flag{funni_audio_thingi}`_