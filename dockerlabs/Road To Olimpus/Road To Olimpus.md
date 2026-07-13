# Road to Olympus

En este writeup resolvemos el laboratorio **Road to Olympus** de DockerLabs, formado por tres máquinas encadenadas: **Hades**, **Poseidón** y **Zeus**. No todas son accesibles directamente: hay que comprometer cada una para pivotar hacia la siguiente, avanzando por la infraestructura hasta llegar a la última.

El objetivo no es solo tomar cada máquina, sino entender el porqué de cada paso: cómo una pista dejada en el código nos da el primer acceso, cómo montamos túneles con chisel para llegar a redes que no vemos desde fuera, y cómo cada compromiso nos acerca a la siguiente máquina.

---

## Topología del laboratorio

Al descomprimir el laboratorio vemos que se despliegan tres máquinas: Hades, Poseidón y Zeus. Empezaremos por Hades.

![](<imagenes/Pasted image 20260713213805.png>)

El esquema de cómo se conectan las tres máquinas queda así, lo que nos anticipa que tendremos que pivotar de una a otra:

![](<imagenes/Pasted image 20260713172338.png>)

---

## Hades (10.10.10.2) — primera máquina

### Escaneo de puertos

Empezamos con un escaneo completo de la primera máquina:

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 10.10.10.2 -oN Escaneo
```

![](<imagenes/Pasted image 20260713183605.png>)

La versión de SSH es bastante moderna, así que la obviamos y nos centramos en el puerto 80, que corre **Werkzeug**, el servidor de desarrollo que usa **Flask** (el framework web de Python).

### Análisis del servicio web

Al abrir `10.10.10.2` en el navegador nos aparece lo siguiente:

![](<imagenes/Pasted image 20260713171616.png>)

Y al pulsar en cerrar, esto otro:

![](<imagenes/Pasted image 20260713171745.png>)

Inspeccionando el código fuente, al final encontramos una línea comentada muy reveladora:

```html
<!-- Guardar en KeePass y borrar al verlo. Nueva contraseña para el servicio SSH de cerbero el perro de 3 cabezas JZKECZ2NPJAWOTT2JVTU42SVM5HGU23HJZVFCZ22NJGWOTTNKVTU26SJM5GXUQLHJV5ESZ2NPJEWOTLKIU6Q==== -->
```

La cadena está codificada, así que la decodificamos y obtenemos la contraseña del servicio SSH del usuario **cerbero**:

![](<imagenes/Pasted image 20260713183734.png>)

La contraseña es **P0seidón2022!**.

### Acceso por SSH

Con esas credenciales accedemos por SSH a Hades:

![](<imagenes/Pasted image 20260713183856.png>)

Con esto damos por hecha la primera máquina.

---

## Pivoting hacia Poseidón (chisel + socat + proxychains)

Poseidón no es accesible directamente desde nuestra máquina; hay que pasar por Hades. Para ello montaremos un túnel con **chisel**.

Descargamos la versión más reciente de chisel:

![](<imagenes/Pasted image 20260713184054.png>)

Lo descomprimimos y lo renombramos a `chisel` para trabajar más cómodos:

![](<imagenes/Pasted image 20260713184223.png>)

También descargamos **socat**, que necesitaremos más adelante, y lo renombramos a `socat`:

![](<imagenes/Pasted image 20260713184333.png>)

Ya lo tenemos todo descargado y toca pasarlo a la máquina Hades. Como disponemos de credenciales de SSH, usamos **scp** para transferir tanto chisel como socat desde nuestra máquina a Hades:

![](<imagenes/Pasted image 20260713184719.png>)

![](<imagenes/Pasted image 20260713184726.png>)

Una vez en la máquina, les damos permisos de ejecución:

![](<imagenes/Pasted image 20260713184843.png>)

Levantamos el servidor de chisel en nuestra Kali por el puerto 1234:

![](<imagenes/Pasted image 20260713185051.png>)

Y en la terminal de cerbero (Hades) lanzamos el cliente de chisel:

![](<imagenes/Pasted image 20260713185157.png>)

Toca editar el archivo `/etc/proxychains.conf`: comentamos la línea de `strict_chain` y descomentamos la de `dynamic_chain`.

![](<imagenes/Pasted image 20260713185724.png>)

Por último, comprobamos que abajo del todo esté descomentada la línea `socks5 127.0.0.1 1080`, que es el puerto por defecto que usa chisel.

![](<imagenes/Pasted image 20260713185949.png>)

A partir de aquí, todas nuestras acciones contra la red de Poseidón deben ejecutarse a través de **proxychains** / **proxychains4**, que usan el archivo de configuración que acabamos de editar.

---

## Poseidón (20.20.20.3) — segunda máquina

### Escaneo a través del túnel

Con el túnel montado, escaneamos Poseidón con nmap a través de proxychains para ver sus puertos, servicios y versiones:

```bash
proxychains4 -q nmap -sT -Pn -p 22,80 20.20.20.3
```

![](<imagenes/Pasted image 20260713192143.png>)

Vemos los puertos 22 y 80, así que miraremos el 80 en el navegador. Antes tenemos que configurar **FoxyProxy** para que el tráfico del navegador pase por el túnel:

![](<imagenes/Pasted image 20260713192424.png>)

Al poner `20.20.20.3` en el navegador vemos lo siguiente:

![](<imagenes/Pasted image 20260713200502.png>)

En la barra de navegación hay tres apartados: Buscar, Ranking y Perfil. Nos interesa **Buscar**, que lleva a un subdirectorio con un sistema de búsqueda:

![](<imagenes/Pasted image 20260713200605.png>)

Revisamos el código fuente para ver cómo tramita la petición este buscador:

![](<imagenes/Pasted image 20260713200629.png>)

Observamos que envía mediante el método **POST** una petición al archivo **database.php**, así que entendemos que pasa el parámetro del campo de búsqueda y con él realiza la consulta a la base de datos.

### Inyección SQL (SQLite)

Intentamos usar **sqlmap** para volcar la base de datos, pero no funcionó. Tras varios intentos por averiguar qué motor se usaba, deducimos que es **SQLite**, ya que este motor emplea una tabla interna con el esquema de la base de datos llamada **sqlite_master**.

Para leer esa información introducimos en el campo de búsqueda la consulta:

```sql
select name from sqlite_master
```

![](<imagenes/Pasted image 20260713200735.png>)

Aparecen tablas que nos interesan, como las de usuarios y contraseñas, así que las consultamos:

![](<imagenes/Pasted image 20260713200836.png>)

![](<imagenes/Pasted image 20260713200847.png>)

Obtenemos los usuarios **poseidon** y **megalodon**, con contraseñas que parecen codificadas:

```
$sha1$oceanos$QqFgxFPmqRex1ZKFCZ2ONJKWOTTNKFTU46SBM5ZKFCZ2ONJKWOTTNKFTU4GQPdkh3nQSWp3I=
$sha1$hahahaha$JZKFCZ2ONJKWOTTNKFTU46SBM5HG2TLHJV5ECZ2NPJEWOTL2IFTU26SFM5GXU23HJVVEKPI=
```

Las decodificamos:

![](<imagenes/Pasted image 20260713201422.png>)

Una de ellas corresponde a **megalodon → Templ02019!**.

### Acceso por SSH y escalada a root

Una de las contraseñas no funciona, pero la otra sí, así que entramos por SSH a Poseidón:

![](<imagenes/Pasted image 20260713201521.png>)

Con esto ya estamos dentro de la segunda máquina.

Una vez conectados, vemos que tenemos permisos en `sudoers` para ejecutar cualquier cosa, así que con un `sudo bash` nos convertimos en root:

![](<imagenes/Pasted image 20260713201625.png>)

---

## Pivoting hacia Zeus (segundo salto)

Poseidón tiene conexión con la última máquina, Zeus, así que repetimos una idea parecida a la del primer salto.

Pasamos chisel a Poseidón:

![](<imagenes/Pasted image 20260713202042.png>)

Arrancamos socat para que reenvíe el puerto 1111 hacia nuestro chisel server en Kali:

![](<imagenes/Pasted image 20260713202416.png>)

Lanzamos chisel dentro de Poseidón:

![](<imagenes/Pasted image 20260713202714.png>)

Y reconfiguramos el archivo `/etc/proxychains.conf`, comentando la entrada anterior y añadiendo la nueva:

![](<imagenes/Pasted image 20260713202816.png>)

Con el nuevo túnel, escaneamos a ver qué hay:

![](<imagenes/Pasted image 20260713203048.png>)

Comprobamos si llegamos a Zeus con curl:

```bash
proxychains4 -q curl -s -m 8 http://30.30.30.3/ -o /dev/null -w "HTTP: %{http_code}\n"
```

Nos devuelve **HTTP 200**, así que la conexión está OK:

![](<imagenes/Pasted image 20260713203237.png>)

---

## Zeus (30.30.30.3) — tercera máquina

### Enumeración SMB con enum4linux

Vemos que Zeus tiene abiertos los puertos **21, 22, 80, 139 y 445**. En los puertos 139 y 445 corre **SAMBA**, así que usamos **enum4linux** para obtener información, en concreto para enumerar los usuarios:

![](<imagenes/Pasted image 20260713203337.png>)

Descubrimos dos usuarios: **rayito** y **hercules**.

### Fuerza bruta FTP con Hydra

Con los usuarios en mano, podemos intentar un ataque de fuerza bruta contra el puerto 21 (FTP). Antes creamos un archivo con los usuarios:

![](<imagenes/Pasted image 20260713203634.png>)

Para la fuerza bruta usamos una versión reducida de rockyou, ya que el ataque a través del túnel es lento:

```bash
head -5000 /usr/share/wordlists/rockyou.txt > minirock
proxychains4 -q hydra -L users -P minirock ftp://30.30.30.3
```

![](<imagenes/Pasted image 20260713204356.png>)

Después de un rato largo, Hydra nos da la contraseña de **hercules**:

![](<imagenes/Pasted image 20260713212109.png>)

### Acceso FTP y contraseña de rayito

Con esa contraseña accedemos por FTP:

![](<imagenes/Pasted image 20260713212302.png>)

Dentro vemos un archivo, así que entramos a verlo. En su contenido encontramos una cadena que, al decodificarla, nos da la contraseña del usuario **rayito**:

![](<imagenes/Pasted image 20260713212755.png>)

### Acceso por SSH como rayito

Solo nos queda acceder por SSH con esas credenciales:

![](<imagenes/Pasted image 20260713212903.png>)

Con esto completamos la tercera y última máquina.

---

## Conclusión

**Road to Olympus** se resuelve encadenando tres máquinas y dos saltos de pivoting. En **Hades**, una pista dejada en un comentario del código HTML nos entrega, tras decodificarla, la contraseña SSH de cerbero. Desde ahí montamos un túnel con chisel, socat y proxychains para alcanzar **Poseidón**, donde una inyección SQL sobre SQLite nos expone los usuarios y sus contraseñas codificadas; con las de megalodon entramos por SSH y, gracias a un `sudoers` permisivo, escalamos a root. Un segundo salto de pivoting nos lleva a **Zeus**, donde enumeramos usuarios por SMB con enum4linux, obtenemos la contraseña de hercules por fuerza bruta en FTP y, dentro del FTP, recuperamos la contraseña de rayito para el acceso final por SSH.

El laboratorio refuerza varias ideas clave: nunca dejar secretos en comentarios del código, no almacenar contraseñas en formatos fácilmente reversibles, evitar configuraciones de `sudo` que permitan ejecutar cualquier cosa, y entender el pivoting como herramienta para alcanzar redes que no son visibles directamente desde el exterior.

⚠️ Realizado con fines educativos y en un entorno controlado.
