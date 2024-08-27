---
layout: post
title: Compiler Marlin pour la FYSECT Cheetah 1.2b
---

![]({{ site.baseurl }}/images/img/cheetah.png){:class="img-responsive"}

Je suis heureux propriétaire d'une Ender3, c'est une machine solide qui sort des impressions vraiment propre sans faire de grosses modifications. Mais elle souffre d'un gros problème : elle fait beaucoup de bruit. 

En cherchant un peu sur le web j'ai découvert les contrôleurs moteurs TMC2208 qui permettent de réduire les vibration des moteurs pas à pas en optimisant le signal envoyé a ces derniers. 

J'ai finis par trouvé la Cheetah de FYSECT qui en plus réintégrer des TMC2208 offre une compatibilité totale avec la Ender3 pour seulement 25 euros. Sauf que contrairement à ce que pas mal d'articles de blog disent, pour 25 euros vous n'avez pas une machine exempte de bug. La carte est livrée avec un firmware pré configuré pour la Ender3 mais les contrôleurs envoient trop d'intensité aux moteurs et les ventilateurs commandés ne fonctionnent qu'en tout ou rien. 

Enfin bref, compilons ce firmware pour corriger le problème. 

## Récupérer PlatformIO 

Pour cette partie je vous laisse aller faire un tour sur le site https://platformio.org/

## Télécharger les bonnes sources du firmware

Ici je part sur le repo git officiel de Marlin : 

```bash
git clone https://github.com/MarlinFirmware/Marlin
``` 

Une fois fait, il faut ajouter les fichiers de configuration trouvable ici : 

```bash
https://github.com/MarlinFirmware/Configurations/tree/bugfix-2.1.x/config/examples/Creality/Ender-3/FYSETC%20Cheetah%201.2
```
Copier les fichiers suivants dans le sous dossier `Marlin` : 

- `_Bootscreen.h`
- `_Statusscreen.h`
- `Configuration.h`
- `Configuration_adv.h`  



Débutons par le fichier `Marlin/Configuration_adv.h ` à la ligne **1978** : 

```c
  /**
   * Optimize spreadCycle chopper parameters by using predefined parameter sets
   * or with the help of an example included in the library.
   * Provided parameter sets are
   * CHOPPER_DEFAULT_12V
   * CHOPPER_DEFAULT_19V
   * CHOPPER_DEFAULT_24V
   * CHOPPER_DEFAULT_36V
   * CHOPPER_PRUSAMK3_24V // Imported parameters from the official Prusa firmware for MK3 (24V)
   * CHOPPER_MARLIN_119   // Old defaults from Marlin v1.1.9
   *
   * Define you own with
   * { <off_time[1..15]>, <hysteresis_end[-3..12]>, hysteresis_start[1..8] }
   */
  #define CHOPPER_TIMING CHOPPER_DEFAULT_24V
```

Il faut vérifier que la valeur est bien en 24v sinon les moteurs vont surchauffer rapidement. 

> Mon moteur d'axe X est monté a 70c.. Pas top !

Ensuite, il faut changer la valeur de gestion des contrôleurs TMC2208 dans `Marlin/Configuration.h` et remplacerl es valeurs `TMC2208_STANDALONE` par la valeur `TMC2208`:

```c
/**
 * Stepper Drivers
 *
 * These settings allow Marlin to tune stepper driver timing and enable advanced options for
 * stepper drivers that support them. You may also override timing options in Configuration_adv.h.
 *
 * A4988 is assumed for unspecified drivers.
 *
 * Options: A4988, A5984, DRV8825, LV8729, L6470, TB6560, TB6600, TMC2100,
 *          TMC2130, TMC2130_STANDALONE, TMC2160, TMC2160_STANDALONE,
 *          TMC2208, TMC2208_STANDALONE, TMC2209, TMC2209_STANDALONE,
 *          TMC26X,  TMC26X_STANDALONE,  TMC2660, TMC2660_STANDALONE,
 *          TMC5130, TMC5130_STANDALONE, TMC5160, TMC5160_STANDALONE
 * :['A4988', 'A5984', 'DRV8825', 'LV8729', 'L6470', 'TB6560', 'TB6600', 'TMC2100', 'TMC2130', 'TMC2130_STANDALONE', 'TMC2160', 'TMC2160_STANDALONE', 'TMC2208', 'TMC2208_STANDALONE', 'TMC2209', 'TMC2209_STANDALONE', 'TMC26X', 'TMC26X_STANDALONE', 'TMC2660', 'TMC2660_STANDALONE', 'TMC5130', 'TMC5130_STANDALONE', 'TMC5160', 'TMC5160_STANDALONE']
 */
#define X_DRIVER_TYPE  TMC2208
#define Y_DRIVER_TYPE  TMC2208
#define Z_DRIVER_TYPE  TMC2208
#define E0_DRIVER_TYPE TMC2208

```

