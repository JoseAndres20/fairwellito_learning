
```python
Accessories' UNION SELECT @@version, NULL-- -
```

### Explicación del payload paso a paso:
``
- **`Accessories'`**: Cierra la consulta original de la aplicación.
- **`UNION SELECT`**: Une tu consulta maliciosa con la consulta legítima.
- **`@@version`**: Es el campo donde pedimos que se extraiga la versión de MySQL.
- **`, NULL`**: Es el segundo campo necesario para completar las 2 columnas que tiene la tabla original.
- **`-- -`**: Es el estándar para comentar el resto de la consulta SQL y evitar errores de sintaxis al final.
- 
![[Pasted image 20260705125055.png]]
