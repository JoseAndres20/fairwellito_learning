# 🕵️ Nmap - Reconocimiento Sigiloso (Stealth)

Guía de comandos Nmap para reconocimiento sin ser detectado fácilmente por IDS/IPS.

---

## 📖 ¿Qué es Nmap?

**Nmap** (Network Mapper) es la herramienta de escaneo de redes más popular y poderosa del mundo. Permite:

- 🔍 **Descubrir hosts activos** en una red
- 🚪 **Identificar puertos abiertos** y servicios corriendo
- 🖥️ **Detectar sistemas operativos** y versiones de software
- 🛡️ **Auditar seguridad** de redes y sistemas
- 🕵️ **Realizar reconocimiento** en pentesting

**Creado por:** Gordon Lyon (Fyodor)  
**Licencia:** Open Source (GPL)  
**Plataformas:** Linux, Windows, macOS

---

### Instalación

**Linux (Debian/Ubuntu):**
```bash
sudo apt update
sudo apt install nmap -y
```

**Kali Linux:**
```bash
# Ya viene preinstalado
nmap --version
```

**Verificar instalación:**
```bash
nmap -V
```

---

### Sintaxis Básica

```bash
nmap [Tipo de Escaneo] [Opciones] [Target]
```

**Ejemplo simple:**
```bash
nmap 192.168.1.1
```

---

## 🎯 Objetivo de Esta Guía

Realizar escaneos de reconocimiento **minimizando las probabilidades de detección** por sistemas de seguridad, firewalls o IDS/IPS.

---

## ⚠️ Advertencia Legal

**IMPORTANTE:** Estos comandos solo deben usarse en:
- Entornos de práctica autorizados (TryHackMe, HackTheBox, laboratorios propios)
- Sistemas donde tengas permiso explícito por escrito
- **NUNCA** en redes o sistemas sin autorización (es ilegal)

---

## 🛠️ Comandos Esenciales Stealth

### 🔰 Escaneo Básico (Para Comparar)

**Escaneo normal (RUIDOSO - te detectarán):**
```bash
nmap 192.168.1.100
```

**Problema:** Completa handshake TCP, genera logs, fácil de detectar.

---

### 1. SYN Scan (Escaneo Sigiloso Básico)

**El más usado para evadir detección:**
```bash
sudo nmap -sS 10.64.183.181
```

**¿Por qué es sigiloso?**
- No completa el handshake TCP (envía SYN, recibe SYN-ACK, pero no envía ACK final)
- Muchos sistemas no lo registran como conexión completa
- Requiere privilegios root/admin

---

### 2. Escaneo SYN con Timing Lento

**Reducir velocidad para evadir IDS:**
```bash
sudo nmap -sS -T2 192.168.1.100
```

**Niveles de timing:**
- `-T0` (Paranoid): Extremadamente lento, casi indetectable
- `-T1` (Sneaky): Muy lento, evita IDS
- `-T2` (Polite): Lento, reduce carga de red
- `-T3` (Normal): Velocidad por defecto
- `-T4` (Aggressive): Rápido
- `-T5` (Insane): Muy rápido, fácil de detectar

---

### 3. Fragmentación de Paquetes

**Dividir paquetes para evadir firewalls:**
```bash
sudo nmap -sS -f 192.168.1.100
```

**Con fragmentación más agresiva:**
```bash
sudo nmap -sS -f -f 192.168.1.100
```

---

### 4. Decoy Scan (IP Señuelo)

**Ocultar tu IP real entre IPs falsas:**
```bash
sudo nmap -sS -D RND:10 192.168.1.100
```

**Especificar IPs señuelo manualmente:**
```bash
sudo nmap -sS -D 192.168.1.5,192.168.1.8,ME,192.168.1.12 192.168.1.100
```

- `RND:10` = 10 IPs aleatorias como señuelo
- `ME` = Tu IP real (se mezcla con las falsas)

---

### 5. Cambiar Source Port

**Usar un puerto origen específico (ej. 53 DNS, 80 HTTP):**
```bash
sudo nmap -sS --source-port 53 192.168.1.100
```

**¿Por qué funciona?**
- Algunos firewalls permiten tráfico desde puertos "confiables" (53, 80, 443)

---

### 6. Idle Scan (Zombie Scan)

**El más sigiloso: usar una máquina intermedia "zombi":**
```bash
sudo nmap -sI 192.168.1.50 192.168.1.100
```

- `192.168.1.50` = Host "zombi" (debe estar idle)
- `192.168.1.100` = Target real
- **Tu IP nunca aparece en los logs del target**

---

### 7. Desactivar Ping (No Ping)

