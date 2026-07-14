Attention, **H4ckN3t** team!

Thanks to the latest leak we got in Gorgan, we have managed to pull our strings, learning that Epsilon makes its own reflections and keeps hidden messages in them.

We got hold of a couple of files, a **Text** and an **Image**. The text it contains looks like a philosophical reflection, but I am sure there is something crucial hidden in it. There may be a key inside the text that will allow us to see what is hidden in plain sight, the only thing we know is that the length of the key is 9 characters.

It's time to find out what Epsilon has tried to hide!

Your mission is to carefully analyze the content, piece together the keywords and uncover the secrets that Epsilon has tried to keep hidden. Each find brings us closer to victory!

>Nos dan  una imagen y un txt

![[Pasted image 20260714165636.png]]
![[Pasted image 20260714165738.png]]

### Vemos un Patron en las iniciales de cada reglon

```python
GOLDSTEIN
```

### Usamos la herrmina
```python
 steghide extract -sf Quote.jpg
 
#R/
https://drive.google.com/file/d/1Gaf4lVQkPd_tmZ2zz9xq0Ybu4lccbEU1/view?usp=sharing

```

![[Pasted image 20260714170150.png]]
#### Nos dan otra imagen la subimos en esta web

```python
https://www.aperisolve.com/
```

#### Encontramos la flag
![[Pasted image 20260714170306.png]]