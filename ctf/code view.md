---
tags: [CTF, Criptografía, AES, FuerzaBruta, Python, Pwntools]
fecha: 2026-05-23
plataforma: HackRocks
estado: Resuelto
---

# 🛡️ CTF Writeup: Crypto - Doble AES Vulnerable

## 📝 Resumen del Reto
El reto nos presenta un servicio que cifra un mensaje (la flag) utilizando un sistema de **Doble AES**. El texto plano se encripta primero con una llave secundaria ($K_1$) y el resultado se vuelve a encriptar con una llave principal ($K_2$) en modo ECB. 

* **Fórmula Matemática:** `C = E_K2(E_K1(P))`
* **Objetivo:** Obtener la flag real aprovechando la debilidad en la generación de estas llaves.

## 🐛 Análisis de la Vulnerabilidad
Analizando el código fuente interceptado, la implementación criptográfica contiene fallos críticos que destruyen la seguridad del algoritmo:

1. **Llave 1 (Estática y Pública):** La variable `first_key` no es secreta, está declarada directamente en el código fuente como `b'9123891238912389'`[cite: 2].
2. **Llave 2 (Espacio de búsqueda ínfimo):** La variable `second_key` se genera a partir de un simple número pseudoaleatorio (`random.randint`) restringido entre `0` y `99999`[cite: 2].
3. **Mala entropía:** Ese número aleatorio simplemente se rellena con ceros, se multiplica y se trunca a 16 bytes[cite: 2]. Esto significa que existen, como máximo absoluto, **100,000 combinaciones posibles** para $K_2$, lo que hace que un ataque de fuerza bruta offline sea trivial.

## ⚔️ Estrategia de Explotación
Dado que el espacio de llaves es minúsculo, no es necesario realizar criptoanálisis avanzado:
1. Conectarnos al servidor remoto mediante `pwntools` para extraer la flag cifrada (en formato hexadecimal).
2. Desconectar inmediatamente para evitar ruidos en el servidor.
3. Iterar offline a través de los 100,000 números posibles, generar la llave temporal `K2` con la misma lógica vulnerable y probar la doble desencriptación.
4. Remover el *padding* PKCS7 y validar si el resultado contiene la cabecera estándar `flag{`.

## 💻 Exploit (`solve_auto.py`)
**Dependencias requeridas:** `pwntools`, `pycryptodome`
```bash
sudo apt install python3-pwntools python3-pycryptodome -y
```
Python
```python
from pwn import *
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad

def main():
    # 1. Extracción automatizada
    print("[*] Estableciendo conexión con challenges.hackrocks.com:1100...")
    context.log_level = 'error' 
    io = remote('challenges.hackrocks.com', 1100)

    io.recvuntil(b"choose: ")
    io.sendline(b"1")
    io.recvuntil(b"the flag: ")
    encrypted_flag_hex = io.recvline().strip().decode()
    io.close()

    print(f"[+] Flag capturada: {encrypted_flag_hex[:30]}...")
    print("[*] Iniciando ataque de fuerza bruta offline sobre K2...")

    # 2. Configuración de ataque
    first_key = b'9123891238912389'
    encrypted_flag = bytes.fromhex(encrypted_flag_hex)

    # 3. Fuerza Bruta (Máx 100,000 iteraciones)
    for i in range(100000):
        second_key = (str(i).zfill(5)*4)[:16].encode()
        
        try:
            # Reversar K2 (Capa exterior)
            cipher2 = AES.new(second_key, mode=AES.MODE_ECB)
            intermediate = cipher2.decrypt(encrypted_flag)
            
            # Reversar K1 (Capa interior)
            cipher1 = AES.new(first_key, mode=AES.MODE_ECB)
            plaintext_padded = cipher1.decrypt(intermediate)
            
            # Limpieza y validación
            plaintext = unpad(plaintext_padded, 16)
            
            if b"flag{" in plaintext:
                print("\n" + "="*50)
                print(f"[+] ¡VULNERABILIDAD EXPLOTADA CON ÉXITO!")
                print(f"[+] Llave K2 del servidor: {second_key.decode()}")
                print(f"[+] FLAG: {plaintext.decode()}")
                print("="*50 + "\n")
                break
                
        except Exception:
            # Padding inválido = Llave incorrecta
            continue

if __name__ == '__main__':
    main()
```