Si cela n'est pas configuré correctement vous obtiendrez `TMC CONNEXION ERROR` au lancement de l'imprimante. En fait les TMC2208 sont des contrôleurs "intelligent" et peuvent communiquent avec le MCU de la carte. Si les paramètres ne sont pas exact et que le firmware attend des infos de la part des contrôleurs une erreur apparaît et rend l'imprimante inutilisable.

> Note : Des génies vous diront de mettre les TMC en mode standalone ce qui est une mauvaise idée. Dans mon cas cette idée de génie s'est soldée par un layer shifting de l'espace et des stepper chaud comme une baraque a frite. 



Vous pouvez maintenant cliquer sur la tête de martien a gauche de l'écran chercher `STM32F103RC_fysetc_maple`. Attendez quelques secondes que l'option `Build` apparaisse et cliquer dessus.

![platformio]({{ site.baseurl }}/images/Build_marlin.png){:class="img-responsive"}

Si la compilation se déroule comme prévue, vous devez obtenir ceci :

```bash
Advanced Memory Usage is available via "PlatformIO Home > Project Inspect"
RAM:   [==        ]  19.9% (used 9800 bytes from 49152 bytes)
Flash: [========  ]  75.3% (used 197428 bytes from 262144 bytes)
===== [SUCCESS] Took 157.33 seconds ===========================

Environment              Status    Duration
-----------------------  --------  ------------
megaatmega2560           IGNORED
megaatmega1280           IGNORED
STM32F103RC_fysetc       SUCCESS   00:02:37.326
STM32F103RC_bigtree      IGNORED
===== 1 succeeded in 00:02:37.326 ==============================
```

Maintenant place au flash du firmware, branchez l'imprimante au PC pour trouver le port série vers lequel envoyer la purée : 

> Note : pas besoin de l'allumer, vous pouvez même le faire avant installation.

```bash
ls /dev/serial/by-id/
usb-1a86_USB2.0-Serial-if00-port0
```

Maintenant, place a l'installation de l'outil de flash : 

```bash
git clone https://git.code.sf.net/p/stm32flash/code stm32flash-code
Clonage dans 'stm32flash-code'...
remote: Enumerating objects: 1324, done.
remote: Counting objects: 100% (1324/1324), done.
remote: Compressing objects: 100% (649/649), done.
remote: Total 1324 (delta 890), reused 997 (delta 671)
Réception d'objets: 100% (1324/1324), 1.03 MiB | 941.00 KiB/s, fait.
Résolution des deltas: 100% (890/890), fait.

cd stm32flash-code/

make
cc -Wall -g   -c -o dev_table.o dev_table.c
cc -Wall -g   -c -o i2c.o i2c.c
cc -Wall -g   -c -o init.o init.c
cc -Wall -g   -c -o main.o main.c
cc -Wall -g   -c -o port.o port.c
cc -Wall -g   -c -o serial_common.o serial_common.c
cc -Wall -g   -c -o serial_platform.o serial_platform.c
cc -Wall -g   -c -o stm32.o stm32.c
cc -Wall -g   -c -o utils.o utils.c
cd parsers && make parsers.a
make[1] : on entre dans le répertoire « /home/jugu/Téléchargements/stm32flash-code/parsers »
cc -Wall -g   -c -o binary.o binary.c
cc -Wall -g   -c -o hex.o hex.c
ar rc parsers.a binary.o hex.o
make[1] : on quitte le répertoire « /home/jugu/Téléchargements/stm32flash-code/parsers »
cc  -o stm32flash dev_table.o i2c.o init.o main.o port.o serial_common.o serial_platform.o stm32.o utils.o parsers/parsers.a

sudo make install
[sudo] Mot de passe de jugu : 
cd parsers && make parsers.a
make[1] : on entre dans le répertoire « /home/jugu/Téléchargements/stm32flash-code/parsers »
make[1]: « parsers.a » est à jour.
make[1] : on quitte le répertoire « /home/jugu/Téléchargements/stm32flash-code/parsers »
install -d /usr/local/bin
install -m 755 stm32flash /usr/local/bin
install -d /usr/local/share/man/man1
install -m 644 stm32flash.1 /usr/local/share/man/man1
```

