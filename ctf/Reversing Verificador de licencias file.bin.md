---
tags: [CTF, Reversing, Máquina Virtual, XOR, Bytecode, Python]
fecha: 2026-05-23
plataforma: HackRocks
estado: Resuelto
---

# ⚙️ CTF Writeup: Reversing - Verificador sencillo de licencias

## 📝 Resumen del Reto
Se nos proporciona un archivo `bytecode.bin` perteneciente a un verificador de licencias que ejecuta su lógica dentro de una Máquina Virtual (VM) ofuscada personalizada. 

* **Objetivo:** Aplicar ingeniería inversa al bytecode para reconstruir la cadena exacta (la flag) que hace que la VM se detenga en el estado de validación exitosa.
* **Formato conocido:** La cadena debe tener el formato `flag{...}`.

## 🐛 Análisis de la Vulnerabilidad
En lugar de revertir el binario nativo de la VM, analizamos directamente los patrones dentro del `bytecode.bin`. 

Al examinar la estructura de los datos, el bytecode presenta bloques repetitivos de 3 bytes:
1. **Byte 1:** Un carácter cifrado (ej. `3`, `9`, `4`, `2`).
2. **Byte 2:** La constante `U` (Hexadecimal `0x55`).
3. **Byte 3:** Caracteres que descienden en valor, actuando como punteros o índices de memoria de la VM.

### Known Plaintext Attack (KPA)
Dado que sabemos que la solución comienza con `flag{`, podemos deducir la operación matemática. 
Comparamos el primer byte del archivo (`0x33` que es '3') con el primer byte esperado (`0x66` que es 'f'):

`0x33 ^ 0x66 = 0x55`

La operación es un simple cifrado **XOR** ($\oplus$) utilizando la constante `U` (`0x55`) como llave. La VM valida la entrada aplicando un XOR a la contraseña ingresada y comparándola con la primera columna del bytecode.

## 💻 Script de Resolución (Exploit)
Para evitar hacer la conversión a mano, automatizamos el ataque de texto plano conocido leyendo el primer carácter de cada bloque lógico y aplicando la llave `0x55`.

**Código en Python (`solve_vm.py`):**
```python
#!/usr/bin/env python3

def decode_bytecode():
    # Estructura extraída visualmente del volcado del bytecode
    # Se omiten los índices finales y la constante 'U'
    cipher_chars = [
        '3', '9', '4', '2', '.', '#', '8', '\n',
        "'", '0', '#', '0', "'", '&', '0', "'",
        '\n', '8', '4', '&', '!', '0', "'", '('
    ]
    
    key = 0x55  # El valor hexadecimal de la constante 'U'
    flag = ""
    
    print("[*] Iniciando decodificación de la Máquina Virtual...")
    
    for char in cipher_chars:
        # Extraemos el valor numérico del carácter cifrado
        cipher_byte = ord(char)
        # Aplicamos la operación matemática de la VM a la inversa
        decrypted_byte = cipher_byte ^ key
        # Convertimos el número resultante de vuelta a texto
        flag += chr(decrypted_byte)
        
    print(f"[+] ¡Bytecode revertido con éxito!")
    print(f"[+] FLAG: {flag}")

if __name__ == '__main__':
    decode_bytecode()

```

tags: [Metodología, Forense, Binarios, Reversing, Comandos, Cheatsheet]
fecha: 2026-05-23
estado: Referencia
---

# 🛠️ Metodología Forense: Análisis Inicial de Binarios (.bin)

Cuando nos enfrentamos a un archivo binario crudo (`.bin`) sin formato aparente, los editores de texto convencionales fallarán mostrando caracteres corruptos. Se debe aplicar una metodología de análisis descendente (de la superficie a los bytes a bajo nivel) utilizando herramientas de terminal.

A continuación, el flujo de trabajo estándar para encontrar patrones y vulnerabilidades ocultas.

## 1. Identificación de Cabeceras (`file`)
El primer paso es siempre confirmar el tipo de archivo real leyendo sus "magic bytes" (cabeceras mágicas), independientemente de la extensión que tenga el archivo.

```bash
file bytecode.bin
````

- **Objetivo:** Determinar si es un ejecutable estándar (ELF, PE), un archivo comprimido, o simplemente `data` (datos crudos/formato personalizado). Si el resultado es `data`, la única forma de avanzar es mediante análisis hexadecimal.
    

## 2. Inspección Profunda a Nivel de Bytes (`xxd`)

Para leer la estructura real de los datos, utilizamos un visor hexadecimal. `xxd` muestra la memoria del archivo dividida en tres columnas clave: Offsets (posiciones), Hexadecimal (bytes crudos) y ASCII (texto humano).

Bash

```
xxd bytecode.bin | head -n 20
```

- **Explicación del comando:** `head -n 20` limita la salida a las primeras 20 líneas para no inundar la terminal, permitiendo un análisis controlado.
    
- **En la práctica:** Al observar la salida, buscamos patrones visuales: repeticiones de bloques, saltos anómalos o constantes fijas (como descubrir que el byte `0x55` se intercala rítmicamente entre los datos).
    

## 3. Comprobación Matemática Rápida (Known Plaintext Attack)

Si sospechamos de un cifrado lógico (como XOR) y conocemos el formato esperado de la respuesta (ej. sabemos que las flags siempre empiezan con `flag{`), podemos probar nuestra teoría directamente en la terminal usando Python como calculadora, sin necesidad de programar un script completo.

Bash

```
python3 -c "print(hex(0x33 ^ 0x66))"
```

- **Explicación del comando:**
    
    - `-c`: Ejecuta el código contenido entre comillas.
        
    - `0x33`: El primer byte cifrado (extraído de la vista de `xxd`).
        
    - `^`: Operador matemático XOR en Python.
        
    - `0x66`: El byte real esperado (el código hexadecimal para la letra 'f').
        
- **Objetivo:** Si el resultado coincide con la constante que vimos repetida (ej. `0x55`), confirmamos matemáticamente la vulnerabilidad lógica.
    

## 4. Extracción Rápida de Texto (`strings`)

Como paso complementario, le pedimos al sistema que filtre la "basura" binaria y extraiga únicamente las secuencias de caracteres que sean imprimibles y legibles.

Bash

```
strings -n 1 bytecode.bin
```

- **Explicación del comando:** `-n 1` obliga a la herramienta a imprimir cualquier secuencia de al menos 1 carácter (por defecto son 4). Esto es crítico en archivos muy ofuscados donde las letras están separadas por bytes nulos o basura.
    
- **Objetivo:** Obtener una pista visual rápida de variables ocultas, contraseñas _hardcodeadas_ o el alfabeto ofuscado que utiliza el programa.