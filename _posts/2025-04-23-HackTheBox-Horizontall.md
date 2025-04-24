---
layout: single
title: Horizontall - HackTheBox
excerpt: "Comprometemos la máquina Horizontall de HackTheBox mediante reconocimiento, descubrimiento de subdominios, explotación de Strapi y escalada de privilegios usando PwnKit, obteniendo acceso tanto al usuario como a root"
date: 2025-04-23
classes: wide
header:
  teaser: /assets/images/Horizontall/Horizontall.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - PwnKit
  - Strapi Exploit
  - Laravel
---

![](/assets/images/Horizontall/Horizontall.png){:style="max-width:450px;height:auto;"}

## Resumen

En esta máquina de nivel medio, realizamos un escaneo con nmap y descubrimos servicios en los puertos 22 (SSH) y 80 (HTTP). Mediante herramientas como GoBuster, encontramos un subdominio que nos lleva a un panel de administración corriendo Strapi en versión beta, el cual explotamos para conseguir credenciales y acceso al sistema. Luego, aprovechamos una vulnerabilidad RCE para ejecutar comandos y obtener una reverse shell. Finalmente, escalamos privilegios explotando Laravel con PwnKit, logrando acceso como root.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Horizontall/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- HTTP en el puerto 80

## Página Web

Al hacer el reconocimiento del puerto 80, nmap nos dice que no podemos seguir el redireccionamiento hacia 'horizontall.htb'. Esto lo solucionamos añadiendo el dominio en el archivo '/etc/hosts':

![](/assets/images/Horizontall/Hosts-1.png)

El servicio web en el puerto 80:

![](/assets/images/Horizontall/Web.png)

## Shell

Haciendo uso de la herramienta 'GoBuster' conseguimos descubrir un subdominio:

![](/assets/images/Horizontall/GoBuster-1.png)

El subdominio lo añadimos al archivo '/etc/hosts':

![](/assets/images/Horizontall/Hosts-2.png)

Nos dirigimos al subdominio:

![](/assets/images/Horizontall/Web-Subdomain.png)

Como no hay nada interesante, vamos a volver a hacer uso de la herramienta 'GoBuster':

![](/assets/images/Horizontall/GoBuster-2.png)

Descubrimos un panel administrador:

![](/assets/images/Horizontall/Admin-Panel.png)

Vemos que usa una versión de 'strapi' en beta:

![](/assets/images/Horizontall/Strapi-Version.png)

Haciendo uso de la herramienta 'searchsploit', verificamos que existen algunos exploits para esta versión:

![](/assets/images/Horizontall/Searchsploit.png)

Lo descargamos y lo ejecutamos:

![](/assets/images/Horizontall/Exploit-Strapi.png)

Nos da unas credenciales, así que vamos a iniciar sesión:

![](/assets/images/Horizontall/Admin-Panel-Home.png)

No hay nada interesante en el panel administrador, copiamos el JWT y volvemos a buscar otro exploit:

![](/assets/images/Horizontall/Searchsploit-RCE.png)

Vemos que hay disponible un exploit que nos proporciona la posibilidad de ejecutar comandos.

Nos traemos el exploit y lo ejecutamos con un 'id' como prueba:

![](/assets/images/Horizontall/RCE.png)

Como ya tenemos la posibilidad de ejecutar comandos, vamos a obtener la bandera.

La bandera 'User' la encontramos en el directorio 'home' del usuario 'developer':

![](/assets/images/Horizontall/User-Flag.png)

Nos ponemos por escucha y ejecutamos el exploit para hacer una reverse shell:

![](/assets/images/Horizontall/Rev-Shell-Kill.png)

Hay un proceso con el PID '1975' que nos expulsa, tenemos que matar el proceso.

En la raiz '/' existe un archivo con el nombre '/pid' que cada vez que es ejecutado, da un pid distinto.

Con el siguiente comando que vamos a ejecutar, ejecutamos el archivo '/pid' para que nos de un 'pid', para matarlo y lograr una reverse shell estable:

![](/assets/images/Horizontall/Reverse-Shell.png)

Hacemos el tratamiento de la tty:

    script -c bash
    ctrl + z

    stty raw -echo; fg
                reset
        xterm

    export TERM=xterm SHELL=bash
    stty rows 39 columns 161 # tty -a para saber las filas y columnas

## Escalada de Privilegios

Hacemos un curl en 'http://127.0.0.1:8000', podremos observar que nos da la información de que esta corriendo un 'Laravel v8 (PHP v7.4.18)', esto nos lleva a una herramienta llamada 'PwnKit', ejecutamos lo siguiente en nuestra máquina:

    curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit

Ahora lo pasamos y lo ejecutamos:

![](/assets/images/Horizontall/Exploit-Root.png)

La bandera 'Root' se encuentra en el directorio 'home' del usuario 'root':

    9dd7d77e20f343c5a480d2e500e85d6e

### Conclusión

Horizontall es una máquina que destaca la importancia del reconocimiento profundo, incluyendo descubrimiento de subdominios y paneles ocultos. El uso de exploits públicos para vulnerabilidades conocidas en CMS como Strapi y frameworks como Laravel demuestra cómo configuraciones inseguras y software desactualizado pueden comprometer completamente un sistema. Es un excelente ejemplo práctico de explotación web y escalada local.
