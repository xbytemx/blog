---
title: "HTB Writeup: Blackfield"
date: 2020-10-03T11:45:41-05:00
draft: true
toc: false
images:
tags: 
  - hackthebox
  - pentesting
  - boot2root
---



<!--more-->

# Machine info
La información que tenemos de la máquina es:

| Name | Maker | OS  | IP Address |
| ---- | ----- | --- | ---------- |
| ![blackfield image](https://www.hackthebox.eu/storage/avatars/7c69c876f496cd729a077277757d219d_thumb.png) [blackfield](https://www.hackthebox.eu/home/machines/profile/255) | [aas](https://www.hackthebox.eu/home/users/profile/6259) | Windows | 10.10.10.192 |

Su tarjeta de presentación es:

![cardinfo](/img/htb-blackfield/cardinfo.png)

# Port Scanning and Service Discovery

Iniciamos por ejecutar un `masscan` para descubrir puertos udp y tcp abiertos, y posteriormente `nmap`, para identificar servicios expuestos en estos puertos.

## masscan

```
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.192 | tee /home/tony/htb/blackfield/masscan.log
Discovered open port 389/tcp on 10.10.10.192
Discovered open port 3268/tcp on 10.10.10.192
Discovered open port 445/tcp on 10.10.10.192
Discovered open port 53/udp on 10.10.10.192
Discovered open port 5985/tcp on 10.10.10.192
Discovered open port 135/tcp on 10.10.10.192
Discovered open port 593/tcp on 10.10.10.192
Discovered open port 88/tcp on 10.10.10.192
Discovered open port 53/tcp on 10.10.10.192
```

- `-e tun0` para ejecutarlo nada mas en la interface tun0
- `-p0-65535,U:0-65535` TODOS los puertos (TCP y UDP)
- `--rate 500` para mandar 500pps y no sobre cargar la VPN ):

Descubrimos 9 puertos abiertos; 53(DNS UDP y TCP), 88(Kerberos), 135(RPC), 445(SMB), 593(RPCoHTTP), 389,3268(LDAP), 5985(WinRM). Ahora con `nmap`, identifiquemos si servicios por defecto se encuentran en estos puertos.

## nmap services

```
# Nmap 7.80 scan initiated Thu Jun 11 01:10:41 2020 as: nmap -sS -sV -sC -n -v -p 389,3268,445,5985,135,593,88,53 -oA /home/tony/htb/blackfield/normal 10.10.10.192
Nmap scan report for 10.10.10.192
Host is up (0.10s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-06-11 13:15:04Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=6/11%Time=5EE1CAEC%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h04m14s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2020-06-11T13:17:23
|_  start_date: N/A

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Jun 11 01:13:47 2020 -- 1 IP address (1 host up) scanned in 187.03 seconds
```

- `-sS` para seleccionar el escaneo TCP vía SYN
- `-sC` para que ejecute los scripts safe-discovery de nse
- `-sV` para que me traiga el banner del puerto
- `-p 389,3268,445,5985,135,593,88,53` para escanear solo los puertos TCP 80 y 22
- `-oN` para guardar el output en formato normal o salida por defecto de nmap
- `-n` para no ejecutar resoluciones
- `-v` para modo *verboso*

Como podemos ver por la salida de nmap, tenemos el dominio de la maquina `blackfield.local`, algunos servicios que permiten el acceso con cuentas sin contraseña y algunos otros servicios que una vez con credenciales, nos pueden dar acceso remoto al servidor. Trabajemos ahora con esos servicios tales como SMB y RPC.

# Enumeración

## SMB

Ejecutamos una conexión anonima sobre el protocolo SMB, esto con la intencion de desplegar todo el contenido que podamos del servidor, en mi caso usare `smbclient.py` de impacket, pero podemos usar `smbclient` nativo o `smbmap`. Una vez conectado sacamos informacion sobre los recursos compartidos con el comando shares.

```cmd
xbytemx@laptop:~/htb/blackfield$ smbclient.py anonymous:@10.10.10.192
Impacket v0.9.22-dev - Copyright 2019 SecureAuth Corporation

Password:
Type help for list of commands
# shares
ADMIN$
C$
forensic
IPC$
NETLOGON
profiles$
SYSVOL
```

Como podemos ver este servidor comparte algunos recursos de manera anonima, tanto algunos que son por defecto tales como `SYSVOL` y `NETLOGON`, como otros que son de mayor interes como `forensic` y `profiles$`. Veamos el contenido de `profiles$` accediendo al recurso y listando su contenido:

```cmd
# use profiles$
# ls
drw-rw-rw-          0  Wed Jun  3 11:47:12 2020 .
drw-rw-rw-          0  Wed Jun  3 11:47:12 2020 ..
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 AAlleni
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 ABarteski
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 ABekesz
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 ABenzies
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 ABiemiller
drw-rw-rw-          0  Wed Jun  3 11:47:11 2020 AChampken
...
```



```cmd
xbytemx@laptop:~/htb/blackfield$ smbclient -c 'ls' -N '//10.10.10.192/profiles$' | awk '{print $1}' | grep -vE '^$|[0-9]|\.' | tee list1.txt
AAlleni
ABarteski
ABekesz
ABenzies
ABiemiller
AChampken
ACheretei
ACsonaki
```

Despues de intentar con crackmapexec la lista de nombres como usuario y contraseña, 

```
```

# user.txt
## Privilege Escalation
# root.txt
