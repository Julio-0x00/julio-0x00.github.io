---
layout: single
title: Return - HackTheBox
excerpt: "Obtenemos acceso inicial desde una interfaz web de impresora expuesta, y escalamos privilegios manipulando servicios como 'vss' con sc.exe gracias a pertenecer al grupo Server Operators"
date: 2025-04-05
classes: wide
header:
  teaser: /assets/images/Return/Return.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - CTF
  - Windows Exploitation
  - Print Operators
---

![](/assets/images/Return/Return.png){:style="max-width:450px;height:auto;"}

## Resumen

La máquina Return de HackTheBox presenta un entorno Windows en el que obtenemos acceso inicial gracias a una interfaz de impresora expuesta que revela credenciales en texto claro. Usamos estas credenciales con Evil-WinRM para acceder como el usuario 'svc-printer'. Posteriormente, aprovechamos que este usuario pertenece al grupo 'Server Operators' para manipular el servicio 'vss' con 'sc.exe' y lograr la ejecución de comandos como NT AUTHORITY\SYSTEM, obteniendo así la bandera root.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles:

![](/assets/images/Return/Reconocimiento-Puertos.png)

También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Return/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- Domain en el puerto 53

- HTTP en el puerto 80

- Kerberos-sec en el puerto 88

- MSRPC en el puerto 135

- Netbios-SSN en el puerto 139

- LDAP en el puerto 389

- Microsoft-DS en el puerto 445

- Kpasswd5 en el puerto 464

- Ncacn_HTTP en el puerto 593

- TCPwrapped en el puerto 636

- LDAP en el puerto 3268

- TCPwrapped en el puerto 3269

- HTTP en el puerto 5985

- MC-NMF en el puerto 9389

- HTTP en el puerto 47001

- MSRPC en los puertos 49664, 49665, 49666, 49667, 49671, 49674, 49675, 49679, 49682, 49697

## Página Web

El servicio web en el puerto 80 está corriendo el panel admin de la impresora, y no necesita credenciales ni permisos para acceder:

![](/assets/images/Return/Web.png)

## Shell

En el apartado 'settings', si le agregamos nuestra dirección IP y un puerto, y nos ponemos en escucha, nos proporcionará en texto claro credenciales, tanto el usuario, como la contraseña que utiliza en el sistema la impresora:

![](/assets/images/Return/Web-Settings.png)

![](/assets/images/Return/Credentials-SVC.png)

Ahora con la herramienta 'Evil-WinRM', vamos a obtener una shell proporcionando las credenciales:

![](/assets/images/Return/Shell-EvilWinRM.png)

La bandera 'User', la encontramos en el directorio 'Documents' del usuario 'svc-printer':

![](/assets/images/Return/User-Flag.png)

## Escalada de Privilegios

Vamos a ver la información del usuario 'svc-printer':

![](/assets/images/Return/SVC-User-Info.png)

Vemos que el usuario 'svc-printer' pertenece al grupo 'Server Operators' y también al grupo 'Print Operators' lo que nos da suficientes permisos para modificar la configuración de ciertos servicios como 'vss' usando la herramienta 'sc.exe'.

Vamos a pasar a la máquina Return la herramienta 'nc.exe':

![](/assets/images/Return/Upload-NC.png)

Una vez pasamos 'nc.exe' a la máquina Return, configuramos el servicio 'vss' con 'sc.exe':

![](/assets/images/Return/SC-Config.png)

Paramos primero el servicio 'vss' (para evitar conflictos al reconfigurarlo), y iniciamos el servicio 'vss':

![](/assets/images/Return/SC-Start.png)

Y con esto obtenemos acceso como 'NT AUTHORITY\SYSTEM'.

La bandera 'Root' se encuentra en el directorio 'Desktop' del usuario 'Administrator':

![](/assets/images/Return/Root-Flag.png)

### Conclusión

Esta máquina demuestra cómo servicios aparentemente inofensivos, como los paneles de administración de impresoras, pueden exponer credenciales sensibles si no están adecuadamente protegidos. También resalta los riesgos de otorgar membresía a grupos como 'Server Operators' a usuarios no administradores, ya que esto les otorga capacidades avanzadas sobre los servicios del sistema, lo que facilita una escalada de privilegios. En entornos reales, es crucial asegurar que interfaces administrativas web sean debidamente autenticadas, restringiendo el acceso y evitando el uso excesivo de privilegios, como los del grupo 'Server Operators', que pueden ser explotados por usuarios no autorizados. Esta máquina es un excelente ejercicio para practicar técnicas de escalada de privilegios en entornos Windows reales, y subraya la importancia de una correcta gestión de privilegios en sistemas críticos.
