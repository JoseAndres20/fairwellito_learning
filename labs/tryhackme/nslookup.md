![[Pasted image 20260527151252.png]]

>Importante es apra visualisar dns  dominios con el -type=  es para ver  diferentes file

Syntax:

- `nslookup DOMAIN_NAME` performs a simple lookup using your default resolver.
- `nslookup -type=TYPE DOMAIN_NAME [SERVER]` specifies a record type and an optional DNS server.

| Query type | Result                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------- |
| A          | IPv4 address(es) for the domain                                                           |
| AAAA       | IPv6 address(es) for the domain                                                           |
| CNAME      | Canonical Name: an alias that points one domain name to another                           |
| MX         | Mail Servers: the servers responsible for handling email for the domain                   |
| SOA        | Start of Authority: the primary name server, admin email, and zone serial number          |
| TXT        | Text Records: arbitrary text, commonly used for SPF, DKIM, DMARC, and domain verification |


```bash
nslookup -type=txt  thmlabs.com

//answer
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
*** Can't find thmlabs.com: No answer

Authoritative answers can be found from:
thmlabs.com
	origin = kip.ns.cloudflare.com
	mail addr = dns.cloudflare.com
	serial = 2404001052
	refresh = 10000
	retry = 2400
	expire = 604800
	minimum = 1800
```
```bash

nslookup -type=MX  thmlabs.com

//answer
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
thmlabs.com	text = "THM{a5b83929888ed36acb0272971e438d78}"

Authoritative answers can be found from:


```


Other command is dig   domain  type

![[Pasted image 20260527151833.png]]



|Purpose|Command-line Example|
|---|---|
|Lookup WHOIS record|`whois tryhackme.com`|
|Lookup DNS A records (legacy)|`nslookup -type=A tryhackme.com`|
|Lookup DNS MX records at specific server (legacy)|`nslookup -type=MX tryhackme.com 1.1.1.1`|
|Lookup DNS TXT records (legacy)|`nslookup -type=TXT tryhackme.com`|
|Lookup DNS A records (recommended)|`dig tryhackme.com A`|
|Lookup DNS MX records at specific server (recommended)|`dig @1.1.1.1 tryhackme.com MX`|
|Lookup DNS TXT records (recommended)|`dig tryhackme.com TXT`|
|Passive subdomain discovery (browser-based)|Visit https://crt.sh and search `%.tryhackme.com`|