---
layout: post
title: Pourquoi utiliser AWK ?  
---

![awk]({{ site.baseurl }}/images/awk.png){:class="img-responsive"}

J'ai longtemps été réticent vis a vis de l'apprentissage de AWK. 
La syntaxe me semblait indigeste et puis il est possible de faire la même chose avec sed, grep, cut ou encore tr alors a quoi bon ? 

Ce n'est que récemment que je me suis penché sur cet outil et depuis, il remplace tous les autres. 
AWK sait tout faire, traiter des CSV ou en créer. Traiter des logs.. Et j'en passe. 

# Le workflow 

Avant de parler de AWK laissez moi vous donner des astuces sur l'utilisation de tee et de sed. 
Tout deux sont essentiels dans l'industrialisation et le traitement de vos sortie de commande. 

## tee

Utiliser un tee pour permettre de faire un fichier de log tout en affichant le retour de la commande en console :

```bash
manspider --threads 256 ip.txt -s 9999mo -m 99 -u jugulaire -p SuperSekurePassword -f '(ntds.*|SYSTEM|system)' | tee Log_spider.log
```

> tee -a pour faire un append

## sed 

Une fois les logs obtenus, faire un nettoyage des caractère de colorisation bash car ils empêchent le bon déroulement des regex et donc l'utilisation de AWK.

```bash
sed -i 's/\x1B\[[0-9;]\{1,\}[A-Za-z]//g' Logs_files
```

SED permets aussi de faire du filtrage via des pipe. Pour cela il suffit de retirer le -i et de faire comme cela : 

```bash
cat Logs_files | sed  's/\x1B\[[0-9;]\{1,\}[A-Za-z]//g' 
```

La syntaxe est identique à celle de VI et ED avant lui : 

```bash
sed 's/pattern/remplacement/g' monfichier
```

Il est donc possible de faire n'importe quelle opération de remplacement avec SED.

> -i remplace le contenus du fichier. Retirez-le pour tester vos remplacements.

## AWK

Place au sujet principal de ce billet.

### Les I/O 

AWK est capable d'ingérer de la data de 3 manière : 
- Interactive si l'on ne spécifie pas de fichier a AWK : `awk '{print $1}'`
- Via un pipe : `cat text.txt | awk '{print $1}'`
- En spécifiant une ou plusieurs sources : `awk '{print $1}' monfichier1 monfichier2`

### Les séparateurs 

Une colonne est délimitée par un séparateur qui est par défaut la tabulation ou une série d'espaces.
Pour en faciliter la gestion et éviter les comportements inattendus, nous le spécifierons à l'aide de `-F` suivis de la séquence de séparation. 
Par exemple : 

```bash
awk -F '\t' '{print $1}'
```

`\t` représente une tabulation mais, tout autre caractère est valable comme `,`, ` ` ou encore `;`. 

### Les colonnes 

Pour accéder aux colonnes, utiliser les signes `$` : 
- `$0` : ligne complete
- `$1` : colonne 1
- `$666` : colonne 666

### Le format d'un programme AWK 

Un programme AWK se découpe en 3 partie : 
- `BEGIN {}` : Code exécuté avant le parcours des lignes
- `{}` : Code exécuté sur les lignes
- `END {}` : Code exécuté a la fin du traitement des lignes

### Programme de base 

Voici un ligne de code AWK basique dont le but est d'afficher les colonnes 1 et 3 d'un fichier texte dont le séparateur est une tabulation : 

```bash
awk -F '\t' 'BEGIN {print "programme de test"}{print $1 " -- >" $3}' test.txt
```

Voici des données pour tester : 

```bash
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
```

L'execution du code nou donne ceci : 

```bash
programme de test
toto-->titi
toto-->titi
toto-->titi
toto-->titi
toto-->titi
toto-->titi
toto-->titi
toto-->titi
toto-->titi
```
BEGIN est exécuté avant le traitement des lignes et affiche "programme de test"
La suite concatène toto et titi avec une fleche. 

On pourrait tout a fait utiliser le BEGIN pour placer une entête CSV. 

