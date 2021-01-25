---
layout: post
lang: fr
title: Sécurisez vos conteneurs sous Docker
---

Je ne présenterai pas `Docker` dans ce billet : il y a énormément de ressources qui sont disponibles en ligne ( par exemple [docker-curriculum](https://docker-curriculum.com/) ). 

Un point qui n'est pas nécessairement abordé, c'est le fait que `Docker` n'a pas inventé les conteneurs : il utilise `LXC` et `cgroups` qui permettaient déjà de faire des conteneurs.

## Utiliser les conteneurs c'est bien, mais pas suffisant

Connectez-vous avec votre utilisateur sur la machine hôte et affichez votre `uid` :

```
theobarrague@hote$ id
uid=1000(theobarrague) gid=1000(theobarrague) groups=1000(theobarrague),998(docker)
```

Ok, donc visiblement vous êtes `theobarrague` et vous appartenez aussi au groupe `docker`, ça tombe bien c'est un prérequis pour utiliser `docker` sans être `root`.

Lancez un conteneur `Ubuntu` : 

```
theobarrague@hote$ docker run -it --hostname conteneur ubuntu:latest
root@conteneur:/# 
```

Vous êtes `root`, quelle abomination ! Affichez votre `uid` :

```
root@f868cf7dea0f:/# id
root@conteneur:/# id
uid=0(root) gid=0(root) groups=0(root)
```

Bon, votre `uid` est bien `0`, cela confirme bien que vous êtes l'utilisateur `root`. Vous devriez pouvoir lister tous les processus et toutes les connexions sur la machine alors, non ? ( `apt-get install -y iproute2` si vous n'avez pas `ss` ) 

```
root@conteneur:/# ps -aedf
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 10:11 pts/0    00:00:00 /bin/bash
root       804     1  0 10:20 pts/0    00:00:00 ps -aedf
root@conteneur:/# ss -ntup4
Netid      State      Recv-Q      Send-Q             Local Address:Port             Peer Address:Port      Process 
```

Ah non, vous ne listez que ceux présents sur votre conteneur : c'est dû à `cgroups` ! Il sépare les différents processus grâce aux espaces de nommage.

Notre conteneur est bien isolé de notre machine hôte mais n'est pas très utile ... Pourquoi ne pas faire tourner un serveur `HTTP` et partager les fichiers entre l'hôte et le conteneur ? Allons-y ! ( J'ai omis d'exposer le port `80` car cela n'importe aucun intérêt dans ma démonstration, je vais seulement exploiter le partage de fichiers )

```
theobarrague@hote$ echo "Hello!" > index.html
theobarrague@hote$ docker run -it -v $(pwd):/mnt ubuntu:latest
root@conteneur:/# ls -la /mnt 
total 8
drwxrwxr-x 3 1000 1000   96 Nov 16 10:25 .
drwxr-xr-x 1 root root 4096 Nov 16 10:25 ..
-rw-r--r-- 1 1000 1000    7 Nov 16 10:24 index.html
```

Cool, on partage des fichiers entre l'hôte et le conteneur. Euh, c'est quoi ce `1000` ? Pourquoi les fichiers / dossiers n'appartiennent pas à `theobarrague` puisque c'est lui qui les a créés ? Parce que les fichiers / dossiers sont taggés avec l'`uid` et le `gid` et non pas le nom d'utilisateur. De plus, impossible de convertir ces `uid:gid` en `theobarrague:theobarrague` puisqu'il n'existe pas dans le conteneur :

```
root@conteneur:/# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
[...]
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

Ajoutons-le dans ce cas :

```
root@conteneur:/# addgroup --gid 1000 theobarrague
Adding group `theobarrague' (GID 1000) ...
Done.
root@conteneur:/# adduser --no-create-home --uid 1000 --gid 1000 theobarrague
Adding user `theobarrague' ...
Adding new user `theobarrague' (1000) with group `theobarrague' ...
Not creating home directory `/home/theobarrague'.
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for theobarrague
Enter the new value, or press ENTER for the default
        Full Name []: 
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] Y
root@conteneur:/# cat /etc/passwd | grep theobarrague
theobarrague:x:1000:1000:,,,:/home/theobarrague:/bin/bash
```

Et relistons le contenu de `/mnt` :

```
root@conteneur:/# ls -lah /mnt
total 8.0K
drwxr-xr-x 3 theobarrague theobarrague   96 Nov 16 10:25 .
drwxr-xr-x 1 root root 4.0K Nov 16 10:25 ..
-rw-r--r-- 1 theobarrague theobarrague    7 Nov 16 10:24 index.html
```

Bon, chouette on affiche enfin le nom d'utilisateur plutôt que l'`uid` / `gid`. Eh mais, lions directement le fichier `/etc/passwd` du conteneur avec le `/etc/passwd` de l'hôte comme ça les utilisateurs seront automatiquement reconnus ( tant qu'on y est, on fait de même avec `/etc/group` et le `/etc/shadow` pour récupérer automatiquement les groupes et mots de passe, comme ça les comptes seront sécurisés ). Allez, hop :

```
theobarrague@hote$ docker run -it -v $(pwd):/mnt -v /etc/passwd:/etc/passwd -v /etc/group:/etc/group -v /etc/shadow:/etc/shadow ubuntu:latest
root@conteneur:/# cat /etc/passwd | grep theobarrague
theobarrague:x:1000:1000:,,,:/home/theobarrague:/bin/bash
root@conteneur:/# cat /etc/group | grep theobarrague
docker:x:998:theobarrague
theobarrague:x:1000:theobarrague
root@conteneur:/# cat /etc/shadow | grep theobarrague
theobarrague:$6$wpft...l4de1:18582:0:99999:7:::
```

Super, mon utilisateur, ses groupes et son mot de passe sont bien importés dans le conteneur. Comme on peut le voir, je n'appartiens qu'à deux groupes `theobarrague` et `docker`, je n'ai donc pas accès à `sudo` ( que ce soit sur la machine hôte ou le conteneur puisque les fichiers sont les mêmes ). 

Mais attends, je suis `root` dans mon conteneur et les fichiers sensibles sont modifiables par `root`, qu'est-ce qui m'empêche de m'ajouter au groupe `sudo` ? Go, go, go !

```
root@conteneur:/# vim /etc/group
root@conteneur:/# groups theobarrague
theobarrague : theobarrague sudo docker
```

Chic, donc si je retourne sur ma machine hôte, je devrais pouvoir sudoter maintenant :

```
theobarrague@hote$ sudo su
root@hote# id
uid=0(root) gid=0(root) groups=0(root)
```

Je suis devenu `root` en utilisant `docker`, pas terrible cette histoire de conteneur finalement ...

## Une première mesure basique mais terriblement efficace

Recréez le même conteneur vulnérable que tout à l'heure mais en ajoutant l'option `ro` ( lecture seule ) sur le montage :

```
theobarrague@hote$ docker run -it -v $(pwd):/mnt -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro -v /etc/shadow:/etc/shadow:ro ubuntu:latest
root@conteneur:/# rm /etc/passwd
rm: cannot remove '/etc/passwd': Read-only file system
root@conteneur:/# echo "backdoor:x:1000:1000:,,,:/:/bin/bash" >> /etc/passwd
bash: /etc/passwd: Read-only file system
```

Intéressant, il est désormais impossible de modifier les fichiers marqués en lecture seule ( évidemment, la lecture reste possible ! ).

Cette pratique reste tout de même fortement déconseillée car vous exposez vos utilisateurs présents sur l'hôte dans le conteneur ! ( bruteforce de mots de passe faibles, création d'une liste de noms d'utilisateur, etc ... ).

## Perdez vos privilèges 

On va repartir sur le même conteneur mais cette fois-ci on va l'exécuter sans privilège et devenir l'utilisateur `nobody` avec `-u nobody` :

```
theobarrague@hote$ docker run -it -v $(pwd):/mnt -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro -v /etc/shadow:/etc/shadow:ro -u nobody ubuntu:latest
nobody@conteneur:/$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
nobody@conteneur:/$ ls -la /etc/shadow 
-rw-r----- 1 root shadow 501 Oct  8 01:31 /etc/shadow
nobody@conteneur:/$ cat /etc/shadow 
cat: /etc/shadow: Permission denied
```

Pas grand chose à dire, on tient dejà quelque chose de plus sécurisé. 

Non seulement les fichiers sensibles ne sont qu'en lecture seule ( `root` peut toujours tenter sa chance, c'est ce que l'on a fait précédemment : il n'a pas réussi, il ne peut pas ) mais en plus l'utilisateur `nobody` n'y a pas accès puisque les règles d'accès sont conservées.

