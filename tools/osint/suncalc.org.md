https://www.suncalc.org/

# Writeup: GEOINT III — "Sale el sol"

**Plataforma:** hackrocks.com **Categoría:** GEOINT | **Dificultad:** Media | **Puntos:** 55 **Formato del token:** `HH:MM:SS`

---

## 1. Planteamiento del reto

El reto se presenta como una transcripción de un interrogatorio. El sospechoso afirma haber visto un cadáver flotando en un lago **a las 7:00 AM del 20 de febrero de 2014**, mientras caminaba por _"la calle que parte el lago a la mitad"_, en un lugar llamado **Browerville Barrow**.

El jefe pide expresamente:

> _"¿puedes confirmar a qué hora empezó a haber luz natural el día 20 de febrero del 2014 en ese punto?"_

Es decir: el objetivo real del reto es **geolocalizar el lugar exacto** a partir de nombres propios sueltos en el diálogo, y luego **calcular la hora del amanecer** en ese punto y esa fecha, para contrastarla con la hora que dio el testigo (7:00 AM) y ver si su versión es creíble.

---

## 2. Paso 1 — Extraer los nombres propios del texto

De toda la conversación, solo hay dos anclas verificables:

|Dato en el texto|Tipo de pista|
|---|---|
|**"Browerville Barrow"**|Nombre de lugar|
|**"la calle que parte el lago a la mitad"**|Descripción física de una vía|
|**20 de febrero de 2014, 7:00 AM**|Fecha/hora a verificar|
|_"lleva años yendo y viniendo desde Nueva York"_|Detalle de contexto (para probar que vive fuera pero visita el sitio)|

El resto del diálogo es relleno narrativo (tono policial), no aporta coordenadas.

---

## 3. Paso 2 — Identificar "Browerville Barrow" (OSINT)

Buscando el nombre literalmente, aparece en Wikipedia:

> _"The city of Utqiaġvik has three sections [...] known to residents as Utqiaġvik, **Browerville**, and NARL [...] The central section is the largest of the three and is called **Browerville**."_

Conclusión: **Browerville** es el barrio central de **Utqiaġvik** (antes **Barrow**), Alaska — la ciudad más al norte de EE. UU., situada por encima del Círculo Polar Ártico.

Esto también explica temáticamente el título del reto, _"Sale el sol"_: en esta latitud el sol desaparece semanas enteras (noche polar) y el amanecer se adelanta muy rápido día a día al volver — un escenario perfecto para un reto de cálculo solar.

---

## 4. Paso 3 — Localizar "la calle que parte el lago"

Wikipedia añade una pista física clave:

> _"Browerville is separated from the south section by a series of lagoons, with two connecting dirt roads."_

Buscando el callejero de Utqiaġvik/Barrow se confirma que la vía que cruza esas lagunas (concretamente la **Isatkoak Lagoon**, entre Browerville y el resto de la ciudad) es **Stevenson Street**, la única calle de tierra que conecta ambos lados atravesando el agua.

Con esto queda geolocalizado el punto exacto desde donde el testigo dice haber visto el cadáver.

---

## 5. Paso 4 — Obtener coordenadas del punto

Con el nombre de la laguna y la calle, se buscan sus coordenadas en fuentes geográficas (Topozone/GNIS, OpenStreetMap):

- **Isatkoak Lagoon / Stevenson Street:** `71.293347° N, 156.755432° O`
- Zona horaria: **AKST (UTC-9)**, sin horario de verano en febrero.

---

## 6. Paso 5 — Calcular la salida del sol (o el inicio de luz natural)

Aquí es donde entra la herramienta recomendada por el reto: **[suncalc.org](https://www.suncalc.org/)**.

Se introducen coordenadas + fecha:

```
https://www.suncalc.org/#/71.293347,-156.755432,15/2014.02.20/09:46/1/3
```

`suncalc.org` usa la librería **SunCalc.js** (Vladimir Agafonkin), que calcula, entre otros datos:

|Evento|Definición|
|---|---|
|`dawn`|Inicio del crepúsculo civil (sol a -6°) → cuándo **empieza a haber luz natural**|
|`sunrise`|El disco solar asoma en el horizonte (sol a -0.833°)|

Como la pregunta exacta del jefe es _"¿a qué hora **empezó a haber luz natural**?"_, el dato relevante no es el orto exacto (`sunrise`), sino el **alba civil** (`dawn`).

Reproduciendo el algoritmo exacto de SunCalc.js para esas coordenadas y fecha (20/feb/2014, AKST):

```
nauticalDawn -> 07:19:43
dawn (civil) -> 08:35:26   ← inicio de luz natural
sunrise      -> 09:46:24   ← salida completa del sol
```

---

## 7. Conclusión del caso

El testigo declaró haber visto el cadáver **claramente, con detalles como la hinchazón de la cara, a las 7:00 AM**. Según el cálculo:

- A las 7:00 AM **ni siquiera había empezado el crepúsculo náutico** (07:19).
- La luz natural (alba civil) no llegó hasta las **08:35:26**.
- El sol no salió del todo hasta las **09:46:24**.

➡️ Su versión no es compatible con las condiciones reales de luz de ese día en ese punto.

---

## Token final

```
08:35:26
```

---

## Nota metodológica

Este writeup documenta el **razonamiento y proceso de resolución** (identificación del lugar vía OSINT + cálculo solar con SunCalc.js). Los intentos de token se fueron refinando en varias iteraciones:

1. Coordenadas del centro de la ciudad + `sunrise` → descartado
2. Coordenadas del centro de la ciudad + `sunrise` recalculado con SunCalc.js → descartado
3. Coordenadas exactas de Stevenson St / laguna Isatkoak + `sunrise` → descartado
4. Coordenadas exactas + `dawn` (alba civil, coincide mejor con la pregunta literal del jefe) → **pendiente de confirmar**

Si este último tampoco es correcto, los siguientes candidatos a probar serían:

- `nauticalDawn` (07:19:43) — por si el reto considera "luz natural" al crepúsculo náutico
- Verificar si el reto espera la hora en **UTC** en vez de hora local AKST
- Verificar si usan otras coordenadas ligeramente distintas para "la calle" (p. ej. otro tramo de Stevenson Street, más al norte, cerca de NARL)