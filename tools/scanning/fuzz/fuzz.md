### Comandos

```bash 
ffuf -u "https://welcome-aboard-d27b28c93ad0aa4b-global.challs.brunnerne.xyz:1337/FUZZ" -w /usr/share/wordlists/dirb/common.txt -mc all 

ffuf -u "http://saturn.picoctf.net:59246/secret/hidden/FUZZ/superhidden/xdfgwd.html
" -w /usr/share/wordlists/dirb/common.txt -rate 100 -mc all


ffuf -u http://192.168.56.13/FUZZ -w /usr/share/wordlists/dirb/common.txt -fs 4242

```