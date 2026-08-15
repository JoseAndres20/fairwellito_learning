### Reto basico de Web comand
![[Pasted image 20260815112227.png]]

### Revisamos la tools aver que trae la web, vemos diferentes js revisamos

- Encontramos un enpoint /api/monitor  que se envia con un secret no sabemos el secret hay que buscarlo
![[Pasted image 20260815112311.png]]


### Encontramos que hace una peticion al options sobre la aplicacion.

- Vemos que encontramos el secret
![[Pasted image 20260815112508.png]]

```java
//secret
Blip-blop, in a pickle with a hiccup! Shmiggity-shmack
```

### Despues de tener el enpoint y el secret hacemos un request al enpoint con el secret

```bash
curl -s -X POST http://154.57.164.82:32265/api/monitor   -H "Content-Type: application/json"   -d '{"command": "
"}'

//Respuesta

{
  "message": "HTB{D3v3l0p3r_t00l5_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}"
}




```


