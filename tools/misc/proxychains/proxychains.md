# Found Code Country

```bash
#Buscar por Code ejemplo is = Islandia 
https://gist.github.com/0x07dc/c2b36774aa5bc8107193c900008bfeb1
```

# Comando para editar el pais 

```bash
#Al final de todo editarlo
nano /etc/tor/torrc   
```

![[Pasted image 20260703110038.png]]


# Ecender lo que es el tor para el proxy
```bash
#status o stop
sudo systemctl start  tor 
```


# Comando para pedir la peticion con el cambio Geografico

```bash
proxychains curl  http://lonely-island.picoctf.net:64319/ 
proxychains curl  http://lonely-island.picoctf.net:64319/ | grep "pico"
```

![[Pasted image 20260703110538.png]]