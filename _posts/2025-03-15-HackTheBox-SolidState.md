---
layout: single
title: SolidState - HackTheBox
excerpt: "Explotación de una vulnerabilidad en Apache James 2.3.2 para obtener acceso inicial, filtración de credenciales a través de POP3 y escalada de privilegios mediante un script en /opt"
date: 2025-03-15
classes: wide
header:
  teaser: /assets/images/SolidState/SolidState.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - CTF
  - Apache-James
  - POP3
---

![](/assets/images/SolidState/SolidState.png){:style="max-width:450px;height:auto;"}

## Resumen

SolidState es una máquina de HackTheBox en la que se explota una vulnerabilidad en Apache James 2.3.2 para obtener acceso inicial. Luego, se filtran credenciales a través de POP3 para acceder vía SSH. Finalmente, se escala privilegios aprovechando un script con permisos de escritura en /opt, logrando acceso como root.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles:

![](/assets/images/SolidState/Reconocimiento-Puertos.png)

También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/SolidState/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- SMTP en el puerto 25

- HTTP en el puerto 80

- POP3 en el puerto 110

- NNTP en el puerto 119

- RSIP? en el puerto 4555 (Apache James 2.3.2)

## Página Web

El servicio web en el puerto 80 no contiene información relevante:

![](/assets/images/SolidState/Web.png)

## Shell

En el puerto 4555 corre un 'Apache James 2.3.2', si buscamos un exploit con la herramienta 'searchsploit', nos aparecerán varios para esta version, seleccionamos uno:

![](/assets/images/SolidState/Searchsploit.png)

Una vez nos traemos a nuestro equipo el exploit, lo ejecutamos:

![](/assets/images/SolidState/Run-Exploit.png)

Este script en python lo que hace es aprovechar una vulnerabilidad en la herramienta de administración remota de Apache James, que por defecto utiliza las credenciales 'root' tanto para el usuario como para la contraseña. Crea un nuevo usuario con un nombre especialmente diseñado que incluye secuencias de directorios relativos, como '../../../../../../../../etc/bash_completion.d'.

Esta técnica permite que el servidor cree un directorio de usuario fuera del directorio de instalación previsto, específicamente en '/etc/bash_completion.d'. Luego, el exploit envía un correo electrónico con una carga útil al nuevo usuario. Debido a la ubicación del directorio, el contenido del correo se guarda en '/etc/bash_completion.d', lo que provoca que la carga útil se ejecute la próxima vez que alguien inicie sesión en el sistema y se cargue el script de bash_completion.

Por lo que ahora nos toca iniciar sesión mediante Telnet, para cambiar la contraseña de un usuario (en este caso el que nos interesa es 'mindy'):

![](/assets/images/SolidState/Telnet.png)

Ahora que hemos cambiada la contaseña, nos conectamos al servicio que corre bajo el puerto '110' (POP3), ya que hay un correo que envia el usuario 'James' al usuario 'Mindy' con su contraseña:

![](/assets/images/SolidState/Mail.png)

Una vez tenemos las credenciales de 'Mindy', nos conectamos por 'SSH', mientras nos ponemos en escucha por el puerto anteriormente proporcionado por el exploit:

![](/assets/images/SolidState/SSH.png)

![](/assets/images/SolidState/Shell.png)

La bandera 'user' se encuentra en el directorio 'home' del usuario 'mindy':

![](/assets/images/SolidState/User-Flag.png)

## Escalada de Privilegios

En el directorio '/opt' existe un archivo que se ejecuta cada 3 minutos y tenemos permisos de escritura:

![](/assets/images/SolidState/File.png)

Lo modificamos para que de permisos 'SUID' al binario de bash:

![](/assets/images/SolidState/Content-File.png)

Una vez el usuario privilegiado haya ejecutado el script, ejecutamos una bash con la flag '-p' (para indicar que queremos ejecutar una bash como usuario privilegiado):

![](/assets/images/SolidState/Bash-p.png)

La bandera 'root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/SolidState/Root-Flag.png)

### Conclusión

SolidState demuestra la importancia de asegurar servicios expuestos y de restringir credenciales por defecto. También resalta la necesidad de monitorear scripts con permisos inseguros, ya que pueden permitir la escalada de privilegios.
