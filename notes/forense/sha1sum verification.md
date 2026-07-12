### Podemos Usar  Tambien autopsy
#### Verificamos los archivos devuelve el nombre de la carpeta y el contenido segun la ruta
```python
find .git/objects/ -type f | grep -v 'pack\|info'                                                  ─╯
.git/objects/aa/1cc01687b4ec94faf9916c3fc6efd83f23b816
.git/objects/26/b809e0c41d8421f1126ed3a4eb06ad66e6d90a
.git/objects/f1/50f0b963ab3ee95ba5656212abd76d7f2fed2e
.git/objects/5e/b896e3ccd51175f66480cdb247fc45f3e8ac2d
.git/objects/e8/0b38b3322a5ba32ac07076ef5eeb4a59449875
.git/objects/6b/1ebe10826d5c1efc58ae475c0a0af10f580b77
.git/objects/6b/f83de540f7d12cc3b683a83d69432e03d84509
.git/objects/2c/0a9b2b15dce92f800393d5030c7454efc278ae
.git/objects/d4/666b9472fad7cd75d05b641e402347d9aac605
.git/objects/ea/d27e2bd5a0fc22868ffb629a768f82dfcda11c
.git/objects/22/f7d0c9bd045563ae33bfacfbe46fe406a5b318
.git/objects/c9/31ae0868411e5f23656a2436e78a4c4699e18c
.git/objects/20/1c707b43219a63c1d3499b29c7d539af079861
.git/objects/21/51ef0ccc15aed1ab88e1afdc7484aaeff211c4
.git/objects/58/27632e046a80a1e0d7b4fc5c7800dd539baeaf
.git/objects/d7/b4a371ebd23e682ffebc7ec355690fdc94fbd1
.git/objects/a0/c13fe974d95661f24e32bc0d79f54f05ea13c5
.git/objects/66/273877d2ff3f51a14473b7200aae5a798ff64f
.git/objects/71/78644433e7cb6da3adf028f1c80d382a18e7b6
.git/objects/71/fd2fafcd5ebd62fbf857769c92a91225ab3954
.git/objects/01/533f718556a0e59f1467dae4fa462eed82c2a1
```

#### Creamos ese script para mayor rapides
```python

for file in $(find .git/objects/ -type f | grep -v 'pack\|info'); do                               ─╯
    # 1. Limpia la ruta para dejar solo el hash SHA-1 de 40 caracteres
    hash=$(echo $file | sed 's/\.git\/objects\///' | tr -d '/')

    # 2. Descomprime y lee el objeto directamente desde la base de datos
    git cat-file -p $hash 2>/dev/null
done | grep "picoCTF"
print hash
Jay: Ask Rusty at the door and use password picoCTF{g17_r35cu3_16ac6bf3}.
```

#### O con esta herramienta  es lo que usa git para descomprimer
```python
zlib-flate -uncompress < .git/objects/71/fd2fafcd5ebd62fbf857769c92a91225ab3954

zlib-flate -uncompress < .git/objects/71/78644433e7cb6da3adf028f1c80d382a18e7b6
  
R/
blob 188Rex: Meet at the old arcade basement for the secret hideout.
Jay: Ask Rusty at the door and use password picoCTF{g17_r35cu3_16ac6bf3}.
Rex: Bring the decoder map so we can plan the route.

```


#### O script en python
```python
python3 << 'EOF'
import os
import zlib

RUTA_GIT_OBJECTS = ".git/objects"
print(f"[+] Buscando banderas en: {RUTA_GIT_OBJECTS}...\n")

for carpeta_raiz, _, archivos in os.walk(RUTA_GIT_OBJECTS):
    for archivo in archivos:
        if len(archivo) == 38:
            ruta_completa = os.path.join(carpeta_raiz, archivo)
            try:
                with open(ruta_completa, 'rb') as f:
                    datos_comprimidos = f.read()
                datos_descomprimidos = zlib.decompress(datos_comprimidos)
                texto = datos_descomprimidos.decode('utf-8', errors='ignore')
                if "picoCTF" in texto or "flag" in texto:
                    hash_completo = os.path.basename(carpeta_raiz) + archivo
                    print(f"🎯 ¡Bandera encontrada en el objeto {hash_completo}!")
                    print(f"Contenido:\n{texto.strip()}\n")
                    print("-" * 50)
            except Exception:
                pass
EOF
```