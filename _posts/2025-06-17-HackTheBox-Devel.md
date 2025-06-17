---
layout: single
title: Devel - HackTheBox
excerpt: "Resolución de la máquina Devel de HackTheBox. Explotación de servicios FTP e IIS en Windows para obtener acceso inicial, seguido de una escalada de privilegios mediante vulnerabilidad local en afd.sys"
date: 2025-06-17
classes: wide
header:
  teaser: /assets/images/Devel/Devel.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - Windows
  - Windows Privilege Escalation
  - Web shell
---

![](/assets/images/Devel/Devel.png){:style="max-width:450px;height:auto;"}

## Resumen

En esta máquina de HackTheBox, aprovechamos un servidor FTP con acceso anónimo para subir una web shell ASPX a un servidor IIS. Con esta shell conseguimos acceso inicial al sistema. Posteriormente, realizamos una escalada de privilegios aprovechando una vulnerabilidad local (afd.sys) en la versión de Windows que corría la máquina. Finalmente, obtenemos ambas banderas: user.txt y root.txt.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Devel/Reconocimiento-Puertos-Scripts.png)

El servidor expone el siguiente servicio:

- FTP en el puerto 21

- HTTP en el puerto 80

## Página Web

El servicio web en el puerto 80 contiene una web predeterminada de Internet Information Services (IIS 7):

![](/assets/images/Devel/Web.png)

## Shell

Podemos ver que el script de nmap (ftp-anon) puede iniciar sesión por FTP con el usuario 'anonymous', podemos acceder a los recursos compartidos de FTP:

![](/assets/images/Devel/FTP-Anon.png)

Ahora vamos a subir un archivo ASPX, para hacernos con una web shell:

![](/assets/images/Devel/Aspx-cmd.png)

![](/assets/images/Devel/Aspx-cmd-copy.png)

Lo subimos a la máquina:

![](/assets/images/Devel/Aspx-cmd-ftp.png)

Y nos dirigimos al navegador:

![](/assets/images/Devel/Web-Shell.png)

Para poder hacernos una reverse shell antes vamos a subir a la máquina 'nc.exe', en el modo 'binary' (de lo contrario, se va a corromper y no se va a ejecutar correctamente):

![](/assets/images/Devel/NC-ftp.png)

Nos ponemos en escucha y nos dirigimos al navegador, para ejecutar lo siguiente:

![](/assets/images/Devel/Shell.png)

La bandera 'User' se encuentra en el directorio 'Desktop' del usuario 'babis':

![](/assets/images/Devel/User-Flag.png)

## Escalada de privilegios

Si ejecutamos el comando 'systeminfo', podremos ver qué versión de Windows está usando:

![](/assets/images/Devel/Systeminfo.png)

Si buscamos un exploit para la versión del sistema en el navegador, podemos ver que se hace referencia a un exploit llamado 'afd.sys':

![](/assets/images/Devel/Vuln-Search-1.png)

Si buscamos algún exploit del archivo antes mencionado, podemos encontrarnos con un binario en un repositorio de GitHub:

![](/assets/images/Devel/Vuln-Search-2.png)

Lo descargamos y lo movemos a nuestro directorio de trabajo y lo compartimos por SMB o lo subimos al servidor mediante FTP:

![](/assets/images/Devel/Exploit-smb.png)

Lanzamos el ejecutable y nos convertimos en el usuario privilegiado:

![](/assets/images/Devel/Shell-Root.png)

La bandera 'Root' se encuentra en el directorio 'Desktop' del usuario 'Administrator':

![](/assets/images/Devel/Root-Flag.png)

### Conclusión

Devel es una máquina que muestra la importancia de restringir accesos anónimos a servicios como FTP y mantener el sistema operativo actualizado. Combinando técnicas básicas de enumeración y reconocimiento con la explotación de vulnerabilidades conocidas, conseguimos una escalada completa de privilegios desde un acceso web inicial.
