---
layout: post
title: L'attaque NTLM Relay
tags: [Mermaid]
mermaid: true
---

![relai]({{ site.baseurl }}/images/relais.jpg){:class="img-responsive"}

Suite à la finalisation de ma première attaque sur un domaine Active Directory utilisant ntlmrelay.py, je souhaitais en apprendre davantage sur le fonctionnement des attaques de type relais NTLM. 

Nous parlerons ici rapidement de LM, NTLM pour nous concentrer sur l'attaque par relais plus en détail.

## LM, vieux et vulnérable 

Il y a fort longtemps (80's) IBM et Microsoft travaillaient ensemble dans le développement du système OS/2, c'est à cette époque qu'est développé Server Message Block (SMB pour les intimes). 

Ce protocole permet entre autre de partager des fichiers mais manque de sécurité. Pour ajouter une couche de sécurité, un super protocole d'authentification a été développé du nom de Lan Manager. 

L'idée est de se baser sur un "challenge" pour permettre l'authentification mais présente de nombreuses failles dont voici une courte liste : 

- Basé sur l'algorithme de chiffrement DES
  - Réputé faible depuis les années 90
  - Souffre des failles du protocole (longueur de clé limitée)
- Seul les 14 premiers caractères sont pris en compte (même si le mot de passe en fait 40)
- Les mots de passes sont insensibles à la casse 

Il finit donc par être remplacé dans les années 90 par NTLM (et l'amélioration est à peine perceptible). 

## NTLM 

NTLM veut dire New Technology Lan Manager, c'est un protocole d'authentification disposant de 3 types de messages : 

- Type 1 : Négociation, échange des protocoles supportés entre serveur et client
- Type 2 : Challenge, la petite "énigme" à résoudre pour l'authentification
- Type 3 : Authentification, la réponse du client

> Dans la réalité c'est un poil plus complexe mais nous souhaitons concentrer notre attention sur NTLM en lui-même. 

### Histoire rapide

Il existe plusieurs versions de NTLM :

- NTLMV1 :
  - Repose sur les hash LM dont nous parlions avant 
  - Un nouveau challenge est introduit : 
    - Le serveur tire un nombre aléatoire sur 8 octets 
    - Le serveur réalise plusieurs chiffrements DES avec le hash LM de l'utilisateur et le stock
    - Le serveur envoie le même nombre aléatoire au client 
    - Le client effectue la même action de chiffrement avec le mot de passe saisie par l'utilisateur 
    - Le client envoie son résultat au serveur
    - Le serveur compare les deux, s'il correspondent c'est validé 
- NTLMV2 :
  - Repose cette fois sur des hash MD4 
  - Se base sur HMAC_MD5 pour son challenge avec deux nombres aléatoire

Comme nous le voyons de nombreuses vulnérabilités existent dans ces protocoles. De plus, les versions plus récentes de Windows embarquent une librairie du nom NTLMSSP permettant de négocier la version de NTLM utilisée dans un objectif de retro-compatibilité. 

Il est ainsi possible de réaliser des demandes de challenges faibles avec des empreintes LM pour attaquer ces protocoles facilement et s'authentifier avec n'importe quel utilisateur. 

### Principe de fonctionnement de base 

Voici une explication rapide des 3 étapes qui composent une authentification NTLM :

- Type 1 : Le client souhaite se connecter et envoie donc au serveur un paquet de négociation 

<div class="mermaid">
graph LR;
client--Demande connexion + protocoles supportés-->serveur;
</div>

- Type 2 : Le serveur envoie son challenge au client pour qu'il le résolve avec son mot de passe 

<div class="mermaid">
graph RL;
serveur--Voici ton challenge, résout-le avec ton MDP-->client;
</div>

- Type 3 : Le client résous le challenge et retourne le résultat au serveur pour validation 

<div class="mermaid">
graph LR;
client--Voici ma réponse-->serveur;
</div>

- Si le challenge est réussi, l'authentification est validée. Sinon elle est refusée.



### Le DNS, point de pivot de l'attaque

Avant de revenir à l'attaque NTLM Relay, parlons de la résolution DNS en environnement Windows. 

Quand vous cherchez à joindre un nom, voici (dans l'ordre) les outils utilisés pour le résoudre :

- Fichier Hosts 
- Cache ou requête DNS
- Le fichier LMHOST local (équivalent du fichier hosts avec NetBIOS)
- LLMNR (*Link-local Multicast Name Resolution*) 
- NBNS (NetBIOS Name Service) 

NBNS et LLMNR peuvent être usurpées avec l'outil responder et ainsi rediriger n'importe quel nom vers n'importe quelle IP. 

## Le relais NTLM

Le protocole WPAD (Web Proxy Auto-Discovery Protocol) peut être détourné pour récupérer des identifiants. Ce protocole permet de pousser des configurations de proxy vers des clients de manière dynamique mais ajoute des failles en contrepartie. 

Voici un exemple d'attaque avec la commande `responder -wFb -I eth0`: 

- Une machine essaie de joindre un nom `Bogus.domain.local` sur son navigateur :

  <div class="mermaid">
  graph LR;
  victime--DNS:bogus.domain.local-->Active-Directory;
  </div>
  
- Le serveur DNS de l'AD lui retourne `No such name` :

  <div class="mermaid">
  graph RL;
  Active-Directory--DNS:No such name-->victime;
  </div>


- Cette machine essaie donc d'utiliser LLMNR ou NB-NS (en broadcast):

  <div class="mermaid">
  graph LR;
  A[victime]--LLMNR:bogus.domain.local-->C[Active-Directory];
  A--LLMNR:bogus.domain.local-->B[Attaquant];
  </div>

- L'attaquant retourne son IP au client car ces requêtes sont faites en broadcast (toutes les machines du segment réseaux les reçoivent) :

  <div class="mermaid">
  graph RL;
  C[Attaquant]--LLMNR:192.168.3.5-->A[victime];
  B[Active-Directory];
  </div>

  

- Par le biais de WPAD, la machine télécharge le fichier `WPAD.dat` et l'attaquant retourne sa propre adresse en tant que proxy :

  <div class="mermaid">
  graph LR;
  A[victime]--WPAD:Donne moi WPAD.dat-->C[Attaquant];
  C--WPAD:le proxy est 192.168.3.5-->A;
  </div>

  > L'attaquant contrôle donc la page affiché et le proxy

- Une fois que le client à joint la page via le proxy , l’attaquant retourne une page forgée avec une ressource nécessitant une authentification  :

  <div class="mermaid">
  graph RL;
  A[victime]--affiche moi http://bogus.domain.local-->C[attaquant];
  C--attacker:666-->A;
  </div>

- Le client demande donc à l'utilisateur de saisir son mot de passe pour pouvoir accéder a ladite ressource. En même temps, l'attaquant tente une connexion vers un serveur du domaine :

  <div class="mermaid">
  graph LR;
  A[victime]--j'ai besoin de ton mot de pase-->C[Utilisateur];
  C --MonMotDePasse-->A;
  A--authentification smb://attacker:666-->B[Attaquant];
  B--je suis domain/client log moi-->D[serveur];
  </div>

- L'attaquant utilise les identifiants de la victime pour s'authentifier sur le serveur et relaie le challenge retourné par le serveur a la victime :

  <div class="mermaid">
  graph RL;
  a[attaquant]--résout ce hallenge-->b[victime];
  c[serveur]--résout ce challenge-->a;
  </div>
  
- L'attaquant transmet le challenge résolu par la victime au serveur sur lequel il souhaite s'authentifier et peut donc accéder à la ressource sans mot de passe : 

  <div class="mermaid">
  graph LR;
  a[victime]--voici mon challenge résolu-->b[attaquant];
  b--voici le challenge de la victime-->c[serveur];
  </div>

  Le serveur pense que l'attaquant est la victime et la victime pense que l'attaquant est le serveur. Tous deux ignorent la présence de l'attaquant qui vient de se servir d'eux pour s'authentifier sans mot de passe sur le serveur.  

> Note : ici les explications sont simplifiées et peuvent contenir quelques erreurs. J'ai notamment volontairement retiré l'échange entre le serveur et l'AD lors de l'authentification NTLM.

## La réalité 

De nombreux patch corrigent cette faille dans sa version théorique, néanmoins il existe de nouvelles alternatives permettant d'exploiter d'autres services tel qu'exchange. 

De plus de nombreux domaine Active Directory ne sont pas patché contre ces vulnérabilités ou continue d'exploiter des machines hors d'age (Windows server 2003 par exemple). Toutes ces failles potentielles font des domaines Active Directory un bon moyen pour un attaquant de s'infiltrer dans une infrastructure et d'y commettre des actes frauduleux tel que des vols de données ou des injection de malwares. 

La meilleure manière de prévenir ce genre d'attaque est d'utiliser le protocole Kerberos à la place de NTLM. Ce dernier offre de meilleurs sécurités que NTLM mais nous verrons certainement dans un autre billet qu'il présente également des failles tel que kerberoast. 

Il est également important de noter que ce genre d'attaque nécessite un pied dans vos infrastructures, il est donc important de mettre en place une stratégie solide en interne pour éliminer au maximum les failles d'accès à votre LAN. 

Je vous laisse un guide de bonne pratique : https://adsecurity.org/?p=1684
