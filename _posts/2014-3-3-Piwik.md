---
layout: post
title: Une alternative libre à Google Analytics avec Piwik !
---

![netdata.png]({{ site.baseurl }}/images/analytics.png){:class="img-responsive"}

Piwik est un logiciel libre et open source de mesure de statistiques web, successeur de PhpMyVisites et conçu pour être une alternative libre à Google Analytics. Piwik fonctionne sur des serveurs web PHP/MySQL.

## [0x100] Pré-requis :

1. Serveur web
  1. Apache
  2. Nginx
2. PHP 
3. Mysql

## [0x200] Installation :

Pour les besoins de ce tuto on part du principe que vous avez déja un serveur web et Mysql fonctionnel.

### [0x210] Mysql :
On va créer un utilisateur et une base pour Piwik :
```bash
sudo mysql -h localhost -p
```
```sql
CREATE USER 'piwik'@'localhost' IDENTIFIED BY 'Strong_Password';
CREATE DATABASE piwik;
GRANT ALL PRIVILEGES ON piwik. * TO 'piwik'@'localhost';
```
> Tips : On peut générer un mot de passe avec **pwgen** pour éviter de mettre "toto" (point de vue de sécurité).

### [0x220] Piwik :
On va télécharger l'archive et la décompresser dans **/var/www/html** :

```bash
wget https://builds.piwik.org/piwik.zip && unzip piwik.zip -d /var/www/html
```
### [0x230] Paramétrage Nginx :

```nginx
#PIWIK
        location /piwik {
                alias /var/www/html/piwik/;
                index index.php;

        }
        location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

```
### [0x240] Paramétrage de l'application :

Rendez vous a l'adresse **domaine.tld/piwik** pour débuter le paramétrage de Piwik. 
Vous pouvez maintenant profiter d'une solution analogue à Google Analytics sans tracking et totalement gratuit !

## [0x300] Prochaines étapes :

- Mettre en place le code Javascript de tracking dans votre site
- Commencer à mesurer les visites
