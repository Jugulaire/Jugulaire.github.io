---
layout: post
title: Installation de Netdata sur Debian
---

# Netdata Howto

## Installation


L'installation se déroule en tois étapes :

1. Installation des dépendances 
2. Installation de netdata
3. Création d'un service 

### 1 - Installation des dépendances 

```bash
curl -Ss 'https://raw.githubusercontent.com/firehol/netdata-demo-site/master/install-required-packages.sh' >/tmp/kickstart.sh && bash /tmp/kickstart.sh -i netdata-all
```
### 2 - Installation de netdata

```bash
# download it - the directory 'netdata' will be created
git clone https://github.com/firehol/netdata.git --depth=1
cd netdata

 # run script with root privileges to build, install, start netdata
 ./netdata-installer.sh
```
### 3 - Création de service

```bash
# stop netdata
killall netdata

# copy netdata.service to systemd
cp system/netdata.service /etc/systemd/system/

# let systemd know there is a new service
systemctl daemon-reload

# enable netdata at boot
systemctl enable netdata

# start netdata
service netdata start
```

> Le monitoring est maintenant disponible sur ```http://localhost:19999```


![netdata.png](../img/netdata.png)


**TODO** Script d'installation pour Debian

## Configuration des alertes par mail

Cette partie se déroule en plusieurs parties :

1. Installation et configuration de SSMTP
2. Configuration des alertes dans netdata

### 1 - 1 Installation de SSMTP
#### Installation (facile)
Sur Debian, un paquet existe dans les dépots :
```bash
apt install ssmtp
```
#### Configuration (pas vraiment plus compliqué)
On va simplement paramétrer un compte Gmail (mais n'importe quel serveur SMTP fera largement l'affaire).
Pour ce faire : 

```haskell
# Config file for sSMTP sendmail
# Fichier /etc/ssmtp/ssmtp.conf
#
# The person who gets all mail for userids < 1000
# Make this empty to disable rewriting.
root=yourmail@mail.com #<-- ICI on met son email 

# The place where the mail goes. The actual machine name is required no 
# MX records are consulted. Commonly mailhosts are named mail.domain.com
mailhub=smtp.gmail.com:587 #<-- ICI on met le serveur de mail SMTP de google .*[AVEC LE PORT]*.

# Where will the mail seem to come from?
rewriteDomain=

# The full hostname
hostname=yourserver.example.com #<-- ICI l'hostname de notre serveur

# Are users allowed to set their own From: address?
# YES - Allow the user to specify their own From: address
# NO - Use the system generated From: address
FromLineOverride=YES

# Username and password for Google's Gmail servers
# From addresses are settled by Mutt's rc file, so 
# with this setup one can still achieve multi-user SMTP
AuthUser=username@gmail.com #<-- On met son email ici aussi
AuthPass=password # Le password 

#other parameters for google.com 
UseTLS=YES
UseSTARTTLS=YES
AuthMethod=LOGIN

```
> Note: ce fichier contient ici notre identifiant en clair ainsi que notre mot de passe.
> Voici donc comment faire pour sécuriser un peut les choses (l'idéale étant de créer un compte mail spécifiquement pour nos alertes)

```bash
 chown root:mail /etc/ssmtp/ssmtp.conf
 chmod 640 /etc/ssmtp/ssmtp.conf
 usermod -a -G mail netdata
```
Testons maintenant que tout fonctionne :

```bash
user@yourmachine ~ $ echo "corp de mail" | mail -s "testing ssmtp setup" une.adresse@unDomaine.com
```
Place au paramétrage de netdata :

```bash 
vi /etc/netdata/health_alarm_notify.conf
```
Rendez-vous ligne 91 (```:91``` dans vi)
```ruby
#------------------------------------------------------------------------------
 # email global notification options
 
 # multiple recipients can be given like this:
 #              "admin1@example.com admin2@example.com ..."
 
 # enable/disable sending emails
 SEND_EMAIL="YES"
  
 # if a role recipient is not configured, an email will be send to:
 DEFAULT_RECIPIENT_EMAIL="une.adresse@unDomaine.com"
 # to receive only critical alarms, set it to "root|critical"

```
On modifie Revaliases :

```ruby
# /etc/ssmtp/revaliases

# Format:   local_account:outgoing_address:mailhub
# Example: root:your_login@your.domain:mailhub.your.domain[:port]
# where [:port] is an optional port number that defaults to 25.
netdata:contact@smartfizz.fr:mail.bla.net:587
~                                                 
```

On test : 

```bash
sudo su -s /bin/bash netdata
/usr/libexec/netdata/plugins.d/alarm-notify.sh test
```


## Mise en place d'un proxy :

> Note: ici on utilise Nginx mais on peut faire la même chose avec Apache.

Voici une configuration permettant d'accéder a Netdata via un sous dossier (Mondomaine.com/netdata):

```nginx
#Monitoring
	location /netdata {
		rewrite /netdata(.*) /$1 break;
		proxy_redirect off;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Server $host;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_http_version 1.1;
		proxy_pass_request_headers on;
		proxy_set_header Connection "keep-alive";
		proxy_store off;
		proxy_pass http://172.18.0.1:19999;
		gzip on;
		gzip_proxied any;
		gzip_types *;
	}
}

```