Ceci dit, pour l'attaquant, rien n'est perdu encore ! Admettons que sur ma machine hôte l'utilisateur `ssi` existe et que par malchance ( ou pur pur hasard hein ) que son `uid` soit exactement de `1069`. Admettons également que je lance le conteneur avec l'`uid 1069` et que je veuille devenir cet utilisateur sur la machine hôte, comment utiliser `docker` pour arriver à mes fins ? Facile, tout se joue avec le `setuid` !

Avec ma technique il faut pas mal de chance : à savoir un dossier accessible en écriture et partagé entre l'hôte et le conteneur, des `uids` qui correspondent ainsi que `gcc` ( ça reste encore envisageable ! ).

```
theobarrague@hote$ docker run -it -v $(pwd):/mnt -u 1069 vulnerable
I have no name!@conteneur:/$ cd /mnt
I have no name!@conteneur:/mnt$ ls -lah
total 8.0K
drwxrwxrwx 3 1000 1000   96 Nov 16 10:25 .
drwxr-xr-x 1 root root    4.0K Nov 16 11:13 ..
-r--r--r-- 1 1000 1000    7 Nov 16 10:24 index.html
nobody@conteneur:/mnt$ cat <<EOF >> main.c
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
    setreuid(geteuid(), geteuid());
    system("/bin/bash");
    return 0;
}
EOF
I have no name!@conteneur:/mnt$ gcc -o evil main.c
I have no name!@conteneur:/mnt$ chmod u+s evil
I have no name!@conteneur:/mnt$ exit
theobarrague@hote$ ls -la evil
-rwsr-xr-x 1 ssi ssi 16744 Nov 16 11:29 evil
theobarrague@hote$ ./evil
ssi@hote$ id 
uid=65534(ssi) gid=1000(theobarrague)
```