### Les tests simples 

L'une des force de AWK c'est sa capacité a faire des tests sur les lignes. 
Il est par exemple possible de tester si une ligne contient une chaîne précise avant de l'afficher : 

```bash
awk -F '\t' '{if($1=="toto"){print $0}}' test.txt
```

Voici le jeu de données : 

```
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
totto   tutu    titi    tata
totto   tutu    titi    tata
toto    tutu    titi    tata
totto   tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
```

Nous allons ici retirer toutes les lignes dont la premiere colonne ne contiens pas "toto" : 

```bash
jugu@LBPLT-egJoTI9Er:~$ awk -F '\t' '{if($1=="toto"){print $0}}' test.txt
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
```

Ici `==` veut dire "est égale à" mais son contraire existe aussi avec `!=`.

### Compter le nombre de lignes 

Ici, nous allons reprendre le même schéma mais compter les lignes qui correspondent : 

```bash
awk -F '\t' 'BEGIN{i=0}{if($1=="toto"){print $0;i++}}END{print "Nombre de valeurs: " i}' test.txt
```
Ce qui donne : 

```bash
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
Nombre de valeurs: 6
```
Ici j'ai instancié une variable i dans le BEGIN. Je l'incrémente après le print dans le corp du programme puis je l'affiche dans le END.

### Les regex 

La vraie puissance de AWK c'est sa capacité a utiliser des regex pour faire ses tests.
Voici la syntaxe pour faire comme l'exemple précédent : 

```bash
awk -F '\t' '{if($1 !~/.*tto/){print $0}}' test.txt
```
Cette fois, on souhaite ne pas afficher les les lignes dont la premiere colonne ne contiens **pas** "tto" avec `!~/.*tto/`.

> Comprenez ici que `~/maregex/` est utiliser pour matcher une regex et `~!/maregex/` pour NE PAS matcher une regex

On obtiens donc le résultat suivant : 

```bash
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
toto    tutu    titi    tata
```

Bien entendus il est possible de faire le contraire en retirant le `!` pour obtenir toutes les lignes dont la premiere colonne contiens "toto" : `~/toto/`

### Les regex avec groupes (gawk)

Imaginons que l'on souhaite faire un filtrage via une regex pour ensuite exploiter les groupes de ladite regex.

> Les groupes dans une regex sont des moyens très puissant de récupérer des valeurs dans un fichier texte. Elle se matérialisent par des parentheses `(.*groupe1)(.*groupe2)`

Prenons les données suivantes issue d'un nslookup imaginaire : 

```bash
Non-authoritative answer:
Name:   H4x0r.local
Address: 120.215.275.145
Name:   H4x0r.local
Address: 147.203.156.180
Name:   H4x0r.local
Address: 120.143.180.144
```

On souhaite ici lister les IP uniquement via une regex `(([0-9]{1,3}\.){3}[0-9]{1,3})` en utilisant le premier groupe de recherche : 

> ici la regex comporte 2 groupes
> Un groupe globale pour toute suite de caractère représentant une ip
> Un sous groupe pour matcher les 3 premiers octets
> voyez le sous groupe comme une solution pour raccourcir la regex
> N'hésitez pas a coller la regex dans regex101 pour l'analyser 

```bash
gawk 'match($0,/(([0-9]{1,3}\.){3}[0-9]{1,3})/,a){print a[1]}' test.txt
```

Le format est le suivant : `match(FIELD,/REGEX/,TABLEAU)` 
- FIELD représente $0 ou $1 ou encore $999 selon le numéro de la colonne 
- /REGEX/ représente la regex contenus entre `/`
- TABLEAU représente le tableau qui va recevoir le résultat de nos groupes

Le tableau retourné par la fonction match permets d'accéder aux résultat des différents groupes de la manière suivante : 
- a[1] groupe 1 
- a[2] groupe 2
- a[44] groupe 44 
- etc..

Ce qui donne ceci dans notre cas: 

```bash
220.215.175.145
147.203.156.180
210.113.180.144
```

