





>Un **programa** te espera en silencio...  
No dice mucho, pero sabe si lo que ingresas es correcto.  
Algo fue ocultado con una técnica simétrica, repetitiva, predecible.  
Tal vez si escuchas lo que dice en sus entrañas, la verdad se revelará.  
¿Podrás abrir el candado sin romperlo?  
La clave está ahí... sólo hay que mirar con los ojos correctos.

#### Tools

http://github.com/extremecoders-re/pyinstxtractor   extraer datos de ejecutables

```python
python3 pyinstxtractor-master/pyinstxtractor.py reto  
```

https://www.decompiler.com/   ver archivos pyc


![[Pasted image 20260715194641.png]]



```python
# Source Generated with Decompyle++

# File: reto.pyc (Python 3.6)

from Crypto.Cipher import AES

import base64

clave = b'clave_secreta_16'

cifrado_b64 = '9xVWXEWN4EsZ9ss/87d5DuHA6K/E8YxlYIss7bu3PVA='

def unpad(s):

return s[:-s[-1]]

def descifrar():

cipher = AES.new(clave, AES.MODE_ECB)

return unpad(cipher.decrypt(base64.b64decode(cifrado_b64))).decode()

def main():

entrada = input('Introduce la flag: ').strip()

print(descifrar()) ##Linea añadida para mostrar la flag descifrada

if entrada == descifrar():

print('¡Correcto! Flag desbloqueada 🎉')

else:

print('Incorrecto. Intenta de nuevo.')

if __name__ == '__main__':

main()
```

Encontramos un file pyc vemos el code editamos el cod epara que nos de la flag listo