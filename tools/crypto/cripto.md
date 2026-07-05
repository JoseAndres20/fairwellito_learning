
tags:
  - ctf
  - writeup
  - forensics
  - cryptography
  - steganography
  - rsa
platform: picoCTF
difficulty: Easy
---

# Writeup: Steganography & RSA Decryption with Exiftool

## 📝 Descripción del Reto
El reto nos proporciona dos archivos principales en el directorio de trabajo:
1. `image.jpg`: Una imagen aparentemente normal en formato JPEG.
2. `flag.enc`: Un archivo binario que contiene el flag cifrado.

El objetivo es auditar la imagen en busca de información oculta (Esteganografía) para encontrar el vector o la llave que nos permita desencriptar el archivo binario.

---

## 🔍 Paso 1: Reconocimiento y Análisis de Metadatos
El primer paso con cualquier archivo multimedia en un CTF de análisis forense es inspeccionar sus metadatos ocultos utilizando `exiftool`.

```bash
exiftool image.jpg
````

### Hallazgo Clave

Al revisar la salida, el campo binario **`Comment`** contenía una cadena masiva en formato hexadecimal (`2d2d2d2d2d424547494e...`).

Si tomamos los primeros bytes de esa cadena hex y los interpretamos como caracteres ASCII:

- `2d2d2d2d2d` $\rightarrow$ `-----`
    
- `424547494e` $\rightarrow$ `BEGIN`
    

Esto nos indica que el comentario de la imagen oculta una llave criptográfica estructurada de OpenSSL.

## 🛠️ Paso 2: Extracción y Decodificación del Comentario (Hex a Texto)

Para no copiar manualmente toda la cadena de texto hexadecimal, utilizamos `exiftool` con modificadores para extraer únicamente el valor del comentario y lo pasamos por un _pipe_ (`|`) hacia `xxd` para realizar la conversión reversa de hexadecimal puro a datos planos.

Bash

```
exiftool -s3 -Comment image.jpg | xxd -r -p > private.key
```

> [!NOTE] Desglose del comando:
> 
> - `-s3`: Extrae únicamente el valor del tag de metadato, omitiendo el nombre del campo y los espacios en blanco.
>     
> - `xxd -r -p`: El flag `-r` (reverse) convierte hex a binario/texto, y `-p` (plain) le dice que procese un volcado de hex continuo sin formato de columnas.
>     
> - `> private.key`: Redirecciona la salida para guardar la estructura de la llave RSA privada en un archivo limpio.
>     

Si inspeccionamos el archivo generado con `cat private.key`, veremos la estructura correcta:

Plaintext

```
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCyi2qh7k1+l1Q7
...
-----END PRIVATE KEY-----
```

## 🔓 Paso 3: Desencriptación de Datos Asimétricos (RSA)

Contando con la llave privada (`private.key`) y sabiendo que el archivo `flag.enc` está cifrado mediante RSA, utilizamos la utilidad de bajo nivel `pkeyutl` de OpenSSL para realizar la operación de descifrado.

Bash

```
openssl pkeyutl -decrypt -inkey private.key -in flag.enc
```

> [!TIP] Parámetros utilizados:
> 
> - `-decrypt`: Especifica que la operación a realizar es un descifrado de datos.
>     
> - `-inkey private.key`: Define el archivo que contiene la llave privada correspondiente.
>     
> - `-in flag.enc`: Indica el archivo de entrada cifrado que queremos procesar.
>     

## 🏁 Bandera Obtenida

Al ejecutar el comando de descifrado, la terminal expone el texto en claro directamente:

Plaintext

```
picoCTF{rs4_k3y_1n_1mg_a9a7c4c9}
```

## 🧠 Lecciones Aprendidas

1. **Los comentarios JPEG no siempre son texto:** Los atacantes o desarrolladores suelen utilizar campos de comentarios o metadatos estándar (`EXIF`, `XMP`, `IPTC`) para ocultar payloads, scripts o llaves comprometidas debido a que no alteran la renderización visual de la imagen.
    
2. **Firmas de archivos comunes:** Recordar los bytes mágicos iniciales en Hex. Ver un bloque que empiece por `2d2d2d2d2d` en un contexto de llaves es un indicador inmediato de guiones (`-----`).
    

```
---
```