Nous allons pouvoir flasher notre carte : 

> Note : Le firmware se trouve dans le dossier `Marlin-bugfix-2.1.x/Marlin-bugfix-2.1.x/.pio\build/STM32F103RC_fysetc_maple/firmware.bin` notez bien où vous l'avez mis !

```bash
cp Marlin-bugfix-2.1.x/Marlin-bugfix-2.1.x/.pio\build/STM32F103RC_fysetc_maple/firmware.bin ./
sudo stm32flash -w firmware.bin -v -i rts,dtr /dev/serial/by-id/usb-1a86_USB2.0-Serial-if00-port0 
stm32flash 0.5

http://stm32flash.sourceforge.net/

Using Parser : Raw BINARY
Interface serial_posix: 57600 8E1

GPIO sequence start
 setting port signal rts to 1... OK
 delay 100000 us
 setting port signal dtr to 1... OK
GPIO sequence end

Version      : 0x22
Option 1     : 0x00
Option 2     : 0x00
Device ID    : 0x0414 (STM32F10xxx High-density)
- RAM        : Up to 64KiB  (512b reserved by bootloader)
- Flash      : Up to 512KiB (size first sector: 2x2048)
- Option RAM : 16b
- System RAM : 2KiB
Write to memory
Erasing memory
Wrote and verified address 0x080308bc (100.00%) Done.
```

## Activer le mesh leveling 

Tout ça c'est bien sympa mais a quoi bon compiler le firmware nous-même si ce n'est pas pour le modifier ? Le mesh leveling c'est un outil qui permet de compensé un plateau pas droit par un leveling en plusieurs points qui seront ensuite compensés par le moteur de l'axe Z. 

En gros au lieu de faire un leveling aux 4 coins, vous le faites en 16 endroits pour en améliorer la précision. La Ender est une machine solide donc pas de panique, ce ne sera pas une action à réaliser tous les matins. 

Pour modifier le firmware nous allons dé-commenter quelques lignes dans `configuration.h` :

```bash
#define MESH_BED_LEVELING
#define GRID_MAX_POINTS_X 5 
#define LCD_BED_LEVELING
#define EEPROM_SETTINGS
#define MESH_INSET 50
```

Faites une recherche dans le fichier, par défaut `GRID_MAX_POINTS_X` est définis à 3 pour faire 9 points de mesures. Mais je préfère en faire 25. 

Ensuite, même schéma qu'avant, cliquer sur build et attendre la compilation du firmware. Nous allons maintenant procéder comme avant pour téléverser le firmware. 

## Flasher la carte sous windows 

Pour flasher notre carte sous Windows, utilisez flymcu disponible [ici](https://github.com/FYSETC/STM32Flasher)

Ouvrez le en tant qu'administrateur avec un clic droit :

![flymcu]({{ site.baseurl }}/images/img/flymcu.png){:class="img-responsive"}

Puis vérifiez que le paramétrage en bas de la fenêtre est bien identique a celui de ma capture avant de récupérer votre firmware et de cliquer sur `Start ISP(P)` :

> Note : Le firmware se trouve dans le dossier `Marlin-bugfix-2.1.x\Marlin-bugfix-2.1.x\.pio\build\STM32F103RC_fysetc_maple\firmware.bin` notez bien où vous l'avez mis !

![flymcu2]({{ site.baseurl }}/images/img/flymcu2.png){:class="img-responsive"}

Une fois finis vous pouvez quitter le programme et vous amuser avec votre nouvelle carte. 

## Point de vigilance 

De base les contrôleurs moteurs sont paramétrés un peut fort en intensité, lancez un Benchy et vérifiez en les touchant qu'il ne devinent pas trop chaud. Si tel est le cas rendez-vous dans le menu `Configuration -> Advanced settings -> TMC Crivers -> Driver Current` Et ajustez les parmaètres. 

Pour ma part j'ai les paramètres suivants : 

| Axe/moteur | Valeur |
| ---------- | ------ |
| X          | 450    |
| Y          | 450    |
| Z          | 750    |
| Extrudeur  | 550    |

Attention toutefois, si vous avez des axes qui bloquent ou un extrudeur qui claque. Augmentez la valeur pour avoir le bon ratio entre température des moteurs et mouvements fluides. 
