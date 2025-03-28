---
layout: single
title: Jeeves - HackTheBox
excerpt: "Explotación de un Jenkins expuesto en HackTheBox para obtener acceso inicial, escalada de privilegios con credenciales extraídas de una base de datos Keepass y recuperación de la bandera root oculta en un flujo alternativo de datos en NTFS"
date: 2025-03-28
classes: wide
header:
  teaser: /assets/images/Jeeves/Jeeves.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - Jenkins
  - Keepass
  - AD
  - CTF
---

![](/assets/images/Jeeves/Jeeves.png){:style="max-width:450px;height:auto;"}

## Resumen

- Reconocimiento: Identificación de servicios mediante Nmap.

- Explotación: Acceso a un Jenkins expuesto y ejecución de comandos maliciosos.

- Escalada de privilegios: Extracción de credenciales desde una base de datos Keepass y ejecución de una reverse shell con privilegios administrativos.

- Bandera root: Ubicada en un flujo alternativo de datos de NTFS.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles:

![](/assets/images/Jeeves/Reconocimiento-Puertos.png)

También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Jeeves/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- HTTP en el puerto 80

- MSRPC en el puerto 135

- Microsoft-ds en el puerto 445

- HTTP en el puerto 5000 (Jetty 9.4)

## Página Web

El servicio web en el puerto 80 no contiene información relevante:

![](/assets/images/Jeeves/Web-80.png)

El servicio web en el puerto 5000 nos encontramos que en la raíz no hay nada:

![](/assets/images/Jeeves/Web-Root.png)

## Shell

En el servicio web en el puerto 5000, si utilizamos la herramienta 'dirbuster', podemos encontrarnos con un directorio:

![](/assets/images/Jeeves/Dirb.png)

Si el directorio lo introducimos en el navegador, nos encontramos con un panel Jenkins:

![](/assets/images/Jeeves/Jenkins-Panel.png)

Una vez estamos en el panel, creamos un nuevo trabajo:

![](/assets/images/Jeeves/Create-New-Job.png)

Bajamos y seleccionamos 'add build step' a 'Execute Windows batch command':

![](/assets/images/Jeeves/Create-New-Job-2.png)

Y le añadimos el comando para hacer una reverse shell:

![](/assets/images/Jeeves/Create-New-Job-Shell.webp)

Una vez tenemos una shell, nos hacemos con la bandera 'user', ubicada en el escritorio del usuario 'kohsuke':

    e3232272596fb47950d59c4cf1e7066a

## Escalada de Privilegios

Hacendo una busqueda por el sistema, nos encontramos en el directorio 'Documents' del usuario 'kohsuke' un archivo llamado 'CEH.kdbx', es una base de datos de contraseñas:

![](/assets/images/Jeeves/File-DB-Pass.png)

Nos pasamos a nuestro equipo el archivo:

- Nuestro equipo:

    ```impacket smbserver smbfolder $(pwd) -smb2support```

- Máquina Jeeves:

![](/assets/images/Jeeves/Copy-File.png)

Una vez pasamos el archivo a nuestro equipo, vamos a utilizar una herramienta llamada 'keepass2john', para poder convertirla en un archivo de texto y poder hacerle fuerza bruta, para averiguar la contraeña:

    keepass2john CEH.kdbx > CEH.txt

Una vez tenemos el archivo a un archivo de texto, lo crackeamos:

![](/assets/images/Jeeves/Crack-File.png)

Y una vez obtenida la credencial, abrimos el archivo (el archivo original, no el de texto):

    keepassxc CEH.kdbx

Y introducimos la credencial:

![](/assets/images/Jeeves/Login-DB.png)

Y buscamos la credencial del usuario 'administrator':

![](/assets/images/Jeeves/Credential-AD.png)

Ahora que tenemos la credencial, vamos a pasar el ejecutable 'nc.exe' para hacernos una reverse shell como administradores:

![](/assets/images/Jeeves/NC-Shell.png)

![](/assets/images/Jeeves/AD-Rev-Shell.png)

La bandera 'root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/Jeeves/Dir-Flag.png)

Si le decimos que nos muestre el contenido, nos dirá lo siguiente:

![](/assets/images/Jeeves/Dir-Flag-Content.png)

Con el siguiente comando 'Dir /r', podemos ver si hay flujos alternativos, ya que 'dir /r', nos muestra los atributos especiales:

![](/assets/images/Jeeves/Dir-Hide-Flag-Content.png)

Aqui vemos que el archivo 'hm.txt' tiene un flujo alternativo 'root.txt', la notación ':$DATA' indica un flujo alternativo de datos en 'NTFS'.

Ahora vamos a mostrar el contenido del flujo alternativo 'root.txt':

![](/assets/images/Jeeves/Root-Flag.png)

### Conclusión

Jeeves es una máquina de HackTheBox que pone a prueba habilidades de reconocimiento, explotación y escalada de privilegios en un entorno Windows. La explotación de un servidor Jenkins expuesto permitió la obtención de una shell inicial. Luego, la escalada de privilegios se logró mediante la recuperación de credenciales almacenadas en un archivo Keepass. Finalmente, se descubrió la bandera oculta en un flujo alternativo de datos en NTFS.
