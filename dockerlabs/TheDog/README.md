# TheDog

**Platform:** DockerLabs  
**OS:** Linux  
**Category:** Web Exploitation (Apache 2.4.49) & Privilege Escalation

🔗 [Ver como web](https://pablogbl.github.io/vulnerable-machines/dockerlabs/TheDog/) · [Ver walkthrough como web](https://pablogbl.github.io/vulnerable-machines/dockerlabs/TheDog/TheDog.html)

---

## Overview

TheDog es una máquina Linux de dificultad media de DockerLabs. El compromiso empieza por un servidor web Apache 2.4.49 vulnerable, que nos da una shell como www-data, y termina en root tras un movimiento lateral vía esteganografía y la extracción de credenciales embebidas en un binario.

---

## Enumeration

El escaneo con nmap revela únicamente el puerto 80 abierto. En el código fuente de la web encontramos una pista que apunta a hacer fuzzing de ficheros, que realizamos con dirb, y confirmamos la versión de Apache como vector de entrada.

---

## Exploitation

Apache 2.4.49 arrastra una vulnerabilidad conocida. Usando el módulo correspondiente de Metasploit contra el puerto 80, obtenemos una sesión de Meterpreter como www-data y estabilizamos una shell interactiva.

---

## Lateral Movement

Enumerando como www-data localizamos un fichero con contenido oculto por esteganografía. Lo exfiltramos a nuestra máquina y, con stegsnow y fuerza bruta sobre rockyou, recuperamos la clave `superman`, que revela la credencial `secret`. Esa contraseña nos permite autenticarnos como el usuario punky.

---

## Privilege Escalation

Como punky encontramos el binario `task_manager`, vulnerable a inyección de comandos aunque sin darnos root directamente. Extrayendo sus cadenas con `strings` recuperamos varias contraseñas embebidas y, probándolas contra `su root` en bucle, damos con la clave de root.

---

## Key Takeaways

- Una versión concreta de un servicio (Apache 2.4.49) puede ser suficiente para lograr ejecución remota: identificar versiones es parte crítica de la enumeración.
- La esteganografía puede ser el puente entre un acceso limitado y un usuario válido; conviene revisar ficheros sospechosos y estar dispuesto a hacer fuerza bruta sobre la clave del stego.
- Cuando el vector obvio (la inyección de comandos del binario) no da root, `strings` sobre ese mismo binario puede revelar credenciales hardcodeadas que sí abren el camino.

---

⚠️ Realizado con fines educativos en un entorno controlado.
