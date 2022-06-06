---
layout: post
title: Le PCI passtrough  KVM, Optimisation
tags: [Mermaid]
mermaid: true
---

![gauge_kvm]({{ site.baseurl }}/images/gauge_kvm.jpg){:class="img-responsive"}



Lors d'un précédent billet j'avais évoqué l'installation d'une VM en mode PCI passtrough. Néanmoins, il manque une partie assez importante à détailler ; l'optimisation. 

Dans ce billet, je vais donc détailler les différentes techniques d'optimisation que j'ai mise en œuvre sur mon passtrough. 

## Le CPU pinning 

### Explication 

> Attention, les explications données ici peuvent être erroné. Elle regroupe les recherches que j'ai effectuées sur le sujet. 

Les CPU modernes (ou plutôt relativement modernes) offrent un technologie appelé SMT (Simultaneous Multi Threading). Le principe est simple, utiliser un coeur CPU pour gérer deux thread en passant d'un thread à l'autre de maniéré plus ou moins efficiente. 

Pour faire simple, c'est comme si vous tentiez de manger un paquet de chips le plus rapidement possible. Les chips représentent les tâches, vos mains les threads et votre bouche représente le cœur CPU. Ajouté un cœur serait donc similaire à vous greffer une bouche, ajouter un thread (faire du SMT donc) reviendrait à utiliser vos deux mains pour amener les chips a votre bouche. Le SMT c'est donc une optimisation de l'acheminement des données au cœur, plus qu'un ajout simple de performance brute. 

Alors tout ça c'est cool, mais pas pour notre machine virtuelle car KVM créé un processus par CPU virtuel sur notre machine hôte. Et donc ces processus sont soumis aux mêmes règles que les autres dans l’ordonnancement des tâches. 

Un ordonnanceur de tâche définis quel tâche pourra accéder au cœurs CPU mais également sur quel cœur CPU elle sera exécuté. Il tiendra pour cela compte de l'utilisation du cœur et de l'importance du processus.

L’ordonnanceur de tâche c'est en gros comme une queue à la caisse d'un super-marché, vous avez des personnes qui représentent les tâches et les caisses qui représente les cœurs. Vous avez des caisses moins de 10 articles, d'autre réservés aux personnes a mobilité réduite ou encore des caisses classiques. L'objectif est de répartir chaque personne en fonction de critères donné juste avant. En tant que client, si une file semble plus courte vous allez passer d'une caisse à l'autre. Mais imaginez que cette caisse soit réservée aux personne à mobilité réduite, si une personne en fauteuil roulant arrive, vous devez lui céder la place (et vous allez perdre du temps). 

Pour notre machine virtuelle c'est la même chose, chacun des processus de nos cœurs virtuels doivent se battre avec ceux de la machine hôte pour accéder au cœurs CPU physiques. Il y a alors de l'attente et donc des latences d'accès car d’autre processus prioritaires passent en premiers dans l’ordonnancement des tâches. 

Ces latences sont encore aggravées de par l'architecture des CPU. Généralement, un CPU multi-cœur possède des caches L1/L2 par cœurs mais partagent le cache L3. Cela signifie que si l'un de nos processus passe du cœur physique 1 vers le cœur physique 3, il perdra par la même occasion son cache L1/L2. Rappelons que ces caches existent pour optimiser les performances d’exécution des tâches. 

### Application

Pour éviter la perte de performances, nous allons assigner des cœurs physiques de notre hôte a notre machine virtuelle en prenant en compte la topologie du CPU. Dans mon cas, je dispose d'un `i7-4790K` disposant de 4 cœurs et 8 thread. Cela veut donc dire que chaque cœur possède 2 thread. Voyons donc quel cœur fonctionne avec quel thread : 

```bash
jugu@WORKSTATION:~$ cat /sys/devices/system/cpu/cpu*/topology/thread_siblings_list | sort | uniq
0,4
1,5
2,6
3,7
```

Voici donc ce qu'il faut ajouter dans la configuration de ma machine : 

```xml
<vcpu placement='static'>4</vcpu>
<cputune>
    <vcpupin vcpu='0' cpuset='2'/>
    <vcpupin vcpu='1' cpuset='6'/>
    <vcpupin vcpu='2' cpuset='1'/>
    <vcpupin vcpu='3' cpuset='5'/>
	<emulatorpin cpuset='0-4'/>
</cputune>
```

La partie `vcpupin` permet d'assigner un cœur a notre VM. La partie `emulatorpin` permet d'assigner des cœurs CPU a l'emulation. Les deux sont séparé pour répartir la charge sur tous les cœurs physiques. 

Bien entendus, chaque stratégie de pinning offre des avantages et des inconvénients il faut donc l'adapter aux besoins. 

