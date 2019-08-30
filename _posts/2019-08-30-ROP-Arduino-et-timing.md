---
layout: post
title: Arduino et timing 
---

![clock]({{ site.baseurl }}/images/clock.jpeg)



  Ce titre de ticket n'évoque pas grand-chose au premier abord mais nous allons ici parler du temps que prend un **ATmega** pour passer une de ses I/O du niveau logique Bas (LOW) au niveau logique haut (HIGH).

  De plus nous aborderons différentes techniques pour manipuler les I/O de nos **ATmega**. 

  

  ## Le matériel 

  ![matos]({{ site.baseurl }}/images/matos.jpg)

  Pour la partie **ATmega** : 

  - Un **ATmega8** à 8 MHz monté sur un Raspberry Pi Zero 
  - Un **ATmega328** sur une carte custom a 16Mhz

  Pour la partie mesure :

  - Un oscilloscope DS203 (également connus sous le nom de DSO QUAD)

  # Test 1 : Low to High, the extra fast blinking 

  Pour ce premier test j'ai utilisé mon **ATmega8 à 8 Mhz** avec le code suivant : 

  ```c
  void setup() {
    pinMode(LED_BUILTIN, OUTPUT);
  }
  
  void loop() {
    digitalWrite(LED_BUILTIN, HIGH);                         
    digitalWrite(LED_BUILTIN, LOW);    
  }
  ```

  Pour résumer, nous mettons la sortie numéro 13 (LED_BUILTIN) à l'état haut puis on la met à l'état bas directement.

  Observons ceci avec un oscilloscope :

 

  ![IMAG000]({{ site.baseurl }}/images/IMAG000.BMP) 

  

Comme nous le voyons ici, un signal carré à une fréquence proche de **50 KHz** avec une période de **20µs** est généré par le microcontrôleur. 

  Faisons de même avec **l'ATmega328 à 16 MHz** :



  ![IMAG004]({{ site.baseurl }}/images/imag004.bmp)



