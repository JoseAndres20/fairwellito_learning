# 🎯 Nmap - Escaneos Enfocados por Objetivo

Comando específico para cada situación. Directo.

---

## 🔍 Escaneos por Objetivo

### Escaneo Completo (CTF/Labs)
```bash
sudo nmap -p- -sV -sC --open -sS -vvv -n -Pn --min-rate=1000 TARGET -oA full_scan
```
**Para qué:** Escanear TODO rápido en laboratorios/CTFs.

---

### Escaneo Inicial Rápido
```bash
nmap -sS --top-ports 1000 --open TARGET -oN quick.txt
```
**Para qué:** Primer reconocimiento, ver qué hay abierto.

---

### Escaneo Web (HTTP/HTTPS)
```bash
nmap -sS -sV -p 80,443,8080,8443 --script=http-enum TARGET
```
**Para qué:** Enumerar servicios web, buscar directorios.

---

### Escaneo SMB/Windows
```bash
nmap -sS -p 445,139,135 --script=smb-enum-shares,smb-os-discovery TARGET
```
**Para qué:** Enumerar shares SMB y detectar Windows.

---

### Escaneo SSH
```bash
nmap -sS -p 22 --script=ssh-auth-methods,ssh-hostkey TARGET
```
**Para qué:** Ver métodos de autenticación SSH disponibles.

---

### Escaneo FTP
```bash
nmap -sS -p 21 --script=ftp-anon,ftp-bounce TARGET
```
**Para qué:** Verificar FTP anónimo y vulnerabilidades.

---

### Escaneo DNS
```bash
nmap -sU -p 53 --script=dns-zone-transfer,dns-recursion TARGET
```
**Para qué:** Intentar transferencia de zona DNS.

---

### Escaneo de Vulnerabilidades
```bash
nmap -sV --script=vuln -p PORTS TARGET -oN vulns.txt
```
**Para qué:** Buscar CVEs conocidos en servicios.

---

### Escaneo MySQL/MSSQL
```bash
nmap -sS -p 3306,1433 --script=mysql-info,ms-sql-info TARGET
```
**Para qué:** Enumerar bases de datos MySQL y MSSQL.

---

### Escaneo RDP
```bash
nmap -sS -p 3389 --script=rdp-enum-encryption,rdp-vuln-ms12-020 TARGET
```
**Para qué:** Verificar RDP y vulnerabilidad BlueKeep.

---

### Discovery de Red (Hosts Activos)
```bash
nmap -sn 192.168.1.0/24 -oG - | grep "Up" | cut -d" " -f2
```
**Para qué:** Listar todas las IPs activas en la red.

---

### Escaneo UDP (Puertos Comunes)
```bash
sudo nmap -sU --top-ports 100 TARGET -oN udp.txt
```
**Para qué:** Escanear servicios UDP (DNS, SNMP, TFTP).

---

### Escaneo Stealth (No Ser Detectado)
```bash
sudo nmap -sS -Pn -f -D RND:10 --source-port 53 -T1 --top-ports 100 TARGET
```
**Para qué:** Reconocimiento sigiloso sin activar IDS/IPS.

---

### Escaneo de Red Team (Máximo Sigilo)
```bash
sudo nmap -sS -Pn --scan-delay 15s -T0 --top-ports 50 TARGET -oN redteam.txt
```
**Para qué:** Pentesting corporativo, evitar detección SOC.

---

### Escaneo LDAP (Active Directory)
```bash
nmap -sS -p 389,636,3268 --script=ldap-rootdse TARGET
```
**Para qué:** Enumerar información de Active Directory.

---

### Escaneo SNMP
```bash
nmap -sU -p 161 --script=snmp-info,snmp-brute TARGET
```
**Para qué:** Enumerar información SNMP del dispositivo.

---

### Escaneo de Heartbleed (OpenSSL)
```bash
nmap -sS -p 443 --script=ssl-heartbleed TARGET
```
**Para qué:** Verificar vulnerabilidad Heartbleed en HTTPS.

---

### Escaneo EternalBlue (MS17-010)
```bash
nmap -sS -p 445 --script=smb-vuln-ms17-010 TARGET
```
**Para qué:** Detectar vulnerabilidad EternalBlue en Windows.

---

### Escaneo Shellshock
```bash
nmap -sS -p 80 --script=http-shellshock --script-args uri=/cgi-bin/test.sh TARGET
```
**Para qué:** Verificar vulnerabilidad Shellshock en CGI.

---

### Escaneo de Todos los Scripts NSE
```bash
nmap -sV --script="default,discovery,safe" -p PORTS TARGET -oN full_scripts.txt
```
**Para qué:** Ejecutar TODOS los scripts seguros en servicios.

---

### Escaneo Masivo (Múltiples Targets)
```bash
nmap -sS --top-ports 1000 -iL targets.txt -oA mass_scan
```
**Para qué:** Escanear lista de hosts desde archivo.

---

### Escaneo con Proxychains (Anonimato)
```bash
proxychains nmap -sT -Pn -p 80,443 TARGET
```
**Para qué:** Rutear escaneo por Tor/VPN para anonimato.

---

### Escaneo Idle (Zombi)
```bash
sudo nmap -sI ZOMBIE_IP -Pn TARGET
```
**Para qué:** Tu IP nunca aparece, usa host zombi.

---

### Escaneo de Puerto Específico (Profundo)
```bash
nmap -sV --version-intensity 9 -sC -p PORT TARGET
```
**Para qué:** Máxima información sobre UN puerto específico.

---

### Escaneo IPv6
```bash
nmap -6 -sS -p 80,443 TARGET_IPv6
```
**Para qué:** Escanear hosts con dirección IPv6.

---

### Escaneo con Output XML (Para Herramientas)
```bash
nmap -sS -sV -p- TARGET -oX scan.xml
```
**Para qué:** Parsear resultados con otras herramientas (Metasploit, etc).

---

## 📊 Tabla Rápida

| Objetivo               | Comando Clave                   | Velocidad  |
| ---------------------- | ------------------------------- | ---------- |
| Escaneo completo       | `-p- -sV -sC --min-rate=1000`   | Rápida     |
| Reconocimiento inicial | `--top-ports 1000`              | Muy rápida |
| Web enumeration        | `-p 80,443 --script=http-enum`  | Media      |
| Windows/SMB            | `-p 445,139 --script=smb-enum*` | Media      |
| Stealth                | `-f -D RND:10 -T1`              | Lenta      |
| Vulnerabilidades       | `--script=vuln`                 | Media      |
| Discovery red          | `-sn 192.168.1.0/24`            | Rápida     |
| UDP scan               | `-sU --top-ports 100`           | Lenta      |

---

**Autor:** José Andrés Acuña Rodríguez  
**Última actualización:** Octubre 2025