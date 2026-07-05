# Phantom Decrypt — Writeup

**Categoría:** Cryptography **Dificultad:** Medium
![[Pasted image 20260704100645.png]]
## Resumen

SecureVault cifra las sesiones de usuario con AES-GCM, un modo de cifrado autenticado que en teoría es muy seguro: si alguien manipula el texto cifrado, la verificación del "tag" debería fallar y rechazar la sesión.

El problema no está en el algoritmo de cifrado en sí, sino en **qué hace el servidor cuando esa verificación falla**.

## El código vulnerable

```python
def decrypt_session(cookie: str) -> dict:
    try:
        raw = base64.b64decode(cookie)
        nonce = raw[:12]
        tag = raw[12:28]
        ciphertext = raw[28:]
        cipher = AES.new(SECRET_KEY, AES.MODE_GCM, nonce=nonce)
        plaintext = cipher.decrypt_and_verify(ciphertext, tag)
        return json.loads(plaintext)
    except Exception:
        # <-- BUG: fallback inseguro
        session_data = json.loads(base64.b64decode(cookie))
        return session_data
```

`decrypt_and_verify` lanza una excepción cuando el "tag" de autenticación no coincide con el texto cifrado — esto es exactamente lo que se espera de un cifrado autenticado, y es correcto.

El problema es el bloque `except`: en lugar de rechazar la cookie cuando falla la verificación, el código intenta "recuperarse" interpretando el mismo contenido **directamente como JSON en texto plano**, sin ningún tipo de verificación criptográfica. Es un mecanismo de fallback pensado probablemente para tolerar errores, pero que en la práctica anula por completo la protección de AES-GCM.

## Por qué esto es explotable

Si el atacante manda una cookie que:

1. Es un base64 válido (para que `base64.b64decode` no falle), y
2. Al decodificarse en base64, el contenido **no** pasa la verificación de AES-GCM (lo cual es casi seguro si no conocemos la clave secreta), pero
3. Al decodificar ese mismo base64 se obtiene un JSON válido...

...entonces el servidor termina aceptando datos de sesión **inventados por el atacante**, incluyendo el campo `role`, sin haber demostrado en ningún momento conocer la clave secreta del servidor.

## Explotación paso a paso

1. Construir el JSON de sesión deseado, con `role: admin`:
    
    ```json
    {"user": "attacker", "role": "admin", "iat": 1783180789}
    ```
    
2. Codificarlo en base64 (sin cifrar, a propósito):
    
    ```python
    import base64, json, time
    
    data = {"user": "attacker", "role": "admin", "iat": int(time.time())}
    token = base64.b64encode(json.dumps(data).encode()).decode()
    print(token)
    ```
    
    Resultado:
    
    ```
    eyJ1c2VyIjogImF0dGFja2VyIiwgInJvbGUiOiAiYWRtaW4iLCAiaWF0IjogMTc4MzE4MDc4OX0=
    ```
    
3. Reemplazar la cookie `session_token` del navegador (DevTools → Application → Cookies) por este valor.
    
4. Recargar `/dashboard`. El servidor intenta descifrar la cookie con AES-GCM, falla (porque no es un blob cifrado real), cae en el `except`, decodifica el JSON falso directamente, y confía en que `role == "admin"`.
    
5. El servidor entonces revela la sección restringida:
    
    ```html
    <p><b>Admin Recovery Key:</b> flag{...}</p>
    ```
    

## Causa raíz

El bug no es criptográfico — AES-GCM en sí funciona correctamente y su clave nunca se compromete. El bug es de **diseño de manejo de errores**: un fallback "silencioso" que trata un fallo de autenticación como una razón para probar un camino _menos_ seguro, en lugar de simplemente rechazar la solicitud.

## Lección / remediación

- Un fallo en la verificación de un MAC/tag de AEAD (GCM, Poly1305, etc.) debe terminar en rechazo total de la entrada — nunca en un intento de "interpretar" los mismos datos de otra forma.
- Cualquier dato de sesión que determine privilegios (como `role`) debe provenir **únicamente** de una fuente autenticada criptográficamente, nunca de un fallback de "mejor esfuerzo".
- El bloque `except Exception` demasiado amplio ocultó la verdadera naturaleza del problema; capturar solo la excepción específica de fallo de verificación (y manejarla como error, no como dato válido) habría evitado esta vulnerabilidad.