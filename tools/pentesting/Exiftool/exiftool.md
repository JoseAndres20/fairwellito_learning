## 📷 2. El comando `exiftool`

### ¿Para qué sirve?

Sirve para **leer los metadatos** de cualquier archivo (especialmente imágenes y fotografías). Los metadatos son la información oculta que genera el dispositivo al crear el archivo: el modelo de la cámara, el software con el que se editó, el nombre del autor, la fecha exacta de creación y, a veces, hasta las coordenadas GPS de dónde se tomó la foto.

En retos de Análisis Forense u OSINT, la bandera suele estar camuflada dentro del campo de "Comentarios", en los derechos de autor ("Copyright") o el reto te exige saber las coordenadas del lugar.

### Ejemplos de comandos directos para el CTF:


**Ver absolutamente todos los metadatos del archivo:**

```bash
exiftool foto_pista.jpg
```

**Filtrar y buscar metadatos específicos que contengan palabras clave:**

```bash
exiftool foto_pista.png | grep -E -i "comment|author|copyright|flag"
```

Extraer únicamente las coordenadas geográficas (Para retos de OSINT/Geolocalización):
```bash
exiftool foto_pista.jpg | grep -i "GPS"
```