CQFD : Perdre les privilèges c'est bien, mais ça ne suffit pas encore.

## Utilisez un espace de nommage

Editez le fichier `/etc/docker/daemon.json` :

```
{
  "userns-remap": "dockremap"
}
```

Créez l'utilisateur `dockremap` :

```
theobarrague@hote$ sudo useradd --shell /sbin/nologin dockremap 
```

Configurez ses `id` utilisateurs et groupes subordonnés :

```
theobarrague@hote$ sudo sh -c 'echo "dockremap:165536:65536" >> /etc/subuid'
theobarrague@hote$ sudo sh -c 'echo "dockremap:165536:65536" >> /etc/subgid'
```

Puis relancez le service `Docker` :

```
theobarrague@hote$ sudo systemctl restart docker
```

Nous venons de configurer un espace de nommage pour l'utilisateur `dockremap` ! 

Maintenant, chaque fois que votre conteneur accédera à des ressources sur la machine hôte ( partage de fichiers par exemple ), cela se fera exclusivement au travers de `uids` et `gids` compris entre `165536` et `231072`. Ainsi, l'`uid 0` dans le conteneur se traduira par un `uid 165536` sur le système hôte :

```
theobarrague@hote$ docker run -it -v $(pwd):/mnt --hostname conteneur ubuntu:latest
root@conteneur:/mnt# cd /mnt
root@conteneur:/mnt# touch tropfort
root@conteneur:/mnt# ls -la tropfort
-rw-r--r-- 1 root root 0 Nov 16 12:52 tropfort
root@conteneur:/mnt# exit
theobarrague@hote$ ls -la tropfort
-rw-r--r-- 1 dockremap dockremap 0 Nov 16 12:52 tropfort
```

En utilisant un espace de nommage comme celui-ci, vous ne devriez pas tomber sur un `uid` existant ou possible sur le système hôte. Si vous ajoutez vos utilisateurs avec la commande `adduser`, exécutez `cat /etc/login.defs | grep UID` pour connaître les `uids` / `gids` minimum et maximum.

## Conclusion

Mettez en pratique toutes ces mesures ! Testez, testez, testez ...

Avec ces mesures relativement simples, vous limitez pas mal la surface d'attaque possible ... Cependant il existe encore de nombreuses autres bonnes pratiques liées à `Docker` ou au système `Linux`, sans être exhaustif : `Dockerfile`, variables d'environnement, images distroless, etc ...
