---
layout: post
title: Freecad : Découper un assemblage de pièces  
---

![freecad]({{ site.baseurl }}/images/freecad.jpeg){:class="img-responsive"}

Dans cet article je vais vous partager la technique que j'ai utilisé pour découper un fichier STL contenant plusieurs pièces. Le but ici est de pouvoir paralléliser l’impression de ce modèle sur mes deux imprimantes. 

Voici le modèle d'origine :

![Model_base]({{ site.baseurl }}/images/Model_base.png){:class="img-responsive"}

Pour permettre la modification, je l'importe dans Freecad (Ctrl+O).

Je me rend dans l'atelier `Part`, je sélectionne la pièce dans l'arbre de gauche puis dans le menu `Pièce`, je clique sur `Créer la forme à partir d'un maillage...` . Dans la Tolérance, je met `0.01` . 

> Note : laissez bosser la machine, en fonction de la complexité du maillage cela prend du temps 

Une fois l'opération finalisé, une forme est ajouté dans l'arbre de gauche :

![Model_import]({{ site.baseurl }}/images/Model_import.png){:class="img-responsive"}

Maintenant que le maillage est importé, nous allons sélectionner le forme et nous rendre dans le menu `Pièce, Composé,Eclater le composé` . 

Après quelques secondes, un dossier apparaît dans l'arbre de gauche sélectionnez-le et cliquez sur chaque pièces avant de les exportés (ctrl+e) au format STL. 

  ![piece_coupé]({{ site.baseurl }}/images/piece_coupé.png){:class="img-responsive"}



