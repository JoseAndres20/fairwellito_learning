**Binwalk** es una herramienta de análisis forense diseñada para buscar, analizar y extraer archivos ocultos dentro de otros archivos (archivos binarios, imágenes, firmware, etc.). En un CTF, es tu herramienta principal cuando sospechas que un archivo (como un `.jpg` o un `.png`) tiene algo "pegado" al final o escondido en su interior, como un archivo `.zip` o un ejecutable.

#### Comandos básicos para Binwalk

Analizar el archivo:

>_Uso:_ Esto escanea el archivo y te muestra una tabla con los resultados, identificando si hay cabeceras de otros archivos ocultos dentro (ej: "Zip archive", "PNG image", etc.).

```python
binwalk nombre_archivo.ext
```

#### Extraer archivos ocultos

>_Uso:_ La `-e` (de _extract_) extrae automáticamente todos los archivos encontrados. Si encuentra un ZIP, creará una carpeta con el contenido de ese ZIP dentro.

```python
binwalk -e nombre_archivo.ext
```

#### Extraer y mantener la estructura:

>_Uso:_ La `-M` (_matryoshka_) le dice a `binwalk` que sea recursivo. Si extrae un archivo y dentro de ese archivo encuentra otro, seguirá extrayendo automáticamente.

```python
binwalk -e -M nombre_archivo.ext
```