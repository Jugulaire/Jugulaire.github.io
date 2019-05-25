---
layout: post
title: J'ai testé le PCI passtrough avec Qemu et KVM  1/2
---

![netdata.png]({{ site.baseurl }}/images/kvm.png){:class="img-responsive"}

## Présentations 

Avant de me lancer dans une analyse plus technique du sujet, c'est quoi le PCI passtrough ? 

### C'est quoi 

La virtualisation n'est pas nouvelle, elle permet de faire  n'importe quoi avec un ordinateur hôte qui accueil des machines virtuelles. Jusqu’à'a là rien de bien nouveau sous le soleil me direz-vous.
Il faut savoir que pour améliorer les performances des machines virtuelles, nos deux fabricants de CPU préférés ont développé des technologies tel que le VT-d ou encore AMD-vi. Elles permettent de déléguer une partie de la gestion des machines virtuelles directement dans le CPU et d'offrir un accès direct au matériel de l'hôte. 
Avec ces technologies sont arrivés des possibilités bien plus intéressantes, le PCI passtrough en fait partie en permettant de présenter un périphérique PCI à une machine virtuelle. 
Cette technique permet donc d'offrir des performances encore meilleures aux machines virtuelles en leur permettant par exemple de bénéficier de performances en calcul GPU. 
Nous sommes donc capables de lancer une petit GTA dans une machine virtuelle ou encore se dédier une machine virtuelle a des outils tel que Hashcat (pour casser des mots de passe). 

## Prérequis 

Primo, de quoi nous avons besoin pour faire tourner ce setup ?

### Hardware

