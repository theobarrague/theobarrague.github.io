---
layout: post
lang: fr
title: Construisez un site statique avec Github et Jekyll
---

L'ARP spoofing ( connue également sous le nom de ARP poisoning ) est une technique d'attaque MITM ( Man In The Middle ). 
Elle permet de compromettre la communication entre deux machines sur un même réseau.

Le protocole ARP ( Address Resolution Protocol ) permet de traduire des adresses IP en adresses MAC. 

En effet, sur un même réseau, les machines communiquent entre elles en envoyant des messages destinés à une adresse MAC : 
lorsqu'une machine reçoit un message réseau, elle regarde si l'adresse MAC de destination correspond à son adresse MAC, si tel 
est le cas, alors la carte réseau va accepter ce message et poursuivre son traitement ( vérifier que l'adresse IP de destination 
correspond bien à notre adresse IP par exemple ).

Le protocole ARP est très simple, il contient deux types de messages :
 - Qui est `w.x.y.z` et quelle est ton adresse MAC ?
 - Je suis `w.x.y.z` et mon adresse MAC est `aa:bb:cc:dd:ee:ff`
 
 Puisqu'une image vaut mille mots ... 
 ![Schéma de fonctionnement de l'attaque ARP spoofing](https://github.com/heobarrague/theobarrague.github.io/raw/master/images/arp-spoof.png)
