## 1. First steps
vérifier que la version de systemd est > 241 = `systemctl --version`  = 243
-    s'assurer que  systemd  est PID1 = `pidof systemd`  ->  902 1
-    check tous les autres processus système (PAS les processus kernel) : `ps ax | less  ->`

## 2. Gestion du temps

	- timedatectl fournit des informations sur la machine
- déterminer la différence entre Local Time, Universal Time et RTC time :
    -   Local Time : l'heure local (région)
    -   Universal Time : l'heure mondial 
    -   RTC time : l'heure propre à la machine , et bien plus précise,
        -   l'avantage d'utiliser RTC time est pour la précision, il est possible de déclencher des tâches ectremement précises.
-   timezones
    - changer de timezone pour un autre fuseau horaire européen :
        -   `timedatectl set-timezone Europe/London`
    -  désactiver le service lié à la synchronisation du temps avec cette commande, et vérifier à la main qu'il a été coupé
        -   `timedatectl set-ntp 0`
        -  `sudo tcpdump -n "broadcast or multicast" | grep NTP`

## 3. Gestion de noms

-   `hostnamectl set-hostname oct`
-   il est possible de changer trois noms avec  --pretty,  --static  et  --transient
-  expliquer la différence entre les trois types de noms. Lequel est à utiliser pour des machines de prod ?
    -   --pretty  : hostname est un nom d'hôte UTF8 de forme libre à présenter à l'utilisateur.
    -   --static  : nom d'hôte traditionnel stocké dans / etc / hostname
    -   --transient  : nom d'hôte dynamique maintenu par le kernel
        -   Pour une utilisation sur des machines de prod , il faut utiliser le  --static

## 4. Gestion du réseau (et résolution de noms)


-   afficher les informations DHCP récupérées par NetworkManager (sur une interface en DHCP)
    
-   stopper et désactiver le démarrage de  `NetworkManager`
    
    -   `sudo systemctl stop NetworkManager`
    -   `sudo systemctl disable NetworkManager`
-    démarrer et activer le démarrage de  `systemd-networkd`
    
    -   `sudo systemctl start systemd-networkd`
    -   `sudo systemctl enable systemd-networkd`
-   éditer la configuration d'une carte réseau de la VM avec un fichier  `.network`
    

### systemd-resolved`

-   activer la résolution de noms par  `systemd-resolved`  en démarrant le service (maintenant et au boot)
    
    -   `sudo systemctl start systemd-resolved`
    -   `sudo systemctl enable systemd-resolved`
    -   `sudo systemctl status systemd-resolved`
-   vérifier qu'un serveur DNS tourne localement et écoute sur un port de l'interfce localhost (avec  `ss`  par exemple)
    
    -   `ss -4 state listening`
        -   nous pouvons voir notre DNS qui écoute
