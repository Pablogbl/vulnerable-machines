# injection

**Platform:** Docker  
**OS:** Linux  
**Category:** Web Exploitation (SQL Injection)

---

## Overview

Máquina Linux desplegada en Docker que expone SSH y un servicio web. El login de la web era vulnerable a inyección SQL, lo que permitió saltarse la autenticación y recuperar unas credenciales válidas. Esas mismas credenciales servían para entrar por SSH, y desde ahí un binario `env` mal configurado permitió escalar a root siguiendo la técnica de GTFOBins.

---

## Enumeration

Al arrancar el contenedor obtuvimos la IP de la máquina y lanzamos un escaneo de puertos que reveló el 22 (SSH) y el 80 (HTTP) abiertos. Esto define la superficie de ataque inicial: un servicio web para atacar y acceso remoto por SSH sobre el que pivotar más tarde.

---

## SQL Injection

Abriendo el puerto 80 en el navegador aparecía una pantalla de login. Probamos una inyección basada en `UNION`, que devolvió un error por no cuadrar el número de columnas. Con esa pista ajustamos el payload a un `or 1=1` en el campo de usuario, logrando un bypass de autenticación que además mostraba unas credenciales por pantalla.

---

## Exploitation

Las credenciales obtenidas por la web (**Dylan / KJSDFG789FGSDF78**) se reutilizaron contra el servicio SSH del puerto 22 con éxito, lo que nos dio una shell en la máquina. La reutilización de la misma credencial entre la web y SSH fue la que permitió convertir el fallo web en acceso al sistema.

---

## Privilege Escalation

Ya dentro por SSH, la enumeración de permisos, grupos y binarios reveló el binario `env` como vía de escalada. Consultando su ficha en GTFOBins ejecutamos el comando indicado, que devolvió una shell con privilegios de root y el compromiso total de la máquina.

---

## Key Takeaways

- Una inyección SQL en un login puede pasar de un simple bypass de autenticación a la exposición directa de credenciales.
- Reutilizar la misma credencial entre servicios (web y SSH) convierte un fallo aislado en acceso al sistema.
- Un binario como `env` con SUID o permitido en `sudoers` es suficiente para escalar a root vía GTFOBins.

---

⚠️ Realizado con fines educativos en un entorno controlado.