- Deux cartes graphiques (une PCI-e et celle intégrée dans votre CPU ou deux PCI-e) 
- Deux moniteurs 
- Un SSD dédié (ou beaucoup d'espace disque) 
- De la RAM (24 Go dans mon cas)

> Note : Deux cartes avec des GPU différents (AMD et NVIDIA par exemple) pour simplifier les manipulations.. 

### Software 

- Un ISO de Windows 10
- Looking Glass ou Barrier (équivalent FOSS de synergy)
- Un Ubuntu 18.04 tout propre 
- KVM installe avec virt-manager 

  - ```
    sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf
    ```

> Note : nous travaillons ici sur une version 18.04 de Ubuntu

### Misc

- Une bonne dose de patience 
- Du café 

### Mon matos 

- Asus Maximus V (c'est une carte mère)
- i5-4690K @ 3.50GHz (Fréquence de base), Remplacé par un i7 4790k 
- 24 Go de DDR3, Remplacé par 32 Go
- 2 SSD de 256 Go (1 pour l’hôte, l'autre pour la VM)
- 1 HDD de 1 To
- Une GTX 770 (GPU principale)
- Une RX 460 (pour le VM)

## Place au sport : le paramétrage de base

Nous allons ici suivre plusieurs étapes : 

- Configurer la carte mère pour la virtualisation et vérifier sa bonne prise en charge par l'OS
- Trouver les ID de la carte graphique que l'on va assigner a la VM
- Configurer Linux pour la prise en charge de l'IOMMU
- Configurer Linux pour la prise en charge du VFIO 
- Configurer Linux pour qu'il ne charge pas de driver sur la carte graphique réserve a notre VM

### BIOS/UEFI

Première étape, faire un tour dans le BIOS/UEFI pour activer les choses suivantes :

- VT-D (cas d'un CPU intel)
- AMD-Vi (cas d'un CPU AMD)

Pour vérifier : 

```bash
jugu@WORKSTATION:~$ dmesg | grep -e "Directed I/O"
[    0.532200] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

Pour du AMD  :

```bash
dmesg | grep AMD-Vi
```

Avec un retour qui ressemble a ceci :

```bash
AMD-Vi: Enabling IOMMU at 0000:00:00.2 cap 0x40
AMD-Vi: Lazy IO/TLB flushing enabled
AMD-Vi: Initialized for Passthrough Mode
```

> Note : Vous venez d'activer le fameux IOMMU sur la carte mère.

### Où est tu jolie RX460 ? 

Grace a `lspci` nous allons trouver toutes les cartes graphiques installée sur la machine :

```bash
jugu@WORKSTATION:~$ lspci | grep VGA
01:00.0 VGA compatible controller: NVIDIA Corporation GK104 [GeForce GTX 770] (rev a1)
02:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Baffin [Radeon RX 460/560D / Pro 450/455/460/555/560] (rev cf)
```

Mais c'est pas tout, une carte graphique contient aussi une partie audio (pour l'HDMI). Affinons donc la recherche : 

```bash
jugu@WORKSTATION:~$ lspci -nn | grep 02:00.
02:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin [Radeon RX 460/560D / Pro 450/455/460/555/560] [1002:67ef] (rev cf)
02:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:aae0]
```

> Note : Remplacez le 02:00 par l'ID de votre carte graphique

Maintenant, notons les identifiants entre crochet : 

- [1002:67ef]
- [1002:aae0]

### Ne pas charger de driver sur la carte réserve a la VM

Toute la subtilité du processus, c'est de ne pas donner le contrôle du matériel que l'on souhaite présenter a notre VM a la machine hôte. 

> Note : En réalité pas vraiment, car le driver utilisé est VFIO pour faire le "passe-plat" entre l'hôte et la VM

Nous allons donc ajouter les drivers `radeon` et `amdgpu` a la fin du fichier `/etc/modprobe.d/blacklist.conf` : 

```bash
...
# EDAC driver for amd76x clashes with the agp driver preventing the aperture
# from being initialised (Ubuntu: #297750). Blacklist so that the driver
# continues to build and is installable for the few cases where its
# really needed.
blacklist amd76x_edac

blacklist radeon
blacklist amdgpu
```

> Note : Ici j'ai indiqué radeon et amdgpu car j'ai décidé de présenter ma RX460 a la VM. Adapter ceci aux besoins.

### Paramétrage de Linux pour VFIO et IOMMU

Bon déjà c'est quoi ces deux noms d'oiseaux ? 

- IOMMU ou Input/Output Memory Management Unit est un mécanisme bas niveau qui permet de traduire des adresses logiques en adresses physique. En gros, cela permet au CPU de discuter avec les périphériques qui l'entoure (comme une carte graphique) . 
- VFIO est une API contenue dans le noyau Linux permettant de rendre ce fameux IOMMU accessible depuis l'userspace. Et donc, en gros, de présenter du hardware a une VM. 

Pour charger tous les modules nécessaires au boot, nous allons modifier le fichier `/etc/default/grub` :

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"
```

> Note : Si vous avez de l'amd il faut mettre `amd_iommu=on`

Maintenant nous allons ajouter les modules dans `/etc/initramfs-tools/modules` pour le rendre opérationnel : 

```bash
vfio-pci ids=1002:67ef,1002:aae0
vfio
vfio_iommu_type1
vfio_pci
vhost-net
```

> Bien entendus, mettez vos ID PCI 

Pour que cela prenne effet, nous allons reconstruire la configuration de grub et celle d'initramfs :

```bash
sudo grub-update
sudo update-initramfs -u 
```

Nous allons maintenant reboot la machine. 

Une fois la machine up, nous allons vérifier que tout est bien charge : 

```bash
root@WORKSTATION:/home/jugu# lsmod | grep vfio
vfio_pci               53248  2
irqbypass              16384  8 vfio_pci,kvm
vfio_virqfd            16384  1 vfio_pci
vfio_iommu_type1       24576  1
vfio                   32768  6 vfio_iommu_type1,vfio_pci
root@WORKSTATION:/home/jugu# dmesg | grep vfio-pci
[    1.814434] vfio-pci 0000:02:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
[    2.567320] vfio-pci 0000:02:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
[  136.561144] vfio-pci 0000:02:00.0: enabling device (0000 -> 0003)
[  136.585130] vfio-pci 0000:02:00.1: enabling device (0000 -> 0002)
root@WORKSTATION:/home/jugu# dmesg | grep VFIO
[    1.810789] VFIO - User Level meta-driver version: 0.3

```

Si vous n'avez rien dans `dmesg | grep vfio-pci`, créer un fichier `/etc/modprobe.d/vfio.conf` contenant ceci : 

```bash
options vfio-pci ids=1002:67ef,1002:aae0
```

> Note : Redémarrage nécessaire avant de retenter.

### Installation de la VM 

Pour ma part, j'ai installé une VM avec les spécifications suivantes depuis virt-manager : 

- 2 cœurs CPU
- 8 Go de RAM FIXE
- Une image Qcow2 sur un SSD de 100 Go driver Virtio 
  - Options avances -> options de performances : writeback 
  - Options avances -> options de performances : Mode d'E/S Threads
- Réseau virtio 
- Carte son AC 97 
- Lecteur CD 1 Windows 10
- Lecteur CD 2 Virtio.iso (https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso)

> Attention, ne pas déployer directement la carte graphique. 

#### Installation de Windows

Lancer une installation de Windows classique, au moment de choisir le disque n’apparaît pas :

- Cliquer sur charger un pilote
- Cliquer sur le disque virtio-win et allez chercher le driver `viostor`
- Cliquer à nouveau sur charger un pilote
- Cliquer sur le disque virtio-win et allez chercher le driver `NETKVM`
- Cliquer une dernière fois sur charger un pilote
- Cliquer sur le disque virtio-win et allez chercher le driver `balloon`
- Poursuivez l'installation 

Une fois que la machine démarre, télécharger le driver [spice-guest-tools](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)

Installez-le, cet exécutable installera tous les drivers automatiquement. 

#### Installation des drivers AC97 

Les drivers de cette carte n’étant plus maintenu, il faut ruser pour les installer quand mème.

Pour cela, faites un clic droit sur le menu démarrer. 

Maintenez Shift puis faites un clic droit sur `Arreter ou se deconnecter` et cliquez sur  `Redémarrer`.

La machine va donc s’exécuter et apparaîtra un écran spécifique, cliquer sur `Dépannage` puis sur `Options avancées` et cliquer sur `Paramètres` et `Redémarrer` .

Au reboot, une page propose 9 choix différents, sélectionner le 7.

Maintenant téléchargez le [driver](https://realtek-download.com/wp-content/uploads/2014/07/6305_vista_win7_pg537.zip) et installez-le depuis le gestionnaire de périphérique (clic droit, mettre à jour sur `Contrôleur audio multimédia` ) . 

Windows vous mettra un message d’avertissement, cliquer sur installer quand mème.  

Sur l’hôte modifier le `.bashrc` :

```
export QEMU_PA_SAMPLES=128 QEMU_AUDIO_DRV=alsa
```

Puis faites de mème dans la console. Le son marche miraculeusement !

#### Ajout du GPU 

Maintenant, ajouter vos deux périphériques ( audio et vidéo) via virt-manager.

Installer les drivers graphiques propres au GPU utilisé.  

Redémarrez ensuite avec un écran branché sur le second GPU (celui de la VM). 

## Looking Glass

Pour se passer de périphériques externes en supplément tel qu'un écran et un combo clavier/souris. Une alternative existe, Looking Glass. 

Ce logiciel encore au stade purement expérimental permet d'effectuer une copie bit a bit du flux vidéo émis par la carte graphique de la VM et de retourner l'affichage dans une fenêtre de la machine hôte. 

C'est donc une sorte de client VNC avec accélération GPU en plus. 

> Note : Le projet est en cours de développement actif alors jouez le jeu en remontant les bugs et les fonctionnalités requises. 

### Installation sur l’hôte 

Il est nécessaire de compiler le client. Débutons par les prérequis : 

~~~bash

apt-get install binutils-dev cmake fonts-freefont-ttf libsdl2-dev libsdl2-ttf-dev libspice-protocol-dev libfontconfig1-dev libx11-dev nettle-dev

~~~

Maintenant compilons le client : 

~~~bash
mkdir build
cd build
cmake ../
make
~~~

### Installation sur l’invité 

> Couper la VM proprement avant

Pour cela, modifier le fichier XML de la machine virtuel Windows en ajoutant ceci dans la partie `Device` :

```bash
jugu@WORKSTATION:~$ virsh list
 Id    Name                           State
----------------------------------------------------
 1     win10                          running
jugu@WORKSTATION:~$ virsh edit win10 
```

Et coller ceci : 

``` bash
  <shmem name='looking-glass'>
      <model type='ivshmem-plain'/>
      <size unit='M'>32</size>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0b' function='0x0'/>
  </shmem>
```

Maintenant dans le gestionnaire de périphérique apparaît un `Contrôleur de RAM standard PCI` dans la catégorie `périphérique système`. 

Télécharger [ceci](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/virtio-win10-prewhql-0.1-161.zip)  et le décompresser. 

Faire un clic droit sur le fameux contrôleur et mettre à jour le pilote par un clic droit.  

Cliquer sur `parcourir mon ordinateur a la recherche du logiciel pilote`  et aller chercher le dossier précédemment obtenu selon l'OS. 

Maintenant télécharger l’exécutable Looking Glass [ici](https://github.com/gnif/LookingGlass/releases/download/a12/looking-glass-host.exe) et modifiez la clé de registre suivante : 

 ```bash
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
 ```

Ajouter une chaîne contenant le chemin vers Looking Glass. 

> Note : Utilisez regedit 
>
> Note 2: Pour copier un chemin, maintenez shift enfoncé et cliquer sur Copier en tant que chemin d’accès  

Les plus perspicaces d’entre vous se demanderont pourquoi nous ne créons pas simplement un service et vous auriez raison. Mais le logiciel utilise une fonction de capture qui est bloqué par Windows sur l'UAC  et qu'il est impossible de lancer de cette façon. 

Maintenant lancez l’exécutable a la main en double cliquant dessus. 

### Utilisation du client

Pour utiliser le client voici une commande de base : 

```bash 
sudo looking-glass-client -c 127.0.0.1 -w 1920 -b 1080 -M -m 72
```

Pour prendre le focus de votre clavier et de votre souris, appuyez sur la touche `Pause attn`, appuyez à nouveau sur cette même touche pour sortir du focus.
> Note : La résolution doit être configurée comme celle de votre écran physique


## Troubleshoot 

- Error 127 : Reboot du système requis. Le périphérique s'est mal déconnecté de la machine virtuelle.
- Error -22 : Activer le Vt-d dans le BIOS. Le Vt-d n'est pas forcément activé avec l'activation de la virtualisation CPU.
- Error : IOMMU groups not viable : Installez le patch ACS override. 
  - Télécharger les `.deb` ici : https://queuecumber.gitlab.io/linux-acs-override/
    - Header 
    - image
  - Installer les header puis l'image 
  - éditez `/etc/default/grub` et ajouter `pcie_acs_override=downstream` a la fin de la ligne `GRUB_CMDLINE_LINUX_DEFAULT=`
- Looking Glass affiche un écran noir : mettre la résolution identique sur l’écran dans Windows et dans Looking Glass.
- Looking Glass client ne se lance pas, faites de l’écran physique l’écran principal dans les options d'affichage. Ou alors branchez-en un si ce n'est pas déjà fait. 
- Looking Glass ne détecte mes mouvements de souris qu'au clic : Désactiver la tablette virtuelle. 

- Pour éviter le crash de certains jeux avec un BSOD ` KMODE_EXCEPTION_NOT_HANDLED` modifier le fichier `/etc/modprobe.d/kvm.conf` :

  ```bash
  options kvm ignore_msrs=1
  ```

  

## Sources 

https://heiko-sieger.info/running-windows-10-on-linux-using-kvm-with-vga-passthrough/

https://davidyat.es/2016/09/08/gpu-passthrough/

http://vfio.blogspot.com/2015/05/vfio-gpu-how-to-series-part-4-our-first.html

https://github.com/saveriomiroddi/vga-passthrough/blob/master/3_BASIC_SETUP.mdhttps://github.com/saveriomiroddi/vga-passthrough/blob/master/3_BASIC_SETUP.md

https://blog.zerosector.io/2018/07/28/kvm-qemu-windows-10-gpu-passthrough/

https://forum.level1techs.com/t/pci-stub-wont-take-video-card/126311/2

https://www.reddit.com/r/VFIO/comments/8e7jk1/vfio_not_working_on_proxmox_51_official_iso/

http://vfio.blogspot.com/2016/09/intel-iommu-enabled-it-doesnt-mean-what.html

http://vfio.blogspot.com/2014/08/vfiovga-faq.html

http://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html

https://forum.level1techs.com/t/two-gpu-in-the-same-iommu-group/130613

https://forum.level1techs.com/t/kernel-updates-on-18-04-acs-patch/129761/3

https://queuecumber.gitlab.io/linux-acs-override/

https://unix.stackexchange.com/questions/478129/gpu-passthrough-works-with-uefi-firmware-but-not-windows-iso

https://www.reddit.com/r/VFIO/comments/945xej/looking_glass_black_screen/

https://github.com/gnif/LookingGlass/releases/tag/a12

http://donewmouseaccel.blogspot.com/2010/03/markc-windows-7-mouse-acceleration-fix.html

https://heiko-sieger.info/tuning-vm-disk-performance/

https://stackoverflow.com/questions/32193050/qemu-pulseaudio-and-bad-quality-of-sound

