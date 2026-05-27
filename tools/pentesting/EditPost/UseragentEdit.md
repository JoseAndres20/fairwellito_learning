## 🌐 2. Falsificación de Identidad Web (User-Agent)

### Ejemplos de User-Agent según el escenario del reto:

- **Para simular el Bot de Google:**
    

```
User-Agent: Googlebot/2.1 (+http://www.google.com/bot.html)
```

- **Para simular Internet Explorer 6 (Navegador antiguo):**
    

```
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)
```

- **Para simular un iPhone (Dispositivo móvil):**
    

```
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0 Mobile/15E148 Safari/604.1
```

### Ejemplo de modificación en Burp Suite:

HTTP

```
GET /admin HTTP/1.1
Host: reto.hackrocks.com
User-Agent: Googlebot/2.1 (+http://www.google.com/bot.html)
Accept: text/html
```