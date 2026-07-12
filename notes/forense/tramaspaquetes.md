```python
strings rogue_tower.pcap                                                              
1#yCARRIER: Verizon PLMN=310410 CELLID=16274
CARRIER: AT&T PLMN=310410 CELLID=16275
device-310410347132170
network
GET /api/register HTTP/1.1
Host: network.carrier.com
User-Agent: MobileDevice/1.0 (IMSI:310410347132170; CELL:16274)
Accept: */*
Connection: close
device-310410009931734
network
GET /api/register HTTP/1.1
Host: network.carrier.com
User-Agent: MobileDevice/1.0 (IMSI:310410009931734; CELL:16274)
Accept: */*
Connection: close
device-310410213084421
network
GET /api/register HTTP/1.1
Host: network.carrier.com
User-Agent: MobileDevice/1.0 (IMSI:310410213084421; CELL:16274)
Accept: */*
Connection: close
device-310410809129581
network
GET /api/register HTTP/1.1
Host: network.carrier.com
User-Agent: MobileDevice/1.0 (IMSI:310410809129581; CELL:16274)
Accept: */*
Connection: close
example
example
updates
apple
UNAUTHORIZED-TEST-NETWORK PLMN=00101 CELLID=92771 
device-310410728284734
network
GET /api/register HTTP/1.1
Host: network.carrier.com
User-Agent: MobileDevice/1.0 (IMSI:310410728284734; CELL:92771)
Accept: */*
Connection: close
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 9
QlFRV3djd
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 9
U9ACFVNB2
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 9
hQB15UbUw
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 9
EQABGbVkF
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 9
CwUHUVEBR
POST /upload HTTP/1.1
Host: 198.51.100.205
Content-Type: application/octet-stream
Content-Length: 3
maps
google
maps
google

```

Revisamos las peticiones las juntamos 
```python
QlFRV3djdU9ACFVNB2hQB15UbUwEQABGbVkFCwUHUVEBRQ==
```

Le aplicamos base64 y Xor pero debemos conseguir la key que en una trama de udp encontramos la auotrizacion fuimos  probando hasta llegar a este 28284734 enotnces aplicamos xor con esa key y formato uft8

```python
picoCTF{r0gu3_c3ll_t0w3r_a7310be3}
```