---
# Deadbolt — Writeup

|**Categoría**|Mobile|
|**Dificultad**|Hard|
|**Puntos**|—|
|**Plataforma**|fluidattacks.ctf.ae|
|**Flag**|`flag{7cc93002f31a2233}`|

![[Pasted image 20260704155054.png]]

## 🧭 TL;DR

El APK de la app tiene un `SEED` y `SALT` hardcodeados para derivar la clave AES. El "device ID" usado en la derivación también es reproducible porque el reto indica que no se necesita un dispositivo real. Con esos tres datos se reconstruye la clave AES-256-GCM, se descifra `vault_backup.enc`, se extrae la contraseña del admin portal, y se obtiene la flag autenticándose contra la API.

---

## 🎯 Objetivo

Descifrar `vault_backup.enc` y usar las credenciales dentro para autenticarse contra la API del reto.

## 🔍 1. Análisis del código decompilado

En `CryptoManager.java` la clave AES se deriva así:

```java
String passphrase = SEED + deviceId;
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
KeySpec spec = new PBEKeySpec(passphrase.toCharArray(), SALT, ITERATIONS, KEY_LENGTH);
```

- `SEED = "fluidvault_master_seed_2026"` (hardcodeado — vulnerabilidad #1: secreto embebido en el APK)
- `SALT = a915c3e00dbb5a55ba13d7cdaf3a126e` (también hardcodeado)
- `deviceId = Integer.toHexString(Build.SERIAL.hashCode())`, con fallback a `"DEFAULT_DEVICE_ID"` si `Build.SERIAL` es `null` o `"unknown"`.

## 💡 2. La pista clave del reto

El challenge dice explícitamente **"No device or emulator required"**. En Android moderno `Build.SERIAL` siempre devuelve `"unknown"` sin el permiso legacy, así que el backup se generó usando el fallback `DEFAULT_DEVICE_ID`. Eso elimina la necesidad de tener un dispositivo real: el "device ID" es constante y predecible.

## 🔑 3. Reproducir la derivación en Python

```python
import hashlib

# Java String.hashCode() de "DEFAULT_DEVICE_ID"
s = "DEFAULT_DEVICE_ID"
h = 0
for c in s:
    h = (h * 31 + ord(c)) & 0xFFFFFFFF
device_id_hex = hex(h)[2:]        # -> c8e21b06

passphrase = ("fluidvault_master_seed_2026" + device_id_hex).encode()
salt = bytes.fromhex("a915c3e00dbb5a55ba13d7cdaf3a126e")
key = hashlib.pbkdf2_hmac("sha256", passphrase, salt, 10000, dklen=32)
```

Esto reproduce exactamente el `PBEKeySpec` de Java (mismos parámetros: SHA-256, 10000 iteraciones, 256 bits).

## 🔓 4. Descifrar el backup

El formato es `IV(12) || ciphertext || tag(16)` en AES-256-GCM:

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
blob = open("vault_backup.enc","rb").read()
iv, ct = blob[:12], blob[12:]
plaintext = AESGCM(key).decrypt(iv, ct, None)
```

Esto reveló el JSON del vault, incluida la entrada **"Admin Portal"** con `password: Fl00d_G4t3s_0p3n!`.

## 🚀 5. Usar la contraseña contra la API

`VaultApiClient.java` mostraba el endpoint real:

```
POST /api/vault/admin
Body: {"master_password": "..."}
```

```bash
curl -k -X POST https://ea3d4041d2baa078.chal.ctf.ae/api/vault/admin \
  -H "Content-Type: application/json" \
  -d '{"master_password": "Fl00d_G4t3s_0p3n!"}'
```

(el `-k` fue necesario porque el certificado TLS de la instancia había expirado, no relacionado con la vulnerabilidad del reto en sí)

**Respuesta:**

```json
{"authenticated":true,"flag":"flag{7cc93002f31a2233}","role":"admin"}
```

## 🩹 Vulnerabilidad raíz

Clave de cifrado derivada de un **secreto hardcodeado + salt hardcodeado + un "device ID" trivialmente reproducible** — cero entropía real por dispositivo o usuario. Cualquiera con el APK puede descifrar cualquier backup sin necesidad de biometría, dispositivo físico ni fuerza bruta.