Pour réaliser les regex je vous recommande de prendre un jeu de données et de les placer dans le site regex101 afin de tester et de valider vos regex ainsi que vos groupes. 
Cet exemple est simple mais montre la puissance de gawk avec les groupes. Il est en effet possible d'aller bien plus loins dans le filtrage sur des lignes bien plus complexes.

### Les regex avec groupe pour faire du CSV

Imaginons les logs suivants : 

```bash
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\WindowsColorSystem.adml (1.63KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\SystemResourceManager.adml (7.01KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\SystemResourceManager.adml (7.01KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemResourceManager.admx (2.77KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\SystemRestore.adml (3.51KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\SystemRestore.adml (3.51KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemRestore.admx (1.68KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\WindowsColorSystem.adml (1.63KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\fr-FR\WindowsColorSystem.adml (1.63KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\WindowsColorSystem.admx (1.98KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemResourceManager.admx (2.77KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemResourceManager.admx (2.77KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemRestore.admx (1.68KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\SystemRestore.admx (1.68KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\WindowsColorSystem.admx (1.98KB)
[+] 222.34.245.202: SYSVOL\toto.local\Policies\testTest\WindowsColorSystem.admx (1.98KB)
```

Je souhaite ici faire un CSV avec les IP, les chemins et pour finir la taille des fichiers retourné par le scan. 
Pour cela j'ai réalisé une regex dans regex101 avec 3 groupes : 

```bash
\[\+\] (([0-9]{1,3}\.){3}[0-9]{1,3}): (.+)\(([0-9]{1,3}\.[0-9]{0,3}(K|G|M)?B)
```
> attention aux yeux

Je la place ensuite dans une programme awk dont le BEGIN me fabrique une entête CSV : 

```bash
gawk 'BEGIN{print "ip,partage,taille"}match($0,/\[\+\] (([0-9]{1,3}\.){3}[0-9]{1,3}): (.+)\(([0-9]{1,3}\.[0-9]{0,3}(K|G|M)?B)/,a){print a[1]","a[3]","a[4]}' test2.txt
```

Décomposons-le : 

```bash
# Je débute par un simple print d'entête CSV
BEGIN{print "ip,partage,taille"}
# Je match sur tout ma ligne ($0) les données dont j'ai besoin
match($0,/\[\+\] (([0-9]{1,3}\.){3}[0-9]{1,3}): (.+)\(([0-9]{1,3}\.[0-9]{0,3}(K|G|M)?B)/,a)

# J'imprime le résultat si et seulement si une ligne match avec ma regex
{print a[1]","a[3]","a[4]}
```

Et j'obtiens ceci : 

```bash
ip,partage,taille
222.34.245.200,SYSVOL\toto.local\Policies\testTest\fr-FR\WindowsColorSystem.adml ,1.63KB
222.34.245.200,SYSVOL\toto.local\Policies\testTest\fr-FR\SystemResourceManager.adml ,7.01KB
222.34.245.200,SYSVOL\toto.local\Policies\testTest\fr-FR\SystemResourceManager.adml ,7.01KB
```

### awk et bash : le choc des titans

Dans cet exemple je voudrais comparer deux liste. Si les elements présent dans la premiere liste existe dans la deuxième, alors je les affiche : 

```bash
#!/bin/bash
echo "sAMAccountName,SID" # entête csv
while read name 
do
        gawk -F '\t' '{if($0 ~/'"$name"'.*/){print  $3 "," $11}}' dump_user.grep dump_user_2.grep
done < user_fragiles.lst
```

Dans le cas présent j'ai une liste d'utilisateurs `user.lst` et je voudrais savoir s'ils existent dans les dumps `dump_user.grep` et `dump_user_2.grep`.

Deux problématiques se pose donc ici : 
- Comment passer une variable bash a AWK ? 
- Comment boucler proprement ?

Pour passer une variable bash a AWK, je vais simplement échapper du bash en encadrant ma variable avec `'"$mavariableBASH"'` 
Pour la boucle, j'aurais pu utiliser un for mais une interpretation hasardeuses des espaces le rends imprévisible. Mieux vaut donc utiliser un while. 