> Je vous laisse faire un tour sur les benchmarks en fonction des stratégies de pinning a [ce lien](https://passthroughpo.st/cpu-pinning-benchmarks/)
>
> Et la source également a [ce lien](https://www.freesoftwareservers.com/display/FREES/CPU+Pinning+on+KVM+for+Windows+10+Gaming+Guest)

## Optimiser Windows

Un autre gros morceau est l'optimisation se situe dans Windows en lui-même. Dans mon cas, ces optimisations permettent surtout d'augmenter la fiabilité de votre OS virtualisé. Voyons voir de quoi il s'agit :

### Corriger l'erreur 127 sur les GPU AMD 

#### L'histoire 

Tous les possesseurs de GPU AMD connaissent cette erreur. Elle survient de manière aléatoire dans mon cas mais beaucoup rapportent une récurrence dans le comportement rencontré. L'erreur 127 est levée par KVM quand le matériel PCI ne parvient pas à se réinitialiser. 

Un GPU est une carte indépendante du reste de l'ordinateur, elle comporte son étage d'alimentation, son BIOS et bien entendus son GPU. Lors de l'extinction de votre PC, l'OS coupe tous les services et les programmes qui tournent sur votre machine, il demande ensuite l’arrêt du matériel et la coupure d'alimentation. Néanmoins, dans le cas des GPU AMD, Windows n'envoie aucun signale de coupure au GPU. Ce dernier reste donc en état actif. Sur une machine physique cela fonctionne bien car l'alimentation est systématiquement coupé. 

Néanmoins, dans le cas d'une machine virtuelle, une fois la VM arrêté le GPU reste alimenté dans un état initialisé. Lors du redémarrage de la VM, l'erreur 127 est levée car le GPU est déjà initialisé et donc potentiellement utilisé par une autre VM/ressource. 

#### L'application

Il existe néanmoins une solution se basant sur l'outil Windows Device Console utilisé dans le développement de drivers pour envoyé des signaux aux périphériques. 

Nous allons ici utilisé chocolatey pour installer cet outil, collez simplement ceci dans une console powershell administrateur pour l'installer : 

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

Maintenant je vais installer l'outil : 

```powershell
choco install devcon.portable
```

Cet outil est très simple d'utilisation : 

```powershell
devcon64.exe enable|disable "<device_id>"
```

Il nous faut maintenant le device ID disponible dans le gestionnaire de péripheriques. Faites un clic droit sur la carte graphique, propriétés, details :

![127_err_gpu]({{ site.baseurl }}/images/127_err_gpu.png){:class="img-responsive"}

Faire de même pour la partie audio :

![127_err_gpu-2]({{ site.baseurl }}/images/127_err_gpu-2.png){:class="img-responsive"}

Je crée ensuite deux scripts batch pour activer le GPU :

```bash
devcon64.exe enable "PCI\VEN_1002&DEV_67EF*"

devcon64.exe enable "HDAUDIO\FUNC_01&VEN_1002&DEV_AA01*"
```

Pour désactiver le GPU :

```bash
devcon64.exe disable "PCI\VEN_1002&DEV_67EF*"

devcon64.exe disable "HDAUDIO\FUNC_01&VEN_1002&DEV_AA01*"
```

Maintenant nous allons déployer une GPO locale pour lancer nos scripts au démarrage et à l’arrêt :

```bash
gpedit.msc
```

Puis ajouter le script d'activation au démarrage et celui de désactivation a l'arrêt : 

![127_err_gpu-3]({{ site.baseurl }}/images/127_err_gpu-3.png){:class="img-responsive"}

C'est terminé ! 

> Sources : [Level1](https://forum.level1techs.com/t/linux-host-windows-guest-gpu-passthrough-reinitialization-fix/121097)

### Désactiver fast startup et sleep mode :

Ces fonctionnalités peuvent rendre votre OS instable voire même faire planter l'hôte. Voici un one-liner pour le couper : 

```bash
REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Power" /V HiberbootEnabled /T REG_dWORD /D 0 /F
powercfg.exe /h off 
```

### Activer les optimisations Hyper-V

Bien qu’étonnante, cette fonctionnalité permet de profiter des optimisations d'hyper-v sur KVM :

```bash
sudo virt-xml $VMNAME --edit --features hyperv_relaxed=on,hyperv_vapic=on,hyperv_spinlocks=on,hyperv_spinlocks_retries=8191
sudo virt-xml $VMNAME --edit --clock hypervclock_present=yes 
```

## Conclusion 

Comme vous le voyez, mettre en place et optimiser une machine virtuelle avec du GPU passtrough n'est pas forcément simple. Néanmoins, j'ai appris énormément sur le fonctionnement des différentes parties d'un ordinateur. Il existe bien entendus d'autres techniques pour optimiser les performances comme l'isolation CPU qui permet de demander au kernel de ne pas utiliser du tout certains CPU pour les dédier a notre VM. Mais, je souhaite profiter des performances de ma workstation Linux pleinement quand je n'exploite pas ma VM c'est pourquoi j'ai décidé de ne pas utiliser l'option isolcpu. 

>  J'ai gagné autour de 25 à 30 fps en jeu avec ces optimisations. 

En espérant avoir aidé certaines personnes. 
