### Observamos los pasos de como funciona el systema, vemos que hace peticion a la bd con la session posible sql inyection por medio del session
![[Pasted image 20260923102500.png]]


### Vemos otra cosa importante ya haiy un usuario llamado marisol  con eso confirmamos que si o si puede ser un inyection por medio de la session.
![[Pasted image 20260923102901.png]]


### Buscamos todo tipo de sql inyection vimos que era este 
```sql
//UNION Para unir consultas 
//-- para comentar lo que sigue

"' UNION SELECT id, usuario FROM personal WHERE usuario='marisol' -- "
```

### ELiminamos el id de cookies y agregamos el sql inyection  nos mostrara la flag
![[Pasted image 20260923104935.png]]