## 📋 Documentación Paso a Paso del Análisis Forense

### Paso 1: Generación del Body File (`fls`)

Como la partición montada no mostraba el flag mediante búsquedas tradicionales (`grep`), decidiste extraer de forma cruda toda la estructura de metadatos del archivo de imagen utilizando `fls`:


```python
fls -m / -r partition4.img > bod.txt
```

#### Mostrar particiones borradas
```python
fls -r -d disko-4.dd  
   
r/r * 522629:	log/messages
r/r * 532021:	log/dont-delete.gz

# Extraer la particiones borradas
 icat disko-4.dd 532021 > dont-delete.gz

```

- **¿Qué pasó aquí?:** El flag `-m /` generó una salida en formato "body file" (un estándar forense que recopila marcas de tiempo de forma cruda), y `-r` le indicó que lo hiciera de manera recursiva por todos los directorios. Toda esta metadata se guardó en `bod.txt`.
    

### Paso 2: Creación de la Línea de Tiempo (`mactime`)

Teniendo la metadata cruda en el archivo de texto, el siguiente paso lógico fue estructurarla cronológicamente para examinar qué acciones o comandos sospechosos se ejecutaron en el sistema antes de que se apagara:

Bash

```python
mactime -b bod.txt -d 2>/dev/null > timeline.csv

#otras maneras vectores de verificacion
#`head`: "De todo ese montón de texto que te acaba de pasar `cat`, **agárrame solo las primeras 10 líneas** y muéstramelas."

cat timeline.csv | head

```

- **¿Qué pasó aquí?:** El flag `-b` tomó el archivo origen y `-d` le dio una estructura delimitada por comas (`.csv`) para poder ser abierta fácilmente en hojas de cálculo como LibreOffice Calc. Limpiaste la salida redirigiendo las advertencias de Perl a `/dev/null`.
    

### Paso 3: Identificación e Inspección de Inodos Sospechosos (`icat`)

Al analizar los eventos y archivos indexados en la línea de tiempo o las cadenas del sistema, localizaste varios inodos clave correlacionados con el apagado del sistema y la persistencia de configuraciones (`4943`, `33020`, `33017`), los cuales fuiste auditando uno por uno de forma directa en los bloques del disco:

Bash

```python
icat partition4.img 4943    # Salida: poweroff
icat partition4.img 33020   # Salida: shutdown%
icat partition4.img 33017   # Salida: Archivos de configuración antiguos (depconfig, deptree, etc.)
```

### Paso 4: Extracción de la Evidencia Escondida (Inodo 32716)

Finalmente, diste con el inodo crítico: el **`32716`**, el cual resguardaba un archivo que contenía una cadena de texto sospechosa que sobrevivió en los bloques lógicos de la partición:

Bash

```python
icat partition4.img 32716
```

- **Resultado obtenido:** `NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK`
    

## 🏁 Fase de Explotación y Obtención del Flag

La cadena que recuperaste está codificada en **Base64** (reconocible por su juego de caracteres alfanuméricos y estructura). Para dar por concluido el reto forense, el paso final es decodificarla usando la terminal:

Bash

```python
echo "NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK" | base64 -d
```

Al ejecutar ese comando, la terminal te va a devolver el texto oculto en texto plano. Solo debes tomar ese resultado y colocarlo dentro del formato oficial del reto: `picoCTF{texto_decodificado}`.
