# Road to Olympus

**Platform:** DockerLabs  
**OS:** Linux  
**Category:** Network Pivoting & Chained Exploitation

---

## Overview

Laboratorio de tres máquinas encadenadas —**Hades**, **Poseidón** y **Zeus**— en redes segmentadas. Cada máquina debe comprometerse para pivotar hacia la siguiente mediante túneles con chisel, socat y proxychains, hasta tomar la última.

---

## Infrastructure

Al desplegar el laboratorio aparecen tres contenedores en redes distintas (10.10.10.2, 20.20.20.3 y 30.30.30.3). Solo la primera es accesible directamente; las otras dos requieren pivotar a través de la máquina anterior.

---

## Hades — Web Recon & SSH

Escaneo con nmap: SSH moderno (descartado) y puerto 80 con Werkzeug/Flask. En el código fuente de la web hay un comentario con una contraseña codificada para el usuario `cerbero`; al decodificarla se obtiene **P0seidón2022!** y se accede por SSH.

---

## Pivoting to Poseidon

Con acceso a Hades, se transfieren chisel y socat por scp, se levanta un chisel server en Kali (puerto 1234) y se configura `/etc/proxychains.conf` (`dynamic_chain` + `socks5 127.0.0.1 1080`) para alcanzar la red de Poseidón.

---

## Poseidon — SQL Injection & Privilege Escalation

Escaneo por proxychains: puertos 22 y 80. El buscador web envía por POST a `database.php`. sqlmap no funciona, pero se identifica **SQLite** y se abusa de `sqlite_master` (`select name from sqlite_master`) para enumerar tablas y volcar usuarios y contraseñas codificadas. Se decodifican y las credenciales de **megalodon → Templ02019!** dan acceso por SSH. Un `sudoers` permisivo permite escalar a root con `sudo bash`.

---

## Pivoting to Zeus

Segundo salto: se pasa chisel a Poseidón, socat reenvía el puerto 1111 hacia el chisel server, se lanza chisel en Poseidón y se reconfigura proxychains. Un `curl` de comprobación devuelve HTTP 200 contra Zeus.

---

## Zeus — SMB Enumeration, FTP Brute Force & SSH

Zeus expone 21, 22, 80, 139 y 445 (SAMBA). Con **enum4linux** se enumeran los usuarios `rayito` y `hercules`. Un ataque de fuerza bruta con Hydra contra FTP (rockyou reducido a 5000 líneas) recupera la contraseña de `hercules`. Dentro del FTP hay un archivo cuyo contenido, decodificado, revela la contraseña de `rayito`, con la que se logra el acceso final por SSH.

---

## Key Takeaways

- Nunca dejar secretos (contraseñas, pistas) en comentarios del código fuente.
- No almacenar contraseñas en formatos fácilmente reversibles; usar hashing robusto y con sal real.
- Evitar reglas de `sudo` que permitan ejecutar cualquier binario: equivalen a dar root.
- El pivoting con chisel + socat + proxychains es la clave para alcanzar redes que no se ven desde el exterior.
- Enumerar bien cada servicio (web, SMB, FTP) abre el siguiente eslabón de la cadena.

---

⚠️ Realizado con fines educativos en un entorno controlado.