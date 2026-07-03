
En este writeup comprometemos una máquina Linux desplegada en Docker, partiendo de una enumeración de puertos hasta conseguir una shell de root. El recorrido pasa por un login web vulnerable a inyección SQL, la reutilización de las credenciales obtenidas contra SSH y una escalada de privilegios a través del binario `env`.

El objetivo no es solo llegar a root, sino entender cada paso: por qué la inyección SQL nos deja saltar el login, por qué la misma credencial nos abre una segunda puerta y por qué un binario mal configurado basta para tomar el control completo del sistema.

---

## Despliegue de la máquina

Al levantar el contenedor obtenemos la IP de la máquina, que será nuestro objetivo durante todo el ejercicio.

![[Pasted image 20260703185411.png]]

---

## Enumeración de puertos

Con la IP identificada, lanzamos un escaneo de puertos para descubrir qué servicios expone la máquina. El escaneo revela dos puertos abiertos: el 22 (SSH) y el 80 (HTTP).

![[Pasted image 20260703185926.png]]

Esto nos deja dos frentes: un servicio web sobre el que empezar a trabajar y un acceso remoto por SSH que probablemente aprovechemos más adelante.

---

## Identificación del login web

Abrimos el puerto 80 en el navegador y nos encontramos con una pantalla de login, que se convierte en el primer punto de entrada a atacar.

![[Pasted image 20260703190139.png]]

---

## Inyección SQL y bypass de autenticación

Empezamos probando una inyección SQL basada en `UNION` para comprobar si el login construye la consulta concatenando nuestra entrada:

```sql
'UNION SELECT NULL,NULL,NULL -- -
```

La inyección devuelve un error: el número de columnas indicado no coincide con el de la consulta original.

![[Pasted image 20260703191745.png]]

Revisando el código de la página encontramos la pista sobre los campos que intervienen en la consulta, lo que nos confirma que la entrada llega sin sanear a la base de datos.

![[Pasted image 20260703191900.png]]

Con esa información cambiamos de enfoque: en lugar de un `UNION`, introducimos en el **campo de usuario** un `or 1=1` que hace la condición siempre verdadera, dejando la contraseña comentada (el valor de contraseña puede ser cualquiera):

```sql
' or 1=1-- -
```

De forma que la consulta queda, según lo observado, así:

```sql
SELECT ... WHERE name = 'or 1=1-- -' AND passwd = '' AND password = '...'
```

El resultado es un bypass de autenticación: no solo entramos sin credenciales válidas, sino que la web nos muestra directamente un usuario y su contraseña.

![[Pasted image 20260703192114.png]]

Las credenciales obtenidas son **Dylan / KJSDFG789FGSDF78**.

---

## Acceso por SSH

Como la máquina tenía el puerto 22 abierto, probamos las credenciales de Dylan contra el servicio SSH. La reutilización funciona y obtenemos una shell en el sistema.

![[Pasted image 20260703192618.png]]

---

## Escalada de privilegios

Ya dentro de la máquina, enumeramos permisos `sudo`, grupos y binarios en busca de una vía de escalada. La enumeración revela el binario `env` como candidato.

![[Pasted image 20260703193123.png]]

Consultamos la ficha de `env` en GTFOBins, que documenta el comando concreto para abusar de este binario y obtener una shell privilegiada.

![[Pasted image 20260703193358.png]]

Ejecutamos el comando indicado en la terminal y obtenemos una shell con privilegios de root, completando el compromiso de la máquina.

![[Pasted image 20260703193519.png]]

---

## Conclusión

La máquina se comprometió encadenando tres fallos: una inyección SQL en el login web que permitía saltarse la autenticación y exponía credenciales, la reutilización de esas credenciales contra SSH y una escalada a root a través del binario `env`.

Cada eslabón habilita el siguiente: sin la exposición de credenciales por la SQLi no habríamos accedido por SSH, y sin ese acceso no habríamos podido enumerar y abusar de `env`. El ejercicio refuerza la importancia de sanear la entrada en las consultas SQL, no reutilizar credenciales entre servicios y revisar los permisos de binarios que permiten escapar a una shell.

⚠️ Realizado con fines educativos y en un entorno controlado.