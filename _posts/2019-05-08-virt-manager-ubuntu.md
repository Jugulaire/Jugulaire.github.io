---
layout: post
title: Installation de virt-manager sous Ubuntu 18.04
---

![Virt-manager]({{ site.baseurl }}/images/Virt-manager.png){:class="img-responsive"}

La version disponible dans les dépôts officiels de Ubuntu n'etant pas a jour, voici un moyen d'installer la version la plus récente de virt-manager.

Installation des prérequis :

~~~bash
sudo apt-get install python-libvirt libgtk-3-dev libvirt-glib-1.0 gir1.2-gtk-vnc-2.0 gir1.2-spiceclientgtk-3.0 libosinfo-1.0 python-ipaddr gir1.2-vte-2.91 python-libxml2 python-requests python-libvirt python3-pip intltool
~~~

Puis, sur pip :

~~~bash
pip3 install libvirt-python libxml2-python3
~~~

Maintenant, clonons le dépôt git :

~~~bash
git clone https://github.com/virt-manager/virt-manager.git
~~~

Enfin, nous lançons l'installateur :

~~~bash
./setup.py install
~~~

