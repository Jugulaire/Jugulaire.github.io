---
layout: post
title: Utiliser un Raspberry Pi pour programmer un AVR
---

![]({{ site.baseurl }}/images/pizer0_avr.jpg)

## Installation de avrdude
Nous allons installer avrdude : 

### Prérequis 

```bash
pi@raspberrypi ~ $ sudo apt-get update
pi@raspberrypi ~ $ sudo apt-get install bison flex -y
```

### Installation 
Depuis les sources :

```bash
pi@raspberrypi ~ $ wget http://download.savannah.gnu.org/releases/avrdude/avrdude-6.2.tar.gz
pi@raspberrypi ~ $ tar xfv avrdude-6.2.tar.gz
pi@raspberrypi ~ $ cd avrdude-6.2/
```

Activer `linuxgpio` dans les sources de avrdude et installation :

```bash
pi@raspberrypi avrdude-6.2/~ $ ./configure – -enable-linuxgpio
pi@raspberrypi avrdude-6.2/~ $ make
pi@raspberrypi avrdude-6.2/~ $ sudo make install
```

Spécifier à avrdude quel GPIO utiliser :

```bash
pi@raspberrypi avrdude-6.2/~ $ sudo vi /usr/local/etc/avrdude.conf
```

Rechercher `linuxgpio` : 

```ini
#programmer
#  id    = "linuxgpio";
#  desc  = "Use the Linux sysfs interface to bitbang GPIO lines";
#  type  = "linuxgpio";
#  reset = ?;
#  sck   = ?;
#  mosi  = ?;
#  miso  = ?;
#;
```

Et le remplacer par ceci :

```ini
programmer
  id    = "linuxgpio";
  desc  = "Use the Linux sysfs interface to bitbang GPIO lines";
  type  = "linuxgpio";
  reset = 4;
  sck   = 11;
  mosi  = 10;
  miso  = 9;
;
```
Voici le câblage :

|  Raspberry   |    Arduino | AVR |
| ------------ | ---------- | --- |
|  GPIO 4      |   RST      | 1   |
|  GPIO 10     |   D11      | 18  |
|  GPIO 9      |   D12      | 17  |
|  GPIO 11     |   D13      | 19  |

### Test 

Testons la communication : 

```bash
sudo avrdude -c linuxgpio -p m8 -v 
```
**-c** Type de programmateur
**-p** Modèle de puce 
**-v** Mode verbeux 

Je me suis trompé de modéle d'aTmega :

```bash
avrdude: Device signature = 0x1e9205 (probably m48)
avrdude: Expected signature for ATmega8 is 1E 93 07
         Double check chip, or use -F to override this check.
```

Changeons le modéle comme l'indique AVR dude :

```bash
sudo avrdude -c linuxgpio -p m48 -v
avrdude: Device signature = 0x1e9205 (probably m48)
avrdude: safemode: hfuse reads as DF
avrdude: safemode: efuse reads as FF

avrdude: safemode: hfuse reads as DF
avrdude: safemode: efuse reads as FF
avrdude: safemode: Fuses OK (E:FF, H:DF, L:E2)

avrdude done.  Thank you.
```

## Premier programme 

Il est tout à fait possible de programmer un AVR ou même un Arduino sans l'IDE grâce à Arduino-MK :

```bash
pi@raspberrypi ~ $ sudo apt install arduino-mk
```

Créons un programme de base : 

```bash
pi@raspberrypi ~ $ mkdir blink
pi@raspberrypi ~ $ cd blink
pi@raspberrypi blink/~ $ vi blink.ino
```

Voici un bout de code :
```c++
#include <Arduino.h>

void setup() {
  pinMode(13, OUTPUT);     
}

void loop() {
  digitalWrite(13, HIGH);
  delay(500);
  digitalWrite(13, LOW);
  delay(100);
}
```

Créons un Makefile :
```makefile
include /usr/share/arduino/Arduino.mk
MCU = atmega48
F_CPU = 1000000L
ARDUINO_DIR = /usr/share/arduino
ARDUINO_LIBS =
```

Compilons tout ceci :

```bash
pi@raspberrypi blink/~ $ make
```

Un dossier `build-uno` est créé à la compilation et contient un fichier `.hex`

```bash
pi@raspberrypi blink/~ $ sudo avrdude -c linuxgpio -p atmega8 -v -U flash:w:build-uno/blink.hex:i
```
> Note : Ajustez le type de puce, n'allez pas griller un aTmega pour rien...

## Ajout de librairies : 

Ajouter ceci dans le Makefile pour ajouter des librairies :
```makefile
USER_LIB_PATH += /home/my_username/my_libraries_directory
ARDUINO_LIBS += Wire \
                SPI \
                my_custom_lib
```

# Troubleshoot :

## Erreur GPIO :
```
Can't export GPIO 4, already exported/busy?: Device or resource busy
```
Fix : 
```bash
echo 4 > /sys/class/gpio/unexport 
```
## Exemple de fuse :
Par défaut certaines puces n'ont pas les bons fuse en place. Ces fuses permettent de déffinir les paramètres des aTmega. Il est possible de les modifier. 

> ATTENTION :
Une mauvaise manipulation de ces fuse peu rendre inutilisables vos µcontrolleur 

### Pour un atmega 48 en internal crystal 8mhz

```bash
sudo avrdude -c linuxgpio -p atmega48 -U lfuse:w:0xe2:m -U hfuse:w:0xdf:m -U efuse:w:0xff:m
```

### Pour un atmega 8 en internal crystal 8mhz 

```bash
sudo avrdude -c linuxgpio -p m8 -U lfuse:w:0xe4:m -U hfuse:w:0xd9:m 
```

Utilisez ce site pour savoir quoi envoyer dans AVR dude : https://www.engbedded.com/fusecalc/

> Internal crystal veut dire que l'aTmega utilise son oscillateur intégré pour fonctionner. Ce n'est pas aussi précis qu'un quartz mais cela permet d'utiliser ces puces sur une breadboard de façon minimaliste. 

> Le blog de ladyada parle aussi de ces fameux fuses https://www.ladyada.net/learn/avr/fuses.html

