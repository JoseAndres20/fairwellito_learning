Primero copia **solamente** el bloque que empieza:

```
UEsDBBQAAQAAAPYTIlvXNpPBSwIAAD8CAA...
```

hasta:

```
...UEsFBgAAAAABAAEAQAAAAHsCAAAAAA==
```

Puedes guardarlo así:

```
nano zip.b64
```

Luego decodifica:

```
base64 -d zip.b64 > expediente_TH-0918.zip
```

Comprueba:

```
file expediente_TH-0918.zip
```

Debería indicar algo como:

```
Zip archive data
```