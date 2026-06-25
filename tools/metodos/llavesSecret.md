

![[Pasted image 20260624145809.png]]
# 🧠 Análisis Técnico: Desencriptación de Diffie-Hellman Vuelto Mierda

## 📌 Contexto Criptográfico

El protocolo **Diffie-Hellman (DH)** es un sistema de intercambio de llaves asimétrico. Su seguridad radica en el **Problema del Logaritmo Discreto**: es fácil calcular las llaves públicas a partir de las privadas, pero computacionalmente imposible hacer lo inverso si el número primo $p$ es lo suficientemente grande (como este de 1048 bits).

### La Vulnerabilidad del Reto

En un escenario real, el secreto privado del cliente ($b$) **nunca se comparte**. El script del reto es vulnerable porque expone directamente el valor de $b$ en el archivo de texto. Al tener la llave pública del servidor ($A$) y la llave privada del cliente ($b$), podemos romper la ecuación sin necesidad de crackear el logaritmo discreto.

## 🛠️ Explicación del Código Paso a Paso

### 1. Preparación de los Datos y Conversión Hexadecimal

Python

```bash 
enc_hex = "4d545e527e697b465955624e0e5e4f0e49620404050f5b5b580b40"
enc = bytes.fromhex(enc_hex)
```

- **¿Qué pasa aquí?**: El flag encriptado viene representado como una cadena de texto en hexadecimal (`str`).
    
- **Por qué se hace**: Para poder operar matemáticamente los bytes del flag, necesitamos transformar ese string en un objeto de bytes puros de Python usando `bytes.fromhex()`. Cada par de caracteres hex (ej. `4d`) se convierte en un número entero entre 0 y 255 (un byte).
    

### 2. Cálculo del Secreto Compartido (Shared Secret)

Python

```bash
shared = pow(A, b, p)
```

Esta es la línea donde ocurre la magia matemática de Diffie-Hellman.

- **La Fórmula**:
    
    $$shared = A^b \pmod p$$
    
- **Por qué funciona**: La llave pública del servidor es $A = g^a \pmod p$. Al elevarla a nuestro secreto $b$, estamos calculando:
    
    $$(g^a)^b \pmod p \equiv g^{ab} \pmod p$$
    
    El servidor haría exactamente lo mismo elevando nuestra pública $B$ a su secreto $a$ ($B^a \pmod p$), llegando al mismísimo número secreto sin que nadie más en la red pueda interceptarlo.
    

> [!IMPORTANT]
> 
> **Optimización en Python:** Usar `pow(A, b, p)` no es lo mismo que hacer `(A ** b) % p`. Si intentás hacer `A ** b` con números de 1048 bits, la computadora se va a quedar sin memoria RAM y se va a congelar antes de terminar. La función nativa `pow(base, exp, mod)` implementa un algoritmo llamado **Exponenciación Binaria (Square-and-Multiply)**, que va reduciendo el número bajo el módulo $p$ en cada paso del cálculo, haciéndolo en milisegundos.

### 3. Derivación de la Llave Simétrica (Reducción a 1 Byte)

Python

```bash
key = shared % 256
```

- **¿Qué pasa aquí?**: El resultado de `shared` es un número colosal de cientos de dígitos. Sin embargo, el script original del reto aplicó una operación de truncado para la encriptación.
    
- **El truco**: Al aplicar el operador módulo 256 (`% 256`), estamos extrayendo únicamente el **último byte** (los 8 bits menos significativos) de ese número gigantesco. Esto nos da un entero limpio entre `0` y `255`, que será nuestra llave única para el cifrado de flujo por XOR.
    

### 4. Descifrado por Operación XOR (Exclusivo OR)

Python

```bash
flag = bytes([x ^ key for x in enc])
```

- **La Lógica**: El cifrado simétrico más simple y efectivo a nivel de bytes es el XOR (`^`).
    
- **Propiedad Matemática**: El XOR es involutivo (su propia inversa). Esto significa que si aplicás XOR a un dato con una llave, se encripta; y si le volvés a aplicar XOR con la **misma** llave al texto cifrado, recuperás el texto original.
    
    $$\text{Si } C = P \oplus K \implies P = C \oplus K$$
    
- **Implementación**: Usamos una _list comprehension_ para iterar por cada byte `x` del archivo cifrado `enc`, aplicarle el XOR con nuestro byte `key`, y finalmente volver a empaquetar toda esa lista de enteros resultantes en un objeto de `bytes` legible.