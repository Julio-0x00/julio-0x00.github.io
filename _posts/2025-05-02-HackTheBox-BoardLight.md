---
layout: single
title: BoardLight - HackTheBox
excerpt: "Explotamos una instalación vulnerable de Dolibarr en un entorno empresarial simulado para obtener acceso inicial, escalar privilegios mediante un binario SUID vulnerable y capturar ambas flags: user y root"
date: 2025-05-02
classes: wide
header:
  teaser: /assets/images/BoardLight/BoardLight.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - Dolibarr
  - Web shell
  - SUID Exploit
---

![](/assets/images/BoardLight/BoardLight.png){:style="max-width:450px;height:auto;"}

## Resumen

En la máquina BoardLight de HackTheBox, identificamos servicios expuestos mediante escaneo de puertos y accedimos a un panel Dolibarr a través de credenciales por defecto. Aprovechando una vulnerabilidad (CVE-2023-30253), subimos una web shell y obtuvimos acceso remoto. Posteriormente, encontramos credenciales en archivos de configuración y cambiamos al usuario Larissa. Finalmente, escalamos privilegios explotando un binario SUID vulnerable de Enlightenment, logrando así acceso root al sistema.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/BoardLight/Reconocimiento-Puertos-Scripts.png)

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- HTTP en los puertos 80

## Página Web

El servicio web en el puerto 80 contiene una web sobre una consultora de ciberseguridad:

![](/assets/images/BoardLight/Web.png)

Si nos dirigimos al apartado 'About' podemos ver que hace referencia a un dominio:

![](/assets/images/BoardLight/Web-About.png)

Lo añadimos a nuestro archivo '/etc/hosts':

![](/assets/images/BoardLight/Hosts-File-1.png)

Y haciendo uso de la herramienta 'ffuf', nos encontramos con un subdominio:

![](/assets/images/BoardLight/Ffuf.png)

Lo añadimos a nuestro archivo '/etc/hosts':

![](/assets/images/BoardLight/Hosts-File-2.png)

Si nos dirigimos al navegador, podemos ver un panel admininstrador al recurso 'Dolibarr':

![](/assets/images/BoardLight/Admin-Panel.png)

Para acceder como administrador solo hace falta proporcionar las credenciales por defecto 'admin:admin':

![](/assets/images/BoardLight/Dashboard.png)

## Shell

Una vez estamos en el panel administrador, vemos la versión que utiliza 'Dolibarr', y haciendo una búsqueda, podemos encontranos con una vulnerabilidad (CVE-2023-30253).

En el panel, nos dirigimos a 'Websites':

![](/assets/images/BoardLight/Websites.png)

Le damos a agregar (+):

![](/assets/images/BoardLight/Websites-Add.png)

Creamos un sitio web:

![](/assets/images/BoardLight/Websites-Create.png)

Y después le damos a la opción 'Edit HTML Source' (para añadir el código PHP):

![](/assets/images/BoardLight/Websites-Edit-Page.png)

![](/assets/images/BoardLight/Websites-Content-PHP-1.png)

No nos deja poner php en minúscula, hay que ponerlo en mayúscula:

![](/assets/images/BoardLight/Websites-Content-PHP-2.png)

Una vez añadido el código, nos dirigimos a la derecha, y saldrá un icono, le damos ahí para así poder guardar los cambios:

![](/assets/images/BoardLight/Websites-Edit-Save-Changes.png)

Ahora nos dirigimos al recurso que acabamos de crear con nuestra web shell, y ejecutamos un comando (id):

![](/assets/images/BoardLight/Web-Shell.png)

Vemos que funciona correctamente, ahora vamos a obtener una shell

![](/assets/images/BoardLight/Rev-Shell.png)

![](/assets/images/BoardLight/Rev-Shell-NC.png)

## Escalada de Privilegios (Larissa)

En la base de datos ubicada en: '/var/www/html/crm.board.htb/htdocs/conf/conf.php', nos encontramos con credenciales del usuario 'Larissa' (Es este usuario porque es el único que se encuentra en el directorio '/home'):

![](/assets/images/BoardLight/Larissa-Credentials.png)

Migramos al usuario 'Larissa':

![](/assets/images/BoardLight/Su-Larissa.png)

La bandera 'User' la encontramos en el directorio 'home' del usuario 'Larissa':

![](/assets/images/BoardLight/User-Flag.png)

## Escalada de Privilegios (Root)

Con la herramienta 'find', buscamos binarios/archivos con permisos 'SUID':

![](/assets/images/BoardLight/Find-SUID.png)

El binario 'enlightenment' puede ser vulnerable.

Miramos que versión tiene 'enlightenment'

![](/assets/images/BoardLight/Enlightenment-Version.png)

Buscamos en 'searchsploit' alguna vulnerabilidad:

![](/assets/images/BoardLight/Searchsploit-Enlightenment.png)

Vemos que hay una que nos proporciona escalada de privilegios.

Nos traemos el exploit:

![](/assets/images/BoardLight/Exploit.png)

Añadimos permisos de ejecución y lo ejecutamos:

![](/assets/images/BoardLight/Run-Exploit.png)

La bandera 'Root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/BoardLight/Root-Flag.png)

### Conclusión

BoardLight es un excelente ejemplo de cómo malas prácticas de configuración (como el uso de credenciales por defecto) y software desactualizado pueden abrir la puerta a intrusiones críticas. Este reto demuestra la importancia de realizar un hardening adecuado, mantener el software actualizado y auditar regularmente los permisos especiales como los SUID. La ruta de explotación fue clara y directa, lo que la convierte en una máquina ideal para reforzar conocimientos sobre reconocimiento web, explotación de CMS y escaladas locales con binarios vulnerables.
