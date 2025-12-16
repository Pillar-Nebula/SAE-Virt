# Compte Rendu d'Installation des Solutions de Virtualisation

Ce document détaille l'avancement des travaux d'installation des hyperviseurs Proxmox VE et Hyper-V sur les serveurs dédiés.

---

## Jour 1 : Préparation et Installation de Proxmox

### 1. Réorganisation de la Baie (Commun, environ 1h30)

Avant toute chose, nous avons, avec l’aide d’autres groupes, notamment celui de Valentin, de Pierre ou encore celui de Soyfoudine, réorganisé les différentes baies pour avoir un **câblage propre** et pour permettre à chaque groupe de savoir où il est branché.

![Page de connexion iDRAC du Serveur 7](images/image.png)

Avec un câblage plus propre, chaque groupe a pu connecter les cartes iDRAC de ses serveurs pour travailler via les postes de la salle.

### 2. Configuration d’iDRAC (Commun, environ 1h)

Étant donné le nombre de groupes, il nous était impossible de tous travailler dans la salle serveur. Pour cela, nous avons configuré les cartes iDRAC de nos 2 serveurs.

#### Adressage des iDRAC

| Paramètre | iDRAC du switch 6 (Serveur Hyper-V) | iDRAC du switch 7 (Serveur Proxmox) |
| :--- | :--- | :--- |
| **Adresse IP** | `10.202.6.216` | `10.202.6.217` |
| **Masque** | `255.255.0.0` | `255.255.0.0` |
| **Passerelle** | `10.202.255.254` | `10.202.255.254` |
| **DNS** | `10.202.255.200` | `10.202.255.200` |

Quant aux solutions de virtualisation, nous avons décidé de choisir **Hyper-V** sur le serveur 6, et **Proxmox** sur le serveur 7.

#### Test de connexion

Nous nous sommes connectés, depuis un poste du réseau, à la page WEB iDRAC correspondant à notre serveur.

![Page de connexion iDRAC du Serveur 7](images/1.png)

### 3. Installation de l'Hyperviseur Proxmox (Romain, environ 1h30)

