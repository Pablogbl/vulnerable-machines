# BreakMySSH

En este writeup comprometemos la máquina BreakMySSH atacando exclusivamente su servicio SSH. El recorrido parte de un escaneo de puertos, sigue con la enumeración de usuarios válidos mediante un módulo de Metasploit y termina recuperando la contraseña de `root` por fuerza bruta con Hydra para entrar por SSH.

El objetivo no es solo llegar a root, sino entender cada paso: por qué enumerar usuarios nos da un objetivo concreto y por qué una contraseña débil convierte esa lista de usuarios en acceso total a la máquina.

---

## Enumeración de puertos

Como siempre, lo primero es lanzar un escaneo de puertos de la máquina para ver qué servicios expone. El escaneo muestra el puerto 22 (SSH) abierto.

![](<imagenes/Pasted image 20260704173218.png>)

Con SSH como punto de entrada, el trabajo se centra en él: primero averiguar qué usuarios son válidos y después intentar autenticarnos.

---

## Enumeración de usuarios SSH con Metasploit

Abrimos Metasploit y buscamos módulos de enumeración de SSH.

![](<imagenes/Pasted image 20260704174222.png>)

Aparecen varios; el que nos interesa es la opción 4. La seleccionamos y configuramos el módulo, incluyendo el parámetro `user_file` con la lista de usuarios que se van a probar.

![](<imagenes/Pasted image 20260704174945.png>)

Con el módulo configurado, lanzamos la ejecución con `run`.

![](<imagenes/Pasted image 20260704175011.png>)

---

## Identificación de root y fuerza bruta con Hydra

La ejecución devuelve un buen número de usuarios válidos. Entre ellos aparece `root`, que es el objetivo más interesante, así que lo atacamos con Hydra para intentar averiguar su contraseña.

![](<imagenes/Pasted image 20260704175404.png>)

Hydra consigue recuperar la contraseña de `root` (visible en la captura).

---

## Acceso por SSH como root

Con la contraseña obtenida, nos conectamos por SSH directamente como `root`.

![](<imagenes/Pasted image 20260704175702.png>)

Ya estamos dentro como `root`, con lo que la máquina queda comprometida por completo y damos el ejercicio por finalizado.

---

## Conclusión

BreakMySSH se compromete atacando únicamente el servicio SSH: se enumeran usuarios válidos con un módulo de Metasploit, se identifica la cuenta `root` y se recupera su contraseña por fuerza bruta con Hydra, lo que da acceso directo como root sin necesidad de escalada posterior.

El ejercicio refuerza un par de ideas: un servicio SSH puede permitir enumerar cuentas válidas, y una contraseña débil en `root` convierte esa enumeración en un compromiso total. Como mitigación, conviene deshabilitar el login directo de `root` por SSH, usar autenticación por clave en lugar de contraseña y forzar contraseñas robustas.

⚠️ Realizado con fines educativos y en un entorno controlado.
