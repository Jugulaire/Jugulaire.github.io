---
layout: post
title: Booster les performances de vos VM Windows sous KVM 
---

![virtio]({{ site.baseurl }}/images/kvm.jpg){:class="img-responsive"}

Lors de mes tests sous Windows, je monte souvent des environnements dédiés a l'aide de KVM. Seulement par défaut, KVM ne propose pas une configuration optimale (sans les drivers virtio) et donc la machine virtuelle est lente. 

La raison pour cela est le fait que Windows ne supporte pas les drivers Virtio par défaut. Par conséquent, si vous changez le disque dur en virtio, la machine ne démarreras plus. 

La solution la plus pratique est des déployer votre machine avec du materiel Virtio au moment de l'installation afin de charger les pilotes directement comme je l'explique dans mon guide sur le [PCI passthrough](https://jugulaire.github.io/GPU-passthrough-KVM-QEMU/). 

Mais qu'en est-il si vous avez une machine déjà installé et dont les drivers Virtio ne sont pas déployés ? Par exemple dans le cas d'une migration d'un hyperviseur tiers vers KVM ? 

Dans ce billet je vais expliquer comment faire pour convertir une machine Windows déjà installé et configuré sans. 

# Comment faire 

1. Installer les drivers viostor et NETKVM de l'iso virtio https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso

2. Installer également les [spice-gest-tools](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)

3. Ouvrir un terminal en tant qu'admin et taper :

```bcdedit /set {current} safeboot minimal\```

4. Couper la machine virtuelle et basculez le stockage en virtio dans les paramètres
  - Options avancées -> options de performances : writeback 
  - Options avancées -> options de performances : Mode d'E/S Threads

6. Démarrer la machine virtuelle, elle va basculer en safe mode.

   Note: Le safe mode va charger tous les drivers liées au démarrage en incluant le driver virtio. Étant donné que la machine comporte désormais un périphérique virtio, le noyau va l'activer par défaut afin qu'il soit chargé aux prochains boot. 

7. Vous pouvez maintenant couper le safe mode avec la commande suivante :

```bcdedit /deletevalue {current} safeboot\```

7. Lors du reboot normal, vous allez pouvoir profiter de performances optimales comparé a l'émulation SATA d'origine.