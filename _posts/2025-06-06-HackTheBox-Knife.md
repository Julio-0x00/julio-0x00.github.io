---
layout: single
title: Knife - HackTheBox
excerpt: "Explotación de una máquina vulnerable en Hack The Box llamada Knife, utilizando una vulnerabilidad en PHP a través de headers manipulados y posterior escalada de privilegios mediante un binario SUID identificado en GTFOBins"
date: 2025-06-06
classes: wide
header:
  teaser: /assets/images/Knife/Knife.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - PHP Vulnerability
  - Web Exploitation
---

![](/assets/images/Knife/Knife.png){:style="max-width:450px;height:auto;"}

## Resumen

En esta máquina llamada Knife de Hack The Box, se realiza un reconocimiento inicial con nmap identificando puertos abiertos, destacando un servicio HTTP en el puerto 80. Analizando con Whatweb, se detecta una versión obsoleta de PHP. Usando Searchsploit, se encuentra un exploit que permite ejecución remota de comandos mediante un header malicioso User-Agentt. Esto se aprovecha para obtener una reverse shell codificada en base64 para evadir detección.

Posteriormente, se accede a la máquina como el usuario james y se localiza la flag de usuario. En la fase de escalada de privilegios, se detecta un binario SUID llamado knife, el cual es conocido por permitir ejecución como root según GTFOBins. Ejecutando el comando adecuado, se obtiene acceso root y se recupera la flag final.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Knife/Reconocimiento-Puertos-Scripts.png)

El servidor expone el siguiente servicio:

- TCP en el puerto 22

- HTTP en el puerto 80

## Página Web

El servicio web en el puerto 80 contiene una web con la apariencia de una agencia europea en medicina:

![](/assets/images/Knife/Web.png)

## Shell

Con la herramienta 'Whatweb', inspeccionamos que versiones utiliza esta web:

![](/assets/images/Knife/Web-Whatweb.png)

Vemos que utiliza una versión de PHP bastante antigua.

Haciendo uso de la herramienta 'Searchsploit', podemos ver que está versión de PHP es vulnerable:

![](/assets/images/Knife/Searchsploit.png)

Nos descargamos el exploit y lo inspeccionamos.

Al inspeccionar el contenido del exploit, nos damos cuenta de que el header es vulnerable, ya que si añadimos un header denominado 'User-Agentt', podemos ejecutar comandos:

![](/assets/images/Knife/Exploit-Vuln-Header.png)

Interceptamos con BurpSuite una petición:

![](/assets/images/Knife/BurpSuite.png)

Añadimos el header con un payload para ejecutar un 'id':

![](/assets/images/Knife/BurpSuite-Rev-Shell.png)

Como ya tenemos la posibilidad de ejecutar comandos, vamos a entablarnos una reverse shell, primero vamos a codificar el payload en base64:

![](/assets/images/Knife/Rev-Shell-Encode-Base64.png)

Después vamos a decirle que nos descodifique nuestra 'reverse shell' codificada a base64, para evitar WAF:

Al mismo tiempo que nos ponemos por escucha.

![](/assets/images/Knife/BurpSuite-Rev-Shell.png)

![](/assets/images/Knife/Shell.png)

La bandera 'User' se encuentra en el directorio 'home' del usuario 'james':

![](/assets/images/Knife/User-Flag.png)

## Escalada de privilegios

Al ejecutar 'sudo -l', podemos ver los permisos SUID que tiene el usuario:

![](/assets/images/Knife/Permissions-SUID.png)

Vemos que tenemos permisos SUID en un binario llamado 'knife', el cual si lo buscamos en la web de 'GTFO Bins' el binario 'knife', veremos que podemos escalar de privilegios:

![](/assets/images/Knife/GTFOBins-Vuln-Bin.png)

![](/assets/images/Knife/GTFOBins-Vuln-Bin-Info.png)

Ejecutamos el comando y nos convertimos en el usuario privilegiado:

![](/assets/images/Knife/Shell-Root.png)

La bandera 'Root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/Knife/Root-Flag.png)

### Conclusión

La máquina Knife demuestra cómo una simple mala configuración en un servicio web puede ser explotada eficazmente para obtener acceso remoto, y cómo herramientas como GTFOBins son clave para identificar vectores de escalada de privilegios. El reto enfatiza la importancia de mantener actualizados los servicios web y de revisar permisos en binarios con SUID.
