

## title: SecureVault - SHA-256 Length Extension Attack tags: [ctf, web, crypto, length-extension, sha256, flask] category: Web/Crypto difficulty: Medium status: Resuelto

# SecureVault File Sharing — Length Extension Attack

## 📋 Resumen

|Campo|Valor|
|---|---|
|**Servicio**|SecureVault File Sharing API v1.2.0|
|**Stack**|Flask (Python)|
|**Vulnerabilidad**|SHA-256 Length Extension Attack|
|**Herramienta clave**|`hashpumpy`|
|**Flag**|`flag{5301f012595187cc}`|

## 🎯 Contexto

API que genera enlaces de descarga firmados para controlar el acceso a archivos. El código fuente completo estaba disponible (`app.py`), lo que permitió identificar la vulnerabilidad directamente en vez de tener que descubrirla a ciegas.

Endpoints:

- `GET /files` → lista archivos públicos junto con su `token` y `sig` (firma) ya calculados.
- `GET /download?token=...&sig=...` → descarga un archivo si la firma es válida.

Archivo objetivo (ya visible en el código, pero protegido):

```python
if filepath == "private/flag.txt":
    return jsonify({"filename": "private/flag.txt", "content": FLAG})
```

## 🔍 La vulnerabilidad

```python
def sign(message_bytes):
    """Sign a message using SHA256(SECRET_KEY + message)."""
    return hashlib.sha256(SECRET_KEY.encode() + message_bytes).hexdigest()
```

Esto es el patrón inseguro **`Hash(secreto || mensaje)`** usando una función hash de la familia **Merkle–Damgård** (SHA-256, SHA-1, MD5 comparten esta debilidad).

### ¿Por qué es explotable?

SHA-256 procesa la entrada en bloques de 64 bytes. El hash final es literalmente **el estado interno del algoritmo tras procesar el último bloque**. Eso significa que si conoces:

1. Un mensaje válido y su hash → `Hash(secreto || mensaje)`
2. La **longitud** del secreto (no su valor)

... puedes "reanudar" el hashing desde ese estado interno y calcular:

```
Hash(secreto || mensaje || padding || datos_extra)
```

para **cualquier `datos_extra`** que elijas — **sin conocer el secreto**.

### El dato que faltaba, regalado en el código

```python
# Our 16-character API key secures all download tokens
SECRET_KEY = os.environ.get("SECRET_KEY", "????????????????")
```

Longitud del secreto: **16 bytes**. Es el único dato que nos faltaba para ejecutar el ataque.

## 🛠️ Explotación paso a paso

### 1. Obtener un token + firma legítimos

```bash
curl -s https://<host>/files
```

```json
{
  "path": "public/welcome.txt",
  "sig": "eb98dbd9635899e1e02a493d7a1e067b540dc3653c8959099fb62f8e44a439fd",
  "token": "action=download&file=public/welcome.txt"
}
```

### 2. Instalar la herramienta de extensión de longitud

```bash
pip install hashpumpy --break-system-packages
```

### 3. Forjar el nuevo mensaje + firma

```python
import hashpumpy
from urllib.parse import quote

original_token = b"action=download&file=public/welcome.txt"
original_sig   = "eb98dbd9635899e1e02a493d7a1e067b540dc3653c8959099fb62f8e44a439fd"
key_length     = 16                          # dato filtrado en el código fuente
append_data    = b"&file=private/flag.txt"   # lo que queremos inyectar

new_sig, new_message = hashpumpy.hashpump(
    original_sig,
    original_token,
    append_data,
    key_length
)

# El servidor decodifica el token con latin-1, así que codificamos igual
encoded_token = quote(new_message.decode("latin-1"), safe="")
url = f"https://<host>/download?token={encoded_token}&sig={new_sig}"
print(url)
```

`hashpumpy` reconstruye internamente el padding SHA-256 (`\x80` + ceros de relleno + longitud en bits) que se inserta automáticamente entre el mensaje original y los datos añadidos, y calcula la firma correspondiente **sin necesitar `SECRET_KEY`**.

### 4. Por qué funciona el `file=` duplicado

El mensaje forjado queda algo así (simplificado, con el padding binario real en medio):

```
action=download&file=public/welcome.txt<PADDING BINARIO>&file=private/flag.txt
```

El parser del servidor:

```python
def parse_token_params(token_bytes):
    parts = {}
    for segment in token_str.split("&"):
        if "=" in segment:
            key, _, value = segment.partition("=")
            parts[key] = value   # <- claves repetidas: la última pisa a la anterior
    return parts
```

simplemente va llenando un diccionario por `key=value`; como `file` aparece dos veces, **la última ocurrencia sobrescribe la primera** → `params["file"]` termina siendo `private/flag.txt`.

### 5. Resultado

```json
{"content":"flag{5301f012595187cc}","filename":"private/flag.txt"}
```

## ❓ Por qué no bastaba con "solo probar la ruta"

El chequeo de firma ocurre **antes** de mirar el archivo pedido:

```python
if sig != expected_sig:
    return jsonify({"error": "Invalid signature"}), 403   # corta aquí
```

Sin una firma matemáticamente válida para un mensaje que contenga `file=private/flag.txt`, el servidor rechaza la petición sin siquiera evaluar el nombre del archivo. No hay atajo por prueba y error: la única forma de producir esa firma sin conocer `SECRET_KEY` es explotando la debilidad estructural de `Hash(secreto || mensaje)` mediante el ataque de extensión de longitud.

## 🩹 Causa raíz y remediación

**Causa raíz:** concatenar `secreto + mensaje` y aplicar una hash Merkle–Damgård no es una firma criptográfica segura — el diseño no resiste extensión de longitud.

**Remediación — usar HMAC:**

```python
import hmac

def sign(message_bytes):
    return hmac.new(SECRET_KEY.encode(), message_bytes, hashlib.sha256).hexdigest()
```

HMAC anida el secreto con padding interno/externo específicamente para neutralizar este ataque, a diferencia de la concatenación simple.

## 🏷️ Tags relacionados

#length-extension-attack #hashpump #sha256 #hmac #flask #signed-urls #ctf-web