**Escanear sin hacer ping previo:**
```bash
sudo nmap -sS -Pn 192.168.1.100
```

**Útil cuando:**
- El firewall bloquea ICMP
- Hosts con ping deshabilitado

---

### 8. Escaneo de Puertos Específicos

**Solo escanear puertos clave:**
```bash
sudo nmap -sS -p 22,80,443,3389 192.168.1.100
```

**Top 100 puertos más comunes:**
```bash
sudo nmap -sS --top-ports 100 192.168.1.100
```

---

## 🧪 Ejemplo Práctico Completo

### Escenario: Escanear un servidor web sin ser detectado

**Target:** `10.10.10.50` (servidor web en TryHackMe)

**Paso 1:** Verificar si el host está activo (sin ping)
```bash
sudo nmap -sn -Pn 10.10.10.50
```

**Paso 2:** Escaneo sigiloso de puertos web
```bash
sudo nmap -sS -p 80,443,8080,8443 -T2 10.10.10.50
```

**Paso 3:** Detectar versiones (lento para evadir IDS)
```bash
sudo nmap -sS -sV -p 80,443 -T1 --version-intensity 2 10.10.10.50
```

**Paso 4:** Guardar resultados
```bash
sudo nmap -sS -sV -p 80,443 -T1 -oN scan_results.txt 10.10.10.50
```

**Output esperado:**
```
PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.41
443/tcp open  ssl/http Apache httpd 2.4.41
```

---

## 🎭 Combinación Stealth Avanzada

### Reconocimiento Ultra Sigiloso
```bash
sudo nmap -sS -Pn -f -D RND:5 --source-port 53 -T1 --top-ports 50  10.64.183.181
```

**Explicación:**
- `-sS` = SYN scan
- `-Pn` = Sin ping
- `-f` = Fragmentación
- `-D RND:5` = 5 IPs señuelo aleatorias
- `--source-port 53` = Puerto origen 53 (DNS)
- `-T1` = Timing muy lento
- `--top-ports 50` = Solo 50 puertos más comunes

---

### Escaneo Completo pero Lento
```bash
sudo nmap -sS -sV -O -Pn -T2 --max-retries 2 192.168.1.100
```

**Explicación:**
- `-sV` = Detectar versiones de servicios
- `-O` = Detectar sistema operativo
- `--max-retries 2` = Solo 2 reintentos (menos ruido)

---

## 🚫 Técnicas de Evasión Adicionales

### 1. Randomizar orden de hosts
```bash
sudo nmap -sS --randomize-hosts 192.168.1.0/24
```

### 2. Agregar datos aleatorios
```bash
sudo nmap -sS --data-length 25 192.168.1.100
```

### 3. Usar MAC address falso (Linux)
```bash
sudo nmap -sS --spoof-mac Apple 192.168.1.100
```

---

## 📊 Comparación de Técnicas

| Técnica | Nivel Stealth | Velocidad | Uso Común |
|---------|---------------|-----------|-----------|
| SYN Scan `-sS` | ⭐⭐⭐ | Rápida | Escaneo base |
| Timing `-T1` | ⭐⭐⭐⭐⭐ | Muy lenta | Evasión IDS |
| Fragmentación `-f` | ⭐⭐⭐⭐ | Media | Evadir firewalls |
| Decoy `-D` | ⭐⭐⭐⭐ | Media | Ocultar IP |
| Idle Scan `-sI` | ⭐⭐⭐⭐⭐ | Lenta | Anonimato total |

---

## 🔍 Detección de Escaneos

**Para saber si te detectaron, revisa logs del sistema (en entornos autorizados):**

```bash
# Linux
sudo tail -f /var/log/syslog

# Ver conexiones sospechosas
sudo netstat -antp | grep SYN
```

---

## 📚 Recursos Adicionales

- [Nmap Official Docs](https://nmap.org/book/man.html)
- [Nmap Stealth Techniques](https://nmap.org/book/man-bypass-firewalls-ids.html)
- Libro: "Nmap Network Scanning" by Gordon Lyon

---

## 🏁 Checklist Pre-Escaneo

```markdown
- [ ] Tengo autorización escrita para escanear
- [ ] Estoy en un entorno legal (lab, CTF, red propia)
- [ ] Sé qué técnica usar según el objetivo
- [ ] Tengo documentado el escaneo (fecha, hora, propósito)
- [ ] Guardé los resultados en archivo (-oN output.txt)
```

---

**Autor:** José Andrés Acuña Rodríguez  
**Categoría:** Herramientas - Reconocimiento  
**Última actualización:** Octubre 2025

---

**Recuerda:** El reconocimiento sigiloso es una habilidad, pero debe usarse éticamente y con autorización. La responsabilidad es tuya.