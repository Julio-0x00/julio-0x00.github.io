---
layout: single
title: Delivery - HackTheBox
excerpt: "Exploramos la máquina Delivery de HackTheBox, donde realizamos reconocimiento de servicios, explotación de credenciales en un sistema de tickets y análisis de bases de datos para escalar privilegios hasta root"
date: 2025-03-10
classes: wide
header:
  teaser: /assets/images/Delivery/Delivery.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - CTF
  - Brute Force
  - Cracking Hashes
---

![](/assets/images/Delivery/Delivery.png){:style="max-width:450px;height:auto;"}

## Resumen

En la máquina Delivery de HackTheBox, identificamos servicios clave expuestos en diferentes puertos, como un sitio web y una plataforma Mattermost. Mediante un sistema de tickets y análisis de credenciales filtradas, conseguimos acceso SSH. Posteriormente, explotamos credenciales almacenadas en un archivo de configuración para obtener acceso a la base de datos, descifrar hashes y escalar privilegios hasta root.

## Reconocimiento

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- Servicio web en el puerto 80

- Servicio desconocido en el puerto 8065

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles:

![](/assets/images/Delivery/Reconocimiento-Puertos.png)

También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Delivery/Reconocimiento-Puertos-Scripts-1.png)
![](/assets/images/Delivery/Reconocimiento-Puertos-Scripts-2.png)

## Página Web

El servicio web en el puerto 80 en el apartado de 'Contact Us', el email hace referencia a un dominio 'delivery.htb':

![](/assets/images/Delivery/Web.png)

![](/assets/images/Delivery/Web-Contact-Us.png)

Editamos nuestro archivo 'etc/hosts' y metemos ese dominio:

![](/assets/images/Delivery/Etc-Hosts-1.png)

Ahora que hemos metido el dominio, vamos a buscar subdominios:

![](/assets/images/Delivery/Subdomains.png)

Encontramos el subdoninio 'helpdesk', lo añadimos al '/etc/hosts':

![](/assets/images/Delivery/Etc-Hosts-2.png)

Nos dirigimos a la web:

![](/assets/images/Delivery/Web-Support-Center.png)

Aqui, vemos que podemos abrir un nuevo ticket, y observar su estado, vamos a crear un ticket:

![](/assets/images/Delivery/Create-Ticket.png)

Una vez creado el ticket, nos dan una id y un correo:

![](/assets/images/Delivery/Request-Ticket.png)

Nos dirigimos a 'check ticket status', para observar el estado del ticket:

![](/assets/images/Delivery/Check-Ticket-Status.png)

Vemos que parece que nos tiene que enviar un correo a la dirección de correo que nos han dado anteriormente:

![](/assets/images/Delivery/Check-Ticket-Status-No-Data.png)

Nos dirigimos al recurso en el puerto '8065', bajo el mismo dominio y subdoninio:

![](/assets/images/Delivery/Web-Login-Mattermost.png)

Nos creamos una cuenta:

![](/assets/images/Delivery/Web-Login-Mattermost-Create-Account.png)

![](/assets/images/Delivery/Web-Login-Mattermost-Create-Account-Request.png)

Volvemos a ver el estado del ticket:

![](/assets/images/Delivery/Check-Ticket-Status-Verify-Account.png)

Abrimos el enlace, para verificar la cuenta:

![](/assets/images/Delivery/Mattermost-Account.png)

Y una vez verificada, iniciamos sesión:

![](/assets/images/Delivery/Mattermost-Blog.png)

Aqui obtenemos unas credenciales para poder entrar via SSH, también hacen referencia a que la palabra 'PleaseSubscribe!' no está en el diccionario 'rockyou', y que si algún hacker consigue los hashes, podrían hacer uso de la herramienta 'hashcat' para crear un diccionario con esta palabra y descifrarlo.

## Reverse Shell

Una vez obtenidas las credenciales de inicio de sesión para el servicio SSH, nos conectamos:

![](/assets/images/Delivery/SSH.png)

La bandera 'user' se encuentra en el directorio 'home' del usuario 'maildeliverer':

![](/assets/images/Delivery/User-Flag.png)

## Escalada de Privilegios

Vamos a buscar en donde se encuentra el recurso 'mattermost':

![](/assets/images/Delivery/Find-Mattermost.png)

Nos dirigimos al recurso, y si listamos las carpetas y los archivos que contiene, vemos que hay una carpeta llamada 'Config', que contiene un archivo de configuración:

![](/assets/images/Delivery/Config.png)

Inspeccionamos el archivo de configuración 'config.json' y vemos que contiene credenciales para acceder a una base de datos mysql:

![](/assets/images/Delivery/Config-File-Content.png)

Una vez obtenidas las credenciales, ejecutamos mysql y listamos las bases de datos:

![](/assets/images/Delivery/DB.png)

Seleccionamos la base de datos de 'mattermost' y listamos las tablas:

![](/assets/images/Delivery/DB-Mattermost.png)

Listamos la tabla 'Users' y listamos los campos 'Username' y 'Password':

![](/assets/images/Delivery/DB-Mattermost-Users.png)

Con la herramienta 'hashcat', vamos a crear un diccionario con la palabra 'PleaseSubscribe!' (anteriormente el usuario 'root' en el foro de mattermost decia que esa palabra no está en el diccionario 'rockyou', y que si algún hacker conseguia sus hashes podrían usar las reglas hashcat para descifrar todas las variaciones de palabras o frases comunes):

![](/assets/images/Delivery/Create-Dictionary.png)

Una vez creado el diccionario, haciendo uso de la herramienta 'John', crackeamos el hash, para obtener la contraseña:

![](/assets/images/Delivery/Crack-Hash.png)

Una vez obtenida la contraseña, migramos al usuario privilegiado:

![](/assets/images/Delivery/Sudo-Su.png)

La bandera 'root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/Delivery/Root-Flag.png)

### Conclusión

La máquina Delivery de HackTheBox nos permite practicar técnicas de reconocimiento de servicios, manipulación de subdominios y explotación de credenciales filtradas en plataformas de mensajería. A través del análisis de un sistema Mattermost y la explotación de credenciales en una base de datos MySQL, logramos escalar privilegios hasta obtener acceso como root. El uso de herramientas como nmap, hashcat y john es clave en este reto.
