### Analizamos el file con wires

>Encontramos que dice flag y es un file .zip lo descargamos seleccionamos el paquete  luego fule export http y se descargar el flag.zip
>
![[Pasted image 20260713160955.png]]


### Vemos que nos pide contrasena lo ideal seria buscar una en las tramas o fuerza bruta 

>En este caso usaremos fuerza bruta con #fcrackzip

```python
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u flag.zip
```
![[Pasted image 20260713161808.png]]
###### Encontramos la contrasena  es t `toloikhapao`
![[Pasted image 20260713161259.png]]
#### Nos da  un file .txt

>Aqui vemos que ya casi la tenemos devemos decifrar esa cadena


![[Pasted image 20260713162028.png]]

#### Identificamos esa cadena con esta web


```python
https://www.dcode.fr/cipher-identifier
```

>Vemos que pueden ser varios etonces vamos dandole clicl y verificamos

![[Pasted image 20260713162515.png]]

>Logramos encontrar que era el rot47 y nos da la flag

![[Pasted image 20260713162652.png]]