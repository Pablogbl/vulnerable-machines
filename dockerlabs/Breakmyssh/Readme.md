# BreakMySSH

**OS:** Linux  
**Category:** SSH Enumeration & Brute Force

---

## Overview

Máquina cuyo único vector de ataque es el servicio SSH. Tras identificar el puerto 22 abierto, se enumeran usuarios válidos con un módulo de Metasploit, se detecta la cuenta `root` y se recupera su contraseña por fuerza bruta con Hydra, obteniendo acceso directo como root por SSH.

---

## Enumeration

Un escaneo de puertos revela únicamente el 22 (SSH) abierto, lo que concentra todo el ataque sobre ese servicio.

---

## SSH User Enumeration

Con Metasploit se busca y selecciona un módulo de enumeración de usuarios SSH (opción 4 del listado), configurado con un `user_file`. Al ejecutarlo devuelve un conjunto de usuarios válidos, entre ellos `root`.

---

## Exploitation

Identificada la cuenta `root`, se lanza Hydra contra el servicio SSH para atacar su contraseña. Hydra la recupera y con esas credenciales se accede por SSH directamente como `root`, comprometiendo la máquina por completo sin necesidad de escalada posterior.

---

## Key Takeaways

- Un servicio SSH expuesto puede permitir enumerar cuentas de usuario válidas, dando al atacante objetivos concretos.
- Una contraseña débil en `root` convierte esa enumeración en un compromiso total de la máquina.
- Deshabilitar el login directo de `root`, usar autenticación por clave y forzar contraseñas robustas corta esta cadena de raíz.

---

⚠️ Realizado con fines educativos en un entorno controlado.