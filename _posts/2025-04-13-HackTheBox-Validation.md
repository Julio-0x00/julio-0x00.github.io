---
layout: single
title: Validation - HackTheBox
excerpt: "Comprometemos la máquina Validation a través de una inyección SQL en un formulario web que nos permite subir una web-shell. Escalamos privilegios accediendo a credenciales root almacenadas en un archivo de configuración expuesto"
date: 2025-04-13
classes: wide
header:
  teaser: /assets/images/Validation/Validation.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - CTF
  - Web Exploitation
  - SQL Injection
---

![](/assets/images/Validation/Validation.png){:style="max-width:450px;height:auto;"}

## Resumen

La máquina Validation de HackTheBox nos plantea un entorno Linux accesible mediante varios servicios web. A través de un formulario web vulnerable a inyección SQL, conseguimos ejecutar comandos en el servidor mediante una web-shell cargada desde el propio backend. Posteriormente, encontramos credenciales de root en un archivo de configuración dentro del directorio web que nos permite escalar privilegios fácilmente y capturar la bandera root


## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Validation/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- HTTP en los puertos 80, 4566, 8080

## Página Web

El servicio web en el puerto 80 está corriendo un formulario de inscripción para la clasificación de Septiembre de UHC:

![](/assets/images/Validation/Web.png)

## Shell

Usamos BurpSuite para interceptar una solicitud POST enviada por el formulario de inscripción:

![](/assets/images/Validation/BurpSuite.png)

Ahora vamos a probar si es vulnerable a SQL injection:

![](/assets/images/Validation/SQL-Injection.png)

Vemos que nos redirige tras la inyección; al seguir la redirección, observamos que devuelve un 1, lo que indica una posible ejecución exitosa de la consulta modificada.

Como ya sabemos que es vulnerable a SQL injection, vamos a crear una web-shell, interceptamos una petición y le añadimos el siguiente payload para crear el archivo 'shell.php':

![](/assets/images/Validation/Create-Web-Shell.png)

Ahora nos dirigimos al navegador y en la URL añadimos el nombre del archivo que acabamos de crear, y ejecutamos un comando básico ‘id’ desde la URL para comprobar la ejecución remota:

![](/assets/images/Validation/Web-Shell.png)

A continuación, reemplazamos la carga útil de la petición por un payload que nos proporciona una reverse shell mientras estamos en escucha:

![](/assets/images/Validation/Reverse-Shell.png)

Una vez obtenido el acceso, hacemos el tratamiento de la tty:

    script -c bash
    ctrl + z

    stty raw -echo; fg
                reset
        xterm

    export TERM=xterm SHELL=bash
    stty rows 39 columns 161 # tty -a para saber las filas y columnas

La bandera 'User' la encontramos en el directorio 'home' del usuario 'htb':

![](/assets/images/Validation/User-Flag.png)

## Escalada de Privilegios

Si observamos en el directorio '/var/www/html/', podemos encontrarnos con un archivo de configuración (normalmente este tipo de archivos suelen tener credenciales o información sensible) 'config.php', que contiene las credenciales del usuario root:

![](/assets/images/Validation/Root-Password.png)

Cambiamos al usuario root:

![](/assets/images/Validation/Sudo-Su.png)

La bandera 'Root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/Validation/Root-Flag.png)

### Conclusión

Validation demuestra lo peligrosas que pueden ser las vulnerabilidades de inyección SQL cuando permiten la escritura de archivos y la ejecución remota de comandos. La máquina también enfatiza la importancia de no almacenar credenciales sensibles en archivos accesibles desde el entorno web. Para entornos reales, es fundamental validar y sanitizar correctamente los inputs, restringir permisos sobre archivos críticos y seguir buenas prácticas como el principio de mínimo privilegio, configuraciones seguras por defecto y la protección del backend frente a accesos no autorizados.