-  Afin d'activer de façon permanente ce serveur DNS, la bonne pratique est de remplacer  `/etc/resolv.conf`  par un lien symbolique pointant vers  `/run/systemd/resolve/stub-resolv.conf`
    
    -   `ln –s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

-  Modifier la configuration de  `systemd-resolved`  
- elle est dans  `/etc/systemd/resolved.conf`  
- ajouter les serveurs de votre choix  - vérifier la modification avec  `resolvectl`  
-  mise en place de DNS over TLS  - renseignez-vous sur les avantages de DNS over TLS  
- effectuer une configuration globale (dans  `/etc/systemd/resolved.conf`)  
- compléter la clause  `DNS`  pour ajouter un serveur qui supporte le DNS over TLS (on peut en trouver des listes sur internet) 
- utiliser la clause  `DNSOverTLS`  pour activer la fonctionnalité  - valeur  `opportunistic`  pour tester les résolutions à travers TLS, et fallback sur une résolution DNS classique en cas d'erreur  - valeur  `yes`  pour forcer les résolutions à travers TLS  
- prouver avec  `tcpdump`  que les résolutions sont bien à travers TLS (les serveurs DNS qui supportent le DNS over TLS écoutent sur le port 853)  - vous pouvez aussi ajouter DNSSEC, en passant la valeur à True dans le fichier de configuration de  `systemd-resolved`

## 6. Gestion d'unité basique (services)

-  trouver l'unité associée au processus  `chronyd`  -`└─system.slice`  `├─httpd.service`  `├─chronyd.service`  `│ └─613 /usr/sbin/chronyd -u chrony`

# II. Boot et Logs

-  générer un graphe de la séquence de boot  ~~ -  `systemd-analyze plot > graphe.svg`~~ ~~ - très utile pour du débug~~ ~~ - déterminer le temps qu'a mis  `sshd.service`  à démarrer~~

# III. Mécanismes manipulés par systemd

## 1. cgroups

-   identifier le cgroup utilisé par votre session SSH
    
    -   identifier la RAM maximale à votre disposition (dans  `/sys/fs/cgroup`)
    -   `systemd-cgls`  puis on cherche notre session ssh -`└─system.slice`
        -   `├─sshd.service`
        -   `│ └─1068 /usr/sbin/sshd -D`
-     modifier la RAM dédiée à votre session utilisateur
    
    -   `systemctl set-property <SLICE_NAME> MemoryMax=512M`
    -   vérifier le changement
        -   toujours dans  `/sys/fs/cgroup`
-   la commande  `systemctl set-property`  génère des fichiers dans  `/etc/systemd/system.control/`
    
    -   vérifier la création du fichier
    -   on peut supprimer ces fichiers pour annuler les changements

## 3. Restriction et isolation

-   lancer un processus sandboxé, et tracé, avec  `systemd-run`
    -   un shell par exemple, pour faire des tests réseaux  `sudo systemd-run --wait -t /bin/bash`
    -   un service est créé :  `systemctl status <NAME>`
    -   un cgroup est créé :  `systemd-cgls`  pour le repérer, puis  `/sys/fs/cgroup`  pour voir les restrictions appliquées
    -   identifier le cgroup utilisé
    -   ajouter des restrictions cgroups
        -   `sudo systemd-run -p MemoryMax=512M <PROCESS>`
    -   ajouter un traçage réseau
        -   `sudo systemd-run -p IPAccounting=true <PROCESS>`
            -   effectuer un ping
            -   quitter le shell
            -   observer les données récoltées
    -  ajouter des restrictions réseau
        -   `-p IPAddressAllow=192.168.56.0/24`  +  `-p IPAddressDeny=any`

----------

Lancer un processus complètement sandboxé (conteneur ?) avec  `systemd-nspawn`  :

-   `sudo systemd-nspawn --ephemeral --private-network -D / bash`
    -   vérifier que  `--private-network`  a fonctionné :  `ip a`
    -   expliquer cette ligne de commande
    -   prouver qu'un namespace réseau différent est utilisé
        -   pour voir les namespaces utilisés par un processus donné, on peut aller voir dans  `/proc`
        -   `ls -al /proc/<PID>/ns`  : montre les liens vers les namespaces utilisés (identifiés par des nombres)
        -   si le nombre vu ici est différent du nombre vu pour un autre processus alors ils sont dans des namespaces différents
    -  ajouter au moins une option pour isoler encore un peu plus le processus lancé

## IV. systemd units in-depth

## 1. Exploration de services existants

-   observer l'unité  `auditd.service` = `systemctl status auditd.service`
    
    -   trouver le path où est définit le fichier  `auditd.service` = `/system.slice/auditd.service`
           ├─691 /sbin/auditd
           └─693 /usr/sbin/sedispatch
    -   expliquer le principe de la clause  `ExecStartPost` = commande qui es executer au demarrage 
    
## 2. Création de service simple

-   Créer un fichier dans  `/etc/systemd/system`  qui comporte le suffixe  `.service`  :

-   doit posséder une description
-   doit lancer un serveur web
-   doit ouvrir un port firewall quand il est lancé, et le fermer une fois que le service est stoppé
-   doit être limité en RAM

```
## 3. Sandboxing (heavy security)

-   Expliquer au moins 5 cinq clauses de sécurité ajoutées

## 4. Event-based activation

### A. Activation via socket UNIX 
Faire en sorte que Docker démarre tout seul s'il est sollicité

-   avoir installer  `docker`
-   vérifier que le service  `docker`  est éteint (`systemctl is-active docker`)
-   création d'un fichier  `/etc/systemd/system/docker.socket`
-   faire en sorte que le socket écoute sur le socket UNIX utilisé par docker
-   activer le socket  `systemd`  et prouver que le démon  `docker`  se lance uniquement lorsque le socket est sollicité

### B. Activation automatique d'un point de montage


Prouver le bon fonctionnement de l'automount

### [](#c-timer-systemd)C. Timer `systemd`

Les timers `systemd` ont un fonctionnement similaire au démon cron : ils permettent de planifier des tâches. Cela dit, les tâches lancées seront des services systemd, on profite alors du logging et monitoring natif du système.

-   `systemctl list-timers` permet de lister les timers en cours
-   `systemctl status <TIMER NAME` permet d'obtenir plus d'infos
-   `systemctl cat <TIMER_NAME` pour voir la construction d'un timer existant

Mise en place :

-    Créer un script simpliste qui archive un dossier sur le path créé dans le 2. : `/data/nfs`
-    Créer un fichier `.service` qui lance la backup
-    Créer un fichier `.timer` qui programme la backup tous les jours à heure fixe
    -   en utilisant la clause `OnCalendar` (voir [la doc officielle](https://www.freedesktop.org/software/syste