Pour commencer, nous avons décidé d’installer Proxmox sur notre serveur 7.
Après quelques tentatives (étant donné que c'était ma première fois sur Proxmox), j'ai réussi à installer Proxmox de la bonne manière :

1. Téléchargement d'un ISO assez récent et placement sur le stockage le plus petit.
2. Redémarrage du serveur et boot sur l'installeur Proxmox.

#### Choix du Stockage
Lors de l'installation, nous avons sélectionné le système de fichiers **ZFS (RAID 0)**, comme convenu avec le professeur.

![Page de connexion iDRAC du Serveur 7](images/2.png)

#### Sécurité et Réseau
* Nous avons choisi de **changer le mot de passe** pour quelque chose de plus approprié qui respecte certaines normes de sécurité, ainsi qu’une adresse mail à nous.
* Nous avons configuré la partie réseau de notre Proxmox en choisissant une adresse dans la même plage que notre iDRAC (`10.202.6.1-10.202.6.255`).

![Page de connexion iDRAC du Serveur 7](images/3.png)

Après l’installation, notre Proxmox est **opérationnel**. Le projet peut continuer avec l'installation de 3 Proxmox en CEPH dans cet hyperviseur.
![Page de connexion iDRAC du Serveur 7](images/4.png)

### 4. Création des Machines Virtuelles (Alexandre)

Nous avons créé les VMs nécessaires à la mise en place du cluster CEPH.

#### Tableau des VMs créées :

| ID VM | Nom | OS | CPU (Cores) | RAM (Go) | Disque (Go) | IP |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: |
| 100 | PVE1 | Proxmox VE | 2 | 8.3 | 32 | 10.202.6.220/16 |
| 101 | PVE2 | Proxmox VE | 2 | 8.3 | 32 | 10.202.6.221/16 |
| 102 | PVE3 | Proxmox VE | 2 | 8.3 | 32 | 10.202.6.222/16 |

On voit que les 3 VMs sont **fonctionnelles** et en marche simultanément.

![Page de connexion iDRAC du Serveur 7](images/5.png)

---

## Objectifs Seconde Journée (Suite du Projet)

* Installation de la seconde solution (Hyper-V sur le serveur 6) (Romain)
* Configuration d'Hyper-V pour qu’il soit 100% fonctionnel (Romain)
* Configuration du CEPH sur notre serveur 7 (Proxmox) (Alexandre)

---

## Jour 2 : Installation et Configuration de Hyper-V

### 1. Installation de Windows Server et Hyper-V (Romain)

Pour cette journée, nous avons **installé** la seconde solution choisie : **Hyper-V**.

#### Problèmes rencontrés : Stockage et ISO

1. **Stockage :** Le serveur 6 avait des soucis de stockage, car il restait des OS et des configurations d’autres groupes, ce qui a nécessité un long travail de remise à blanc.
2. **Partitionnement :** Les disques étaient partitionnés de manière assez étrange, ce qui bloquait l’installation complète de Windows Server (erreur arrivant à la fin de plusieurs dizaines de minutes d’installation).
3. **Incompatibilité d'ISO :** L’ISO que nous avions n’était pas compatible, générant une nouvelle erreur.

![Page de connexion iDRAC du Serveur 7](images/6.png)
![Page de connexion iDRAC du Serveur 7](images/7.png)

### 2. Configuration Hyper-V et Réseau

Une fois l'installation de l'OS finalisée, nous avons fait la configuration de base de Windows Server **au niveau** réseau. Nous sommes ensuite passés à l'étape importante qu’est l’installation d'Hyper-V.

![Page de connexion iDRAC du Serveur 7](images/8.png)

Après avoir **installé** Hyper-V, la première chose à faire était de se concentrer sur la configuration réseau, donc à la création des **vSwitchs** :
* Un en interne (NAT)
* Un en externe (Bridge)

#### Problème d'Adressage IP
J’ai eu un souci **au niveau de l'**adressage IP : Quand je passais sur le vSwitch externe, mes VMs avaient directement un adressage **APIPA**. En changeant l’IP, les routes, etc., rien ne marchait et je ne pouvais même pas ping la passerelle. Pour y remédier, je suis passé par un vSwitch interne pour toutes nos VMs.

![Page de connexion iDRAC du Serveur 7](images/9.png)

Enfin, nous avons vérifié que le service de l'hyperviseur était bien **actif et prêt** à héberger des VMs.
Malgré pas mal de temps perdu, j'ai pu comprendre les subtilités qu'il faut connaître pour installer proprement un Windows Server, sur un serveur qui n'est pas le nôtre.

## Objectifs Troisième Journée (Suite du Projet)

* Régler les problèmes réseau sur Hyper-V
* Finaliser le CEPH

## Jour 3

### Continuité Hyper-V (Romain, environ 1h)

#### Difficultés rencontrées et résolutions : La configuration des vSwitchs

En voulant mettre en réseau mes 3 serveurs, j'ai eu des difficultés concernant la communication entre les machines virtuelles et le réseau physique. Malgré une configuration IP statique correcte, je rencontrais systématiquement l'erreur **"Destination Host Unreachable"** :

* Le commutateur virtuel (vSwitch) et la carte réseau physique du serveur possédaient tous deux une adresse IP sur le même segment réseau.
* Cette redondance créait un conflit de routage, empêchant Windows de diriger correctement les paquets sortants de la VM (le mode "pont" n'était pas effectif).

Pour résoudre ce problème, j’ai d’abord pensé que les problèmes venaient de mes VMs, j’ai donc tout refait de A à Z à ce niveau. En voyant que ce n’était pas efficace, j'ai dû procéder à une réinitialisation de la couche réseau : suppression du vSwitch défaillant, vérification de l'interface physique, et recréation propre du commutateur externe.

**Après quelques essais, j’ai compris que l’interface réseau qui était prise avait le mauvais nom : j’ai modifié cela et tout a marché.**

### Mise à jour doc et point financier (Romain, plusieurs heures)

J'ai pris le temps de me poser pour rattraper le retard sur la documentation. J'ai rédigé et regroupé les comptes rendus des derniers jours pour que tout soit carré, en faisant bien attention à respecter le format (Markdown, que je ne connaissais pas encore très bien, et que j'ai dû approfondir).

En même temps que cette remise au propre, j'ai commencé à regarder les coûts. Je suis en train de comparer les prix des différentes solutions techniques : ça me permet de dégrossir le travail pour avoir une première estimation de la solution que nous allons choisir à la fin.

### Configuration du Stockage sur Windows Server

**Création du volume de stockage "sécurisé" (Disque J:)**

Pour accentuer le côté sécurité, j'ai pris la décision de me renseigner sur comment faire pour, en plus du cluster à basculement, que les 2 SSD puissent se compenser "entre guillemets".
Pour protéger les données contre une panne matérielle, j'ai utilisé la fonction "Storage Spaces" de Windows Server. L'objectif était de combiner mes deux SSD physiques pour qu'ils ne forment qu'un seul lecteur visible : le disque **(J:)**.

J'ai configuré cet espace en mode miroir. En gros, Windows ne voit qu'un seul disque, mais tout ce qu'on écrit dessus est copié en temps réel sur les deux SSD. L'avantage de cette méthode, c'est que si l'un des disques lâche, le système reste en marche et les données sont préservées grâce au deuxième disque qui prend le relais automatiquement.

## Jour 4

Pendant cette demi-journée, j'ai continué les comptes rendus, puis j'ai tenté de commencer les préparatifs pour pouvoir faire le cluster à basculement. Je me suis assuré que les vSwitchs étaient correctement configurés, j'ai noté mes configurations réseau sur un bloc-notes pour ne pas avoir à perdre du temps, etc.

## Jour 4.5 (Week-end)

### Travail à distance (VPN Tailscale)
Pour travailler à distance, pendant mon week-end, j'ai discuté avec le groupe de Valentin, Clovis et Dorian et j'ai fini par télécharger et installer le logiciel Tailscale sur mon serveur Windows, je me suis authentifié via la page web qui s'est ouverte, et j'ai refait exactement la même manipulation sur mon PC chez moi pour les relier au même compte. Une fois les deux machines actives, j'ai simplement récupéré l'adresse IP commençant par 100 fournie par l'application pour le serveur, j'ai ouvert l'outil "Connexion Bureau à distance" de Windows et j'ai collé cette IP. L'écran de connexion est apparu instantanément, j'ai entré mes identifiants habituels et j'ai pris le contrôle total du bureau de mon serveur à distance.

### Mise en œuvre du Cluster S2D sur Hyper-V (Romain, environ 6h30)

#### Phase 1 : Préparation et Mise à niveau de l'infrastructure

##### Analyse du besoin
Avant de commencer la technique, nous avons défini l'objectif : garantir que si un serveur tombe, les services continuent de tourner. Dans une architecture classique, si le serveur meurt, la VM meurt avec lui car le stockage est local.
Nous avons donc décidé de partir sur du S2D (Storage Spaces Direct). Cela nous permet d'utiliser les disques locaux de nos 3 serveurs pour créer un gros volume partagé virtuel, sans avoir à acheter de baie SAN coûteuse.

##### Problème de licence :
Pour commencer, nous avons installé Windows Server 2025 sur nos trois nœuds. Malheureusement, au moment d'activer la fonctionnalité S2D, nous avons rencontré un problème : l'activation échouait systématiquement.
Après recherche, nous avons compris que nos serveurs étaient installés en édition Standard, alors que Microsoft réserve le S2D à l'édition Datacenter.
Pour éviter de perdre une journée à tout réinstaller et reconfigurer le réseau, nous avons opté pour une mise à niveau "sur place".

##### Solution appliquée :
Nous avons utilisé l'outil DISM en ligne de commande pour injecter une "clé KMS". Cela nous a permis de débloquer les fonctionnalités Datacenter à chaud.

##### Vérification :
On voit ci-dessous via la commande Get-ComputerInfo que le cluster est bien passé en édition Datacenter et est maintenant éligible.
*(Imaginez ici une capture d'écran PowerShell montrant l'édition Datacenter)*

#### Phase 2 : Implémentation du Stockage (S2D)

##### Configuration du Pool de Stockage
C'est ici que nous avons rencontré le plus de soucis niveau technique, lié à notre environnement de Virtualisation Imbriquée.
Normalement, tout se scripte en PowerShell. Cependant, comme nos disques sont virtuels (sur les VMs qui sont elles-mêmes des hyperviseurs), PowerShell ne parvenait pas à définir les "Storage Tiers" automatiquement.
Nous avons donc dû passer par l'interface graphique du Gestionnaire de cluster.

##### Architecture retenue
Pour la résilience, nous avons configuré le volume selon les paramètres suivants :

| Paramètre | Valeur | Justification |
| :--- | :--- | :--- |
| **Type de Résilience** | Miroir à 3 voies (3-Way Mirror) | Seule config qui tolère la perte de 2 nœuds simultanément. |
| **Formatage** | ReFS | Optimisé pour la virtualisation et l'intégrité des données. |
| **Capacité Utile** | 30 Go | Pour l'hébergement des VMs. |
| **Slack Space** | 50 Go | Espace non alloué laissé volontairement pour l'autoréparation. |

On voit qu’on a 150 Go en tout qui va nous servir pour nos basculements.

#### Phase 3 : Administration et Tests de Haute Disponibilité

##### Test de panne
Pour valider que notre infrastructure tient la route, nous avons simulé ce scénario : l'arrachage de câble (içi virtuel).

**Scénario :**
* Une VM tourne sur le nœud SRV1.
* Nous provoquons une extinction du SRV1.

**Résultat :**
Le système a détecté la perte de tout signe de vie entre les serveurs après environ 30 secondes.
On voit que le cluster a automatiquement redémarré la machine virtuelle sur le nœud SRV2. La résilience est validée.

#### Phase 4 : Migration à chaud et Réseau

##### Problème de sécurité réseau (MAC Spoofing)
Enfin, nous avons voulu tester la migration dynamique, pour des cas où nous devons mettre à jour le serveur qui héberge la VM.
Ici, j'ai eu un souci réseau bloquant : lors du déplacement de la VM, elle perdait toute connexion réseau.
La cause ? Le vSwitch de l'hôte physique voyait l'adresse MAC de la VM changer de port.

##### Correction :
Nous avons dû activer le "MAC Address Spoofing" sur les cartes réseaux des hyperviseurs virtuels et mettre le nom des vSwitchs (vSwitch_Cluster) partout.

##### Test final :
Nous avons lancé un ping continu vers la VM pendant une migration.

**Résultat :** Aucune perte de paquets !

Ce compte rendu a été écrit à la main et mis en markdown par Gemini
