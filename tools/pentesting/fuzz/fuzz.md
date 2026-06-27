### Comandos

```bash 
ffuf -u "https://ctf25sac672da51.blob.core.windows.net/FUZZ/flag.txt" -w /usr/share/wordlists/dirb/common.txt -mc all 

ffuf -u "http://saturn.picoctf.net:59246/secret/hidden/FUZZ/superhidden/xdfgwd.html
" -w /usr/share/wordlists/dirb/common.txt -rate 100 -mc all


ffuf -u http://192.168.56.13/FUZZ -w /usr/share/wordlists/dirb/common.txt -fs 4242

```