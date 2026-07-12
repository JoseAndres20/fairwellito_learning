### ¿Qué hace `stepic`?

La esteganografía es el arte y la técnica de ocultar información (como mensajes secretos, contraseñas o archivos) dentro de otros archivos que parecen inofensivos, generalmente imágenes (como archivos PNG, JPG o BMP).

En tu caso, `stepic` realizó lo siguiente:

1. **Detección de datos ocultos (Modo `-d`):** El comando `stepic -d -i upz.png` le indicó a la herramienta que buscara datos ocultos dentro de la imagen `upz.png`.
    
2. **Extracción:** `stepic` analiza los bits menos significativos (LSB - _Least Significant Bits_) de los píxeles de la imagen. Al modificar ligeramente estos bits (de una manera imperceptible para el ojo humano), es posible codificar un mensaje o archivo completo sin alterar la apariencia visual de la imagen.
    
3. **Resultado:** Como puedes ver en tu terminal, la herramienta logró extraer exitosamente la cadena de texto que contenía la bandera (`picoCTF{fl4g_h45_fl4g0e590975}`).


#### Vemos que este reto es web forense debemos analizar los vectores

![[Pasted image 20260712125542.png]]
#### Vemos que uno no es un pais es una imagen normal entonces descargamos la imagen para analizarla
![[Pasted image 20260712125614.png]]
#### Usamos la herramineta Stepic por que las otras no fucionaban 
```bash
stepic -d -i nombredelaimagen
```
![[Pasted image 20260712125301.png]]