Comme nous pouvons le constater,  un signal carré avec une fréquence proche de **145 KHz** avec une période de **6.87µs** est cette fois généré.

  ### Observations :

  Nous observons une fréquence du signal non linéaire entre les deux Atmega malgré la fréquence d'horloge doublée.

  Visuellement  la diode reste allumée. C'est en fait une conséquence de la fréquence  très haute du clignotement qui rend ce dernier imperceptible par l'œil humain.

  > NON, c'est pas de la PWM car le "Duty Cycle" est fixe

  ## Test 2 : Les uns derrière les autres 

  Pour ce test nous allons utiliser deux I/O et les faire changer d'état les une après les autres pour voir si une différence notable est visible sur l'oscilloscope. 

  > Note : Pour le test je vais utiliser uniquement l'ATmega à 8 MHz 

  Voici le code utilisé : 

  ```c
  #include <Arduino.h>
     
  void setup() {
  pinMode(13, OUTPUT);
  pinMode(12, OUTPUT);
  }
     
  void loop() {
  digitalWrite(13, HIGH);
  digitalWrite(12, HIGH);
  digitalWrite(13, LOW);
  digitalWrite(12, LOW);
  }
  ```

  Et voilà le signal :



  ![IMAG004]({{ site.baseurl }}/images/IMAG004.BMP)



  Nous constatons ici que la sortie 12 (en jaune) est décalé d'un quart de période par rapport a la sortie 13 soit de **10µs**. Notons également que la fréquence est passée de **50 Khz** a **25 Khz** avec une période de **40 µs**. 

  Mais pourquoi ce décalage ?  

  Voici le code de la fonction **digitalWrite** : 

  ```c
  void digitalWrite(uint8_t pin, uint8_t val)
  {
  	uint8_t timer = digitalPinToTimer(pin);
  	uint8_t bit = digitalPinToBitMask(pin);
  	uint8_t port = digitalPinToPort(pin);
  	volatile uint8_t *out;
  
  	if (port == NOT_A_PIN) return;
  
  	// If the pin that support PWM output, we need to turn it off
  	// before doing a digital write.
  	if (timer != NOT_ON_TIMER) turnOffPWM(timer);
  
  	out = portOutputRegister(port);
  
  	uint8_t oldSREG = SREG;
  	cli();
  
  	if (val == LOW) {
  		*out &= ~bit;
  	} else {
  		*out |= bit;
  	}
  
  	SREG = oldSREG;
  }
  ```

  Dans la première partie du code, la fonction **digitalPinToBitMask** convertis le numéro de pin envoyé en masque pour les différents ports de l'ATmega.

  Nous expliquerons ce fonctionnement juste après, mais il faut comprendre que ce traitement supplémentaire rend l'activation de pin simultanée impossible en utilisant cette fonction. 

  Un délai trop important est nécessaire pour cela, un délai de **10µs** dans le cas de notre ATmega @ 8Mhz.

  ## Test 3 : Les ports 

  Voici un bout de code simplissime pour expliquer le fonctionnement et l'avantage des ports de nos ATmega :

  ```c
  #include <Arduino.h>
  
  void setup() {
  	DDRB = 0x30; //13 et 12 en sortie
  }
  
  void loop() {
  	PORTB = 0x30; //13 et 12 HIGH
  	PORTB = 0x00; //Toutes les pins LOW
  
  }
  ```

  Il existe 3 ports dans les ATMega (qui sont en fait utilisé par la fonction situées plus haut) : 

  - B pour les pins 8 a 13
  - C pour les pins analogiques
  - D pour les pins 0 a 7

  Chaque port possède 3 registres sur 8 bits :

  - DDRx ou **Data Direction Register** utilisé pour assigner les pins en entrée (0) ou sortie (1) 
  - PORTx ou **Data Register** utilisé pour donner un état aux pins
  - PINx ou **Inputs Pin Register** servant à lire l'état des pins 

  > Dans ce mode de fonctionnement pas d'ADC et donc, pas de lecture de valeur analogique

  Voici ce que donne notre code sous forme de graphique :



  ![IMAG005]({{ site.baseurl }}/images/IMAG005.BMP)



  > J'ai affiché la courbe jaune inversé pour que nous puissions observer le timming plus facilement et parce que c'est classe.

  Avec les ports, nous avons directement demandé a notre ATmega d'allumer la sortie 12 et 13 plutôt que de le faire de façons séquentielle.  Ainsi, dès lors que les registres interne a notre puce sont définis, les sorties changent d'état au même moment. 

  Étant donné que cette action ne demande qu'une seule instruction plutôt que deux, elle raccourcit le temps d’exécution mais aussi la taille du code. 

  Il est également important de préciser que cette technique n'est pas recommandé par la documentation officielle Arduino car elle rend le code moins lisible. 

  

  ## Conclusion 

  Par le présent ticket, j'ai appris plusieurs choses assez intéressantes sur les ATmega et leur fonctionnement. 

  Nous avons par exemple vu qu'il existe plusieurs manières d’interagir avec les entrées/sorties avec plusieurs avantages/inconvénients. 

  ### Avantages de digitalRead et digitalWrite

  Ces deux fonctions offrent une meilleure lisibilité au code et peuvent être utilisé dans des cas ou le timing n'est pas important. 

  Ces fonctions sont par ailleurs recommandé par la fondation Arduino car elles offrent (avec pinMode) un bon moyen de manipuler les entrées/sorties mais aussi les résistances de pullup intégrées dans la puce. 

  > Non, je n'ai pas testé la partie digitalRead, mais étant donné qu'elle utilise la même technique pour obtenir l'état d'une pin... 

  ### Avantages des ports 

  Le premier avantage est la rapidité du changement d’état des pins mais c'est au détriment de la lisibilité de votre code. 

  Le deuxième avantage, c'est classe. Vraiment, vous pouvez parler en Hexadécimal ou en binaire à vos entrées sorties. 

  Privilégiez donc les ports dans le cas ou vous souhaitez manipuler des signaux numérique (A.K.A Digital) avec une forte contrainte de timing. 
