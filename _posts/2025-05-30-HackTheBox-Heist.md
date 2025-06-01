---
layout: single
title: Heist - HackTheBox
excerpt: "Acceso inicial mediante credenciales expuestas en un sistema Windows con servicios abiertos. Se consigue la elevación de privilegios utilizando volcados de memoria del navegador para obtener credenciales administrativas"
date: 2025-05-30
classes: wide
header:
  teaser: /assets/images/Heist/Heist.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - Password Spraying
  - ProcDump
  - Windows
---

![](/assets/images/Heist/Heist.png){:style="max-width:450px;height:auto;"}

## Resumen

La máquina Heist de Hack The Box nos presenta un entorno Windows con varios servicios expuestos, entre ellos HTTP, MSRPC y Microsoft-DS. Tras un reconocimiento inicial, se accede a un panel web donde es posible obtener hashes de contraseñas. Estos son crackeados usando herramientas como John The Ripper y Netexec. Posteriormente, se realiza un ataque de password spraying para obtener acceso remoto mediante Evil-WinRM. La escalada de privilegios se logra al extraer credenciales administrativas desde la memoria del proceso Firefox, utilizando ProcDump.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles:

![](/assets/images/Heist/Reconocimiento-Puertos.png)

También ejecutamos scripts básicos de reconocimiento con nmap.

El servidor expone los siguientes servicios:

- HTTP en el puerto 80

- MSRPC en el puerto 135

- Microsoft-DS en el puerto 445

- HTTP en el puerto 49669

## Página Web

En el servicio web en el puerto 80 tenemos un panel de autenticación, si nos logeamos como 'guest' tendremos acceso al recurso:

![](/assets/images/Heist/Web-Login.png)

Una vez iniciamos sesión como 'guest', podemos ver un chat interno de los trabajadores, si nos dirigimos a lo que adjunta el usuario 'Hazard', podemos obtener un hash del usuario 'Hazard':

![](/assets/images/Heist/Web-Issues.png)
![](/assets/images/Heist/Web-Issues-Hash.png)

## Shell

Crackeamos el hash con la herramienta 'John The Ripper':

![](/assets/images/Heist/Hash-Crack-File.png)
![](/assets/images/Heist/Hash-Crack-Pass.png)

Ya sabemos la contraseña, si intentamos conectarnos al servidor con las credenciales mediante la herramienta 'WinRM' (Netexec), no vamos a poder:

![](/assets/images/Heist/WinRM.png)

Vamos a intentar desencriptar la contraseña de tipo 7 de cisco del usuario 'admin':

![](/assets/images/Heist/Hash-Admin.png)
![](/assets/images/Heist/Hash-Crack-Admin-Pass.png)

Si hacemos un 'password-spraying' con la contraseña, vemos que hay un usuario en el sistema que sí tiene como credencial esa contraseña:

![](/assets/images/Heist/Password-Spraying-1.png)
![](/assets/images/Heist/Password-Spraying-2.png)

Obtenemos una shell con las credenciales y con la herramienta 'Evil-WinRM':

![](/assets/images/Heist/Shell.png)

La bandera 'User' se encuentra en el directorio 'Desktop' del usuario 'Chase':

![](/assets/images/Heist/User-Flag.png)

## Escalada de privilegios

Si observamos la lista de procesos, podemos encontrarnos con el proceso del navegador 'Firefox', podemos dumpear su memoria para ver si tiene credenciales de algún panel de autenticación.

![](/assets/images/Heist/List-Process.png)

Para dumpear la memoria del proceso, existe una herramienta llamada 'ProcDump', lo descargamos y lo añadimos a la máquina:

![](/assets/images/Heist/ProcDump-1.png)
![](/assets/images/Heist/ProcDump-2.png)

Una vez tenemos la herramienta en la máquina, la ejecutamos añadiendo la ID del proceso que más consumo de CPU use Firefox:

![](/assets/images/Heist/Dump.png)

Pasamos el dump a nuestra máquina:

![](/assets/images/Heist/Copy-Dump-1.png)
![](/assets/images/Heist/Copy-Dump-2.png)

Y filtramos por la contraseña:

![](/assets/images/Heist/Password-Root.png)

Una vez tenemos la credencial, iniciamos una shell como administrador:

![](/assets/images/Heist/Shell-Root.png)

La bandera 'Root' se encuentra en el directorio 'Desktop' del usuario 'Administrator':

![](/assets/images/Heist/Root-Flag.png)

### Conclusión

La máquina Heist demuestra la importancia de proteger los canales internos de comunicación y la sensibilidad de credenciales almacenadas en memoria. Desde el acceso inicial vía credenciales débiles, hasta la explotación de procesos para escalar privilegios, este reto ilustra técnicas comunes en escenarios reales de Red Team, como el password spraying, la reutilización de contraseñas y el análisis de dumps de memoria para obtener credenciales críticas.
