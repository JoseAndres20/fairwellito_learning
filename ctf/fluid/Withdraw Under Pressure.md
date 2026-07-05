![[Pasted image 20260704113833.png]]
# Writeup — Withdraw Under Pressure (Meridian Trust)

**Categoría:** Web · **Dificultad:** Hard · **Puntos:** 295 **Flag obtenida:** `flag{1b8194e06ee55233}`

## Resumen

Meridian Trust es una banca digital de juguete que le regala **$100.00 de bono de bienvenida** a cada cuenta nueva, y permite transferir fondos entre cuentas de forma instantánea. El objetivo era conseguir **$500.00** para comprar "Premium Account Access" y obtener el flag (un "membership certificate").

La vulnerabilidad de negocio es simple una vez que se ve: **no hay límite de cuántas cuentas puede crear un mismo atacante**, y cada cuenta nueva trae su propio bono de $100. Entonces:

> Bono de bienvenida repetible + transferencias sin restricción de origen múltiple = acumulación arbitraria de saldo.

## Cadena de explotación

1. **Registrar varias cuentas nuevas.** Cada una arranca con $100.00 de bono (visto en `/dashboard` → `AVAILABLE BALANCE: $100.00`).
2. **Elegir una cuenta "principal"** (en este caso, el usuario `123`) como destino final.
3. **Desde cada una de las otras cuentas, transferir su balance completo ($100) hacia la cuenta principal**, usando el formulario `Send funds` (`POST /transfer` con `to_user` y `amount`).
4. Repetir hasta juntar **$500.00** en la cuenta principal (bastan 4 cuentas adicionales transfiriendo $100 c/u, sumadas al $100 propio = $500).
5. Ir a **Upgrades** y comprar "Premium Account Access" ($500.00) → la tarjeta pasa a estado **Owned** y muestra directamente el flag:
    
    ```
    flag{1b8194e06ee55233}
    ```
    

## Por qué funciona (root cause)

- El sistema de bono de bienvenida no está atado a ninguna verificación de identidad real (email único válido, KYC, límite por IP/dispositivo, etc.) — cualquiera puede registrar cuentas ilimitadas y cada una "imprime" $100 de saldo.
- El endpoint `/transfer` valida que el `to_user` exista y que no sea la propia cuenta (`"Cannot transfer to yourself"`), pero **no valida cuántas cuentas distintas del mismo atacante están alimentando a una sola cuenta destino**. No hay detección de patrón de "fan-in" (muchas cuentas nuevas transfiriendo a una sola en poco tiempo), que sería la señal típica de abuso de bonos/promociones en banca real.
- En productos financieros reales esto se llama **bonus abuse / multi-accounting fraud**, y normalmente se mitiga con:
    - Un bono por persona/dispositivo/documento verificado, no por cuenta.
    - Límites de velocidad y montos en transferencias entrantes desde cuentas "nuevas" (< X días de antigüedad).
    - Reglas de detección de fraude que correlacionen múltiples cuentas nuevas enviando dinero a un mismo destino en una ventana corta de tiempo.
    - KYC real (verificación de identidad) antes de habilitar transferencias.

## Cosas que se probaron antes y no eran el camino

Antes de dar con esto se exploraron (y descartaron) otras hipótesis, documentadas acá para referencia:

- **Cookie de sesión (`session=...`) no es un JWT**, es el formato firmado de Flask (`itsdangerous`): `payload.timestamp.firma`, codificado en base64 y firmado con la `SECRET_KEY` de la app. Se intentó romperla con `flask-unsign` (brute-force de diccionario) sin éxito con la wordlist default — no era el camino esperado, al menos no con un diccionario común.
- **Montos negativos / manipulación directa del monto vía curl** (bypaseando el `min="0.01"` del HTML, que es solo validación de cliente) — no se llegó a confirmar si el server lo bloqueaba, quedó descartado al encontrar el camino de multi-cuenta.
- **Race condition en `/transfer`** (dado el nombre "Withdraw Under Pressure", sugiriendo TOCTOU) — hipótesis razonable dado el título del reto, pero la solución real terminó siendo puramente lógica de negocio (multi-accounting), no una condición de carrera de concurrencia.
- El servidor sí valida que el `to_user` exista (`"Recipient not found"`) y bloquea auto-transferencia (`"Cannot transfer to yourself"`), así que hubo que usar cuentas _distintas_ como origen.

## Timeline resumido

```
1. Crear cuenta A (bono $100) — cuenta principal, ej. "123"
2. Crear cuenta B (bono $100) -> transferir $100 a A
3. Crear cuenta C (bono $100) -> transferir $100 a A
4. Crear cuenta D (bono $100) -> transferir $100 a A
5. Crear cuenta E (bono $100) -> transferir $100 a A
   (A ahora tiene $100 propio + 4x$100 = $500)
6. Ir a /upgrades -> comprar "Premium Account Access" ($500)
7. Tarjeta pasa a "Owned" -> flag{1b8194e06ee55233}
```

## Lección general

En retos y sistemas reales de "wallet/banca", cuando se ve:

- un bono de bienvenida fijo por cuenta, y
- transferencias libres entre cuentas propias del mismo sistema,

la primera hipótesis a probar (antes de ir a cripto o concurrencia) debería ser **multi-accounting**: crear N cuentas y consolidar sus bonos en una sola. Es low-tech, no requiere ninguna herramienta especial, y muchas veces es exactamente el diseño (a propósito) del reto para forzar a pensar en abuso de lógica de negocio en vez de vulnerabilidades técnicas clásicas.