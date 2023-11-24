---
layout: post
title: Réparer la resolution dynamique sous Kali Linux avec Spice
---

![virtio]({{ site.baseurl }}/images/kali-fix-res.png){:class="img-responsive"}

En tant qu'utilisateur de KVM, j'ai remarqué que sous Kali Linux, il est impossible par défaut d'utiliser la mise a jour de résolution dynamique dans Spice. 

Il semble également qu'un bug soit ouvert a ce sujet sans véritable correction sous KDE ou XFCE4.
> Le bug n'existe pas sous Gnome

## Prérequis 
Avant toutes choses, vérifier que le paquet 'spice-vdagent' est bien installé. Et que son service est bien actif : 

```bash
systemctl status spice-vdagent      
● spice-vdagentd.service - Agent daemon for Spice guests
     Loaded: loaded (/lib/systemd/system/spice-vdagentd.service; indirect; v>
     Active: active (running) since Fri 2023-11-24 15:21:07 EST; 2min 54s ago
TriggeredBy: ● spice-vdagentd.socket
    Process: 1029 ExecStart=/usr/sbin/spice-vdagentd $SPICE_VDAGENTD_EXTRA_A>
   Main PID: 1037 (spice-vdagentd)
      Tasks: 2 (limit: 9456)
     Memory: 960.0K
        CPU: 155ms
     CGroup: /system.slice/spice-vdagentd.service
             └─1037 /usr/sbin/spice-vdagentd

Nov 24 15:21:07 kali-vm systemd[1]: Starting Agent daemon for Spice guests...
Nov 24 15:21:07 kali-vm systemd[1]: spice-vdagentd.service: Can't open PID f>
Nov 24 15:21:07 kali-vm systemd[1]: Started Agent daemon for Spice guests.
Nov 24 15:21:07 kali-vm spice-vdagentd[1037]: opening vdagent virtio channel
Nov 24 15:21:07 kali-vm spice-vdagentd[1037]: Set max clipboard: 104857600
Nov 24 15:21:07 kali-vm spice-vdagentd[1037]: Set max clipboard: 104857600

```
## La démarche 

De manière générale la correction la plus rapide ensuite est la commande suivante : 

```bash
xrandr --output Virtual-0 --auto
```
Seulement certaines fois, le nom de la carte virtuel n'est pas `Virtual-0` : 

```bash
xrandr --output Virtual-0 --auto
warning: output Virtual-0 not found; ignoring
```
Pour trouver les cartes virtuels je fais simplement `xrandr` : 

```bash
xrandr                                       
Screen 0: minimum 320 x 200, current 913 x 975, maximum 8192 x 8192
Virtual-1 connected primary 913x975+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
   913x975       59.96*+
   2560x1600     59.99    59.97  
   1920x1440     60.00  
   1856x1392     60.00  
   1792x1344     60.00  
   2048x1152     60.00  
   1920x1200     59.88    59

   ...SNIP..
```
Il me suffit alors de chercher le mot clé `connected`, heureusement awk est la pour me sauver : 

```bash
xrandr | awk '{if($0 ~/connected/){print $1; exit; }}'
Virtual-1
```

Je test : 

```bash
xrandr --output "$(xrandr | awk '{if($0 ~/ connected/){print $1}}')" --auto
```

La seule chose qu'il manque pour rendre cela automatique est un événement qui va déclencher la mise a jour de la résolution. 

Pour cela je lance un monitoring des événements udev avec la commande suivante : 

```bash
udevadm monitor
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[27.407164] remove   /devices/virtual/bdi/0:45 (bdi)
UDEV  [27.435621] remove   /devices/virtual/bdi/0:45 (bdi)
KERNEL[37.645700] change   /devices/pci0000:00/0000:00:01.0/drm/card0 (drm)
UDEV  [37.657519] change   /devices/pci0000:00/0000:00:01.0/drm/card0 (drm)
KERNEL[48.645767] change   /devices/pci0000:00/0000:00:01.0/drm/card0 (drm)
UDEV  [48.672822] change   /devices/pci0000:00/0000:00:01.0/drm/card0 (drm)
```

Je constate que si je change la taille de la fenêtre de ma machine virtuel j'obtiens des événements sur `/devices/pci0000:00/0000:00:01.0/drm/card0`. 

Il suffit maintenant de joindre les deux bouts.

## La solution 

Pour déclencher la modification de résolution, je vais créer une rules dans udev qui va lancer mon script sur une détection de changement de la taille de la fenêtre de ma VM : 

```bash
## /etc/udev/rules.d/10-resize.rules
ACTION=="change",KERNEL=="card0", SUBSYSTEM=="drm", RUN+="/usr/local/bin/resize" 
```

Mon script ressemble ensuite a cela : 

```bash
#! /bin/sh 
PATH=/usr/bin
desktopuser=$(/bin/ps -ef  | /bin/grep -oP '^\w+ (?=.*vdagent( |$))') || exit 0
export DISPLAY=:0
export XAUTHORITY=$(eval echo "~$desktopuser")/.Xauthority
xrandr --output "$(xrandr | awk '{if($0 ~/ connected/){print $1}}')" --auto
```

Les raisons de la complexification sont les suivantes : 

- Udev n'a pas accès a X, pour cela il faut définir les variables `DISPLAY` et `XAUTHORITY`
- `XAUTHORITY` depends de l'utilisateur en cours d'utilisation, il faut donc le trouver avant de pouvoir définir cette variable 
- udev ne sait qu'utiliser SH et non BASH


## Sources 

- https://askubuntu.com/questions/1269127/xrandr-fails-when-run-from-udev
- https://opensource.com/article/18/11/udev
- https://superuser.com/questions/1183834/no-auto-resize-with-spice-and-virt-manager
