
# 🛰️ Cosmic Explorer (HTB)

## Qué era el reto

Una web con dos botones:

- "Scan Cosmic Anomaly" → funciona normal.
- "Access Secure Database" → decía "Access denied" (ahí está la flag).

Por detrás hay **dos servicios**:

- **Go** (puerto 8080): el que recibe la petición del usuario, decide si te deja pasar.
- **Python** (puerto 8081): el que tiene la flag de verdad, solo habla con el de Go.

## El bug

Cuando mandás `{"action": "getcosmic"}`, Go **reenvía tu mismo JSON tal cual** al servicio Python, sin cambiarlo.

El problema es que **Go y Python no leen el JSON igual**:

- **Go** no distingue mayúsculas de minúsculas en la clave `action`, y si hay dos claves parecidas, se queda con la última.
- **Python** sí distingue mayúsculas, y solo mira la clave `action` exacta (en minúscula).

Entonces se puede mandar un JSON con **dos claves "action"** distintas, para que cada uno lea una cosa diferente:

```json
{"action": "getSecureCode", "Action": "getcosmic"}
```

- **Go** ve la última clave que le calza (`Action` → `getcosmic`) → piensa "ok, es la acción permitida" → reenvía el JSON completo a Python.
- **Python** recibe ese mismo JSON, pero él lee la clave exacta `action` (minúscula) → `getSecureCode` → ¡te devuelve la flag!

Osea: **engañamos al filtro de Go haciéndole creer una cosa, mientras Python termina ejecutando otra.**

## El curl

```bash
curl -s -X POST http://154.57.164.77:31989/execute \
  -H "Content-Type: application/json" \
  -d '{"action": "getSecureCode", "Action": "getcosmic"}'
```

Respuesta:

```json
{"flag": "HTB{...}", "name": "Captain's Log", ...}
```

![[Pasted image 20260815120835.png]]

## Idea clave (para recordar)

Cuando dos servicios distintos parsean el mismo JSON por separado, pueden no estar de acuerdo en qué dice ese JSON (por mayúsculas, claves duplicadas, etc). Si uno usa ese parseo para "decidir si te deja pasar" y el otro para "ejecutar la acción real", ahí se puede colar una inconsistencia.