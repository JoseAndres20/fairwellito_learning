```python
chmod +x webrecon.sh ./webrecon.sh http://10.10.10.10
```

```bash
#!/bin/bash
# webrecon.sh - Reconocimiento rápido de un sitio web para CTF
# Uso: ./webrecon.sh <URL>   (ej: ./webrecon.sh http://10.10.10.10)

if [ -z "$1" ]; then
    echo "Uso: $0 <URL>"
    echo "Ejemplo: $0 http://10.10.10.10"
    exit 1
fi

URL=$1
# quitar / final si lo pusieron
URL=${URL%/}

# carpeta para guardar todo lo que encuentre
OUT="recon_$(echo $URL | sed 's|https\?://||' | tr '/:' '__')"
mkdir -p "$OUT"

echo "=================================================="
echo " Reconocimiento web -> $URL"
echo " Resultados en: $OUT/"
echo "=================================================="

# -------- 1. Archivos comunes que dan info --------
echo ""
echo "[*] Revisando archivos comunes..."

FILES=(
    "robots.txt"
    "sitemap.xml"
    "humans.txt"
    ".git/HEAD"
    ".git/config"
    ".env"
    ".htaccess"
    "backup.zip"
    "config.php.bak"
    "phpinfo.php"
    "README.md"
    "CHANGELOG.txt"
    "wp-login.php"
    "admin/"
    "server-status"
    "crossdomain.xml"
    ".well-known/security.txt"
)

for f in "${FILES[@]}"; do
    code=$(curl -s -o /dev/null -w "%{http_code}" "$URL/$f")
    if [ "$code" == "200" ] || [ "$code" == "301" ] || [ "$code" == "302" ]; then
        echo "  [+] $f  -> HTTP $code"
        curl -s "$URL/$f" -o "$OUT/$(echo $f | tr '/' '_')"
    fi
done

# -------- 2. Headers y tecnologías --------
echo ""
echo "[*] Headers de respuesta..."
curl -s -I "$URL" | tee "$OUT/headers.txt"

if command -v whatweb &> /dev/null; then
    echo ""
    echo "[*] Tecnologías (whatweb)..."
    whatweb "$URL" | tee "$OUT/whatweb.txt"
fi

# -------- 3. Código fuente de la página principal --------
echo ""
echo "[*] Guardando código fuente de la página principal..."
curl -s "$URL" -o "$OUT/index.html"
grep -Ei "password|api[_-]?key|secret|token|flag|todo|fixme" "$OUT/index.html" && \
    echo "  [!] Se encontraron palabras clave sospechosas arriba"

# -------- 4. Gobuster --------
WORDLIST="/usr/share/seclists/Discovery/Web-Content/common.txt"
if [ ! -f "$WORDLIST" ]; then
    WORDLIST="/usr/share/wordlists/dirb/common.txt"
fi

echo ""
if command -v gobuster &> /dev/null && [ -f "$WORDLIST" ]; then
    echo "[*] Corriendo gobuster (wordlist: $WORDLIST)..."
    gobuster dir -u "$URL" -w "$WORDLIST" -x php,html,txt,bak,zip,js -q -o "$OUT/gobuster.txt"
    echo "[*] Resultados de gobuster guardados en $OUT/gobuster.txt"
else
    echo "[!] gobuster o la wordlist no se encontraron, se salta este paso."
    echo "    Instala gobuster o ajusta la variable WORDLIST en el script."
fi

echo ""
echo "=================================================="
echo " Listo. Revisa la carpeta $OUT/ para todos los resultados"
echo "=================================================="

```