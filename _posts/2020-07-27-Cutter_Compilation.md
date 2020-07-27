---
layout: post
title: Compiler Cutter depuis les sources
---

Le but de ce billet est de donner une solution rapide pour compiler facilement Cutter sur Ubuntu 18.04. 

La première étape est d'installer les dépendances :

```bash
sudo apt install git build-essential cmake libzip-dev zlib1g-dev qt5-default libqt5svg5-dev
```

D'installer meson depuis pip3 :

```bash
sudo apt install python3-pip
umask 022
sudo pip3 install meson
```

Puis de compiler le programme :

```bash
git clone --recurse-submodules https://github.com/radareorg/cutter
cd cutter
mkdir build && cd build
sed -i "s/\/usr\/bin\/meson/\/usr\/local\/bin\/meson" 
cmake -DCUTTER_USE_BUNDLED_RADARE2=ON -DCMAKE_EXE_LINKER_FLAGS="-Wl,--disable-new-dtags" ../src
cmake --build .
```

> Note: il existe des versions pré-compilé sous forme de package Appimage. 
