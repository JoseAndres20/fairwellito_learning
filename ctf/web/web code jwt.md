
# ScreamShot CTF - Writeup y Solución Paso a Paso

El reto "ScreamShot" presentaba un panel de administración vulnerable con la funcionalidad de tomar "capturas de pantalla" de una URL. A continuación se detalla paso a paso cómo se encadenaron las vulnerabilidades para obtener la flag.

## 1. Reconocimiento e Identificación (Fase Inicial)

A partir del código fuente que nos facilitaron (`feature-panel.py`) y las tres pistas, extrajimos la siguiente información clave:
*   **Pista 1:** Sugería buscar archivos de configuración básicos e infraestructura como `/.git`.
*   **Pista 2:** Indicaba que la librería `imgkit` podía ser abusada.
*   **Pista 3:** Sugería que la carpeta `/static/` tenía permisos de escritura y lectura, y recomendaba usarla para guardar nuestros propios *outputs*.

El código mostraba que el panel `/admin` recibía peticiones POST con los parámetros `url`, `output` y `wkhtmltoimage`.

```python
# Fragmento del código original
@app.route("/admin", methods=['POST'])
def checkadminapi():
    data = request.form
    uri, output = data.get('url'), data.get('output')
    if len(data.get('wkhtmltoimage')) > 2:
        param = data.get('wkhtmltoimage')
        try:
            config = imgkit.config(wkhtmltoimage=param)
            imgkit.from_url(uri, str('./static/'+output), config=config)
```

Sin embargo, al intentar enviar una petición POST a `/admin`, el servidor nos respondía con un error `401 Unauthorized` indicando: `<script>alert('you are not admin'); location.replace('./home')</script>`.

Necesitábamos ser administradores.

## 2. Acceso Administrativo (Falsificación de JWT)

### 2.1 Descubrimiento del Archivo `.env`
Haciendo caso a la **Pista 1**, comprobamos si existían archivos ocultos expuestos públicamente en el servidor. Al visitar `https://challenges.hackrocks.com/screamshot/.env`, el archivo fue descargado exitosamente y reveló la clave secreta de la aplicación:

```env
JWT_KEY = S3crEt_t0K3n$$$$
```

### 2.2 Creación de una Cookie JWT Falsa
Sabiendo que la aplicación utilizaba tokens JWT (por los imports en el código de Python: `import jwt`), empleamos la clave `S3crEt_t0K3n$$$$` para fabricar nuestro propio token de sesión como si fuésemos el usuario administrador. 

Creamos el token base64 y firmamos la cabecera usando una pequeña rutina en Python y configuramos el campo `public_id` a `admin`:

```json
// Payload del JWT inyectado
{
  "public_id": "admin",
  "exp": 1779582318
}
```

Al incluir este JWT forjado en la cookie `x-access-token`, obtuvimos acceso legítimo para enviar payloads al endpoint `/admin`.

## 3. Abuso de `imgkit` y RCE (Obtención de la Flag)

Una vez resuelto el control de acceso, nos concentramos en la función vulnerable: `imgkit.from_url`.

### 3.1 Cómo funciona la vulnerabilidad
El campo del formulario web `config path` (que en el código es la variable `param` y se mapea a `wkhtmltoimage`) es utilizado por `imgkit` para determinar **qué ejecutable usar** para tomar la captura.

Internamente, `imgkit` hace uso de `subprocess.Popen` e inserta nuestro comando como el primer elemento de una lista, seguido de la `url` y terminando con el destino `output`.

Si le inyectamos un binario arbitrario en lugar de un path válido de wkhtmltopdf, obligamos a Python a ejecutar dicho comando.

### 3.2 El Payload Perfecto
La **Pista 3** nos sugería utilizar la carpeta `/static/` ya que lo que se escribe ahí es expuesto públicamente. 

Sabiendo que el comando final generado por Python sería conceptualmente:
`[ejecutable_inyectado] [url] ./static/[output]`

En lugar de intentar inyectar Bash y rompernos la cabeza con la sintaxis y los prefijos de carpeta, inyectamos el comando de copia de Linux **`/bin/cp`**.

Nuestra petición maliciosa quedó configurada así:
*   **wkhtmltoimage (Ejecutable):** `/bin/cp`
*   **url (Origen):** `flag.txt` *(Suponiendo que el archivo estaba en la carpeta principal del servidor)*
*   **output (Destino):** `flag.txt` *(Que internamente el código transforma en `./static/flag.txt`)*

El servidor ejecutó:
`/bin/cp flag.txt ./static/flag.txt`

### 3.3 Lectura de la Flag
El comando `cp` copió la flag que estaba escondida en el sistema local a la carpeta pública. Finalmente, solo tuvimos que abrir nuestro navegador y visitar la ruta de la imagen estática:

`https://challenges.hackrocks.com/screamshot/static/flag.txt`

La cual nos devolvió el texto en plano:
`flag{pwn_jwt_4nd_blindly_pwn_the_ISSu3_81}`
