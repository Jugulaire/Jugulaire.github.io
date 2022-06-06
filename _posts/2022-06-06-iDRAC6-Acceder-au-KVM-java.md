---
layout: post
title: iDRAC 6 Accéder au KVM java 
---

![idrac-login-web.png]({{ site.baseurl }}/images/idrac-login-web.png){:class="img-responsive"}

Le KVM des iDRAC 6 est clairement déprécié et repose sur du bon vieux java poussiéreux, il faut donc modifier les paramètres de java pour le faire fonctionner.

# Solution 1 (pas de média virtuel)

## Préparation des paramètres JAVA 

```bash
# Installer JRE
apt install default-jre
# Verifier la version
java -version
openjdk version "11.0.11" 2021-04-20
OpenJDK Runtime Environment (build 11.0.11+9-Ubuntu-0ubuntu2.20.04)
OpenJDK 64-Bit Server VM (build 11.0.11+9-Ubuntu-0ubuntu2.20.04, mixed mode, sharing)
# Chercher/Editer le fichier de conf en fonction de la version 
find / -name "java.security" 2> /dev/null
/usr/lib/jvm/java-11-openjdk-amd64/conf/security/java.security
/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/security/java.security
/etc/java-11-openjdk/security/java.security
/etc/java-8-openjdk/security/java.security
# Edition 
sudo vi /etc/java-11-openjdk/security/java.security
# Commenter les lignes suivantes 
jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL, \
```

## Téléchargement et lancement de la console 

```bash
# Téléchargement
wget https://192.168.2.53/software/avctKVM.jar
# Lancement 
java -cp Downloads/avctKVM.jar com.avocent.idrac.kvm.Main ip=[IP iDRAC] kmport=5900 vport=5900 user=root passwd=calvin apcp=1 version=2 vmprivilege=true "helpurl=https://[IP iDRAC]/help/contents.html"
```

# Solution 2 (avec icedtea netx)

## Préparation des paramètres JAVA 

```bash
# Installer JRE + icedtea
apt install default-jre icedtea-netx icedtea-plugin
# Verifier la version
java -version
openjdk version "11.0.11" 2021-04-20
OpenJDK Runtime Environment (build 11.0.11+9-Ubuntu-0ubuntu2.20.04)
OpenJDK 64-Bit Server VM (build 11.0.11+9-Ubuntu-0ubuntu2.20.04, mixed mode, sharing)
# Chercher/Editer le fichier de conf en fonction de la version 
find / -name "java.security" 2> /dev/null
/usr/lib/jvm/java-11-openjdk-amd64/conf/security/java.security
/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/security/java.security
/etc/java-11-openjdk/security/java.security
/etc/java-8-openjdk/security/java.security
# Edition 
sudo vi /etc/java-11-openjdk/security/java.security
# Commenter les lignes suivantes 
jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL, \
```

## Lancement de la console 

Télécharger le jnlp via la console web iDRAC et le lancer comme suit :

```bash
javaws [MONJNLP.jnlp]
```

> le nommage est anécdotique


# Astuces 

## Utiliser les média virtuels 

Dans la console web iDRAC, il faut activer l'utilisation des médias virtuels


## Reboot iDRAC 

Via SSH 

```bash
ssh root@192.168.2.52
/admin1-> racadm racreset
RAC reset operation initiated successfully. It may take up to a minute 
for the RAC to come back online again.
/admin1-> Connection to 192.168.2.52 closed by remote host.
Connection to 192.168.2.52 closed.
```

# Troubleshoot 

## Connexion failure même après avoir fais les modifications

Il faut se connecter en SSH pour faire un hard reset :

```
racadm racresetcfg
```

Une fois l'idrac réinitialisé, il prend l'ip fixe `192.168.0.120` il faut donc revoir l'adressage. 
