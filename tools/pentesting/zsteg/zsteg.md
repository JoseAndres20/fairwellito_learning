⚠️ **Importante:** `zsteg` solo funciona bien con **PNG y BMP**

### ¿Qué hace exactamente?

Una imagen está hecha de píxeles, y cada píxel tiene valores de color (RGB). La esteganografía aprovecha que **el bit menos significativo (LSB)** de cada valor de color casi no afecta visualmente el color — entonces se puede usar para "esconder" datos (texto, archivos, otra imagen) sin que se note a simple vista.

`zsteg` revisa automáticamente muchas combinaciones posibles de cómo se pudo esconder esa información (distintos bits, distintos canales de color, distintos órdenes) y te dice si encuentra algo que parezca texto o datos válidos.

### Uso básico
```java
zsteg imagen.png
```

![[Pasted image 20260625150648.png]]