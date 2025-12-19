CASTELLI - MASNY
## Projet de Migration : Sortir de VMware


### SYNTHÈSE : Scénario de Sortie de VMware

Nous avons choisi de  sélectionner Proxmox et HyperV
#### Verdict : Choix de Proxmox VE

Notre analyse technique et financière désigne **Proxmox VE** comme la meilleure solution de remplacement.

### 4 Arguments Décisifs

1.  Rentabilité Immédiate (TCO)
Proxmox (Open Source) permet une économie de 40 000 € à l'achat pour 3 serveurs par rapport à Hyper-V (Licences Datacenter + Veeam). Il reste plus rentable jusqu'à 58 VMs.

2.  Migration Fluide (VMware)
Point critique : Proxmox depuis quelques versions intègre un assistant d'import natif qui aspire les VMs VMware en quelques clics. À l'inverse, Hyper-V impose une conversion de format complexe et risquée.

3.  Rapidité de Mise en Œuvre
L'installation de Proxmox est "clé en main" (Cluster + Ceph + Réseau opérationnels en 4h). Hyper-V (S2D) s'est avéré lourd, nécessitant une configuration complexe (AD, DNS, PowerShell).

3.  Solution "Tout-en-un"
Proxmox inclut nativement le Stockage Distribué (Ceph) et le Backup (PBS). Hyper-V nécessite des rôles additionnels ou des logiciels tiers payants.
### Visualisation Stratégique

![[Pasted image 20251219200028.png|300]]
Conclusion :
Bien que Microsoft Hyper-V soit une solution robuste pour les entreprises déjà "Full Microsoft", Proxmox VE surpasse son concurrent sur les critères de coût, de simplicité et d'outillage de migration, répondant parfaitement à la problématique de remplacement de VMware.


---



---

# PARTIE 1 : DOSSIER D'ARCHITECTURE TECHNIQUE (PROXMOX VE)
**Responsable :** Alexandre

## 1. Présentation de l'Architecture

Ce document décrit les spécifications techniques de l'infrastructure de virtualisation mise en place. L'architecture repose sur le principe de **Virtualisation Imbriquée (Nested)** pour simuler un cluster de production à trois nœuds sur un serveur physique unique.

### 1.1 Matériel Physique (Niveau 0)
L'hôte physique est un serveur Dell PowerEdge R640.
* **CPU :** AMD EPYC (Instructions SVM activées).
* **RAM :** Dimensionnée pour supporter 3 hyperviseurs virtuels.
* **Stockage :** 2x SSD 1.7 To configurés en **RAID 1 Matériel** (Contrôleur PERC H730P).
* **Réseau :** Interface `eno1` (1 Gbps) connectée au switch de salle.

### 1.2 Architecture Logique (Niveau 1)
Le cluster Proxmox (`SAE-Cluster`) est composé de trois nœuds virtuels interconnectés via le protocole Corosync.

| Nœud | Hostname | IP Management | Rôle |
| :--- | :--- | :--- | :--- |
| **Node 1** | `pve1` | `10.202.6.220` | Calcul + Stockage (Ceph/ZFS) |
| **Node 2** | `pve2` | `10.202.6.221` | Calcul + Stockage (Ceph/ZFS) |
| **Node 3** | `pve3` | `10.202.6.222` | Calcul + Stockage (Ceph/ZFS) |

---

## 2. Configuration de l'Hyperviseur

### 2.1 Virtualisation Imbriquée
Pour permettre l'exécution de machines virtuelles (KVM) à l'intérieur des nœuds virtuels, le mode CPU "Host" a été forcé dans la configuration de l'hyperviseur racine.
### 2.2 Configuration Réseau

Le réseau repose sur un pont Linux standard (`vmbr0`) sans VLAN tagué au niveau de l'hôte, la segmentation étant gérée en amont.

**Configuration réseau type (`/etc/network/interfaces`) :**

Bash

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.202.6.220/16 < ici par exemple c'est le node 1
    gateway 10.202.6.254
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```
#### 2.3 Optimisation des Flux Cluster (Corosync)

Le bon fonctionnement d'un cluster Proxmox repose sur **Corosync**, le moteur de communication inter-nœuds.

- **Contrainte critique :** Corosync exige une latence réseau inférieure à 2ms.
- **Problématique Nested :** Dans notre environnement virtualisé, le réseau est partagé.
- **Configuration Kronosnet (knet) :** Proxmox 8 utilise par défaut le transport _Kronosnet_ en Unicast. Cela nous évite la complexité de configuration du Multicast (IGMP Snooping) sur les switchs physiques Dell, tout en garantissant le chiffrement des échanges de contrôle entre les nœuds `pve1`, `pve2` et `pve3`.
---

## 3. Stratégie de Stockage Hybride

L'infrastructure utilise une approche hybride combinant stockage distribué, local et répliqué.

### 3.1 Stockage Distribué : Ceph RBD

Utilisé pour la Haute Disponibilité (VMs Web & DNS).

- **Version :** Ceph Reef (18.2).
- **Topologie :** 3 Moniteurs (MON), 3 Managers (MGR), 3 OSDs.
- **Pool :** `pool-vms` (Size: 3, Min_size: 2).
- **Volume :** Chaque nœud contribue avec un vDisk de 100 Go (`/dev/sdb`).

**État du Cluster :**
![[Pasted image 20251219213433.png]]

#### 3.1.1 Fonctionnement de l'algorithme CRUSH

Contrairement à un RAID classique qui utilise une table d'allocation centralisée (goulot d'étranglement), Ceph utilise l'algorithme **CRUSH** (Controlled Replication Under Scalable Hashing).

- **Distribution :** Lorsqu'une VM écrit une donnée, le client calcule à la volée sur quel OSD (Disque) la donnée doit aller.
- **Réplication Synchrone :** Avec un `size=3` et `min_size=2`, l'écriture n'est confirmée à la VM que lorsque la donnée a été écrite physiquement sur **au moins 2 nœuds**. Cela garantit qu'aucune perte de données n'est possible même en cas de coupure électrique immédiate après une écriture.
- **Backend BlueStore :** Les OSD ont été formatés avec le backend _BlueStore_, qui permet à Ceph d'écrire directement sur le disque brut (raw device) sans passer par une couche de système de fichiers (ext4/xfs), augmentant ainsi les IOPS.
### 3.2 Stockage Local : ZFS

Utilisé pour les performances I/O (VM Client).

- **Pool :** `zfs-local`
- **Configuration :** Single Disk (`/dev/sdc`) avec compression LZ4 activée.
#### 3.2.1 Gestion de la mémoire et Compression (ARC)

L'utilisation de ZFS implique une consommation mémoire spécifique liée à l'**ARC (Adaptive Replacement Cache)**.

- **Stratégie :** ZFS utilise la RAM disponible pour mettre en cache les données les plus fréquemment lues (Read Cache). C'est pourquoi nos nœuds virtuels ont été dimensionnés avec 32 Go de RAM, afin de laisser environ 4 Go à l'ARC pour accélérer les VMs.
- **Compression LZ4 :** J'ai activé la compression LZ4 sur le pool. C'est un algorithme "zero-cost" : la vitesse de compression du CPU est plus rapide que la vitesse d'écriture du disque. Cela nous permet de gagner environ **1.4x** d'espace disque tout en augmentant virtuellement la vitesse d'écriture.
### 3.3 Stockage de Sauvegarde : ZFS Replication

Utilisé pour le PRA du serveur de backup.

- **Source :** `pve3/zfs-repl`
- **Cible :** `pve2/zfs-repl`
- **Fréquence :** Synchronisation toutes les 15 minutes.

---

## 4. Services Déployés

### 4.1 Serveur DNS (VM 105)

- **OS :** Alpine Linux.
- **Logiciel :** ISC Bind9.
- **Zone :** `sae.lan`
### 4.2 Serveur Web (VM 106)

- **OS :** Alpine Linux
- **Logiciel :** Nginx  
- **Contenu :** Page statique HTML accessible via `www.sae.lan`    
### 4.3 Proxmox Backup Server (VM 110)

- **IP :** `10.202.6.255`.
- **Rôle :** Centralisation des sauvegardes dédupliquées.
- **Datastore :** `backup-store` (sur ZFS).    

---

## 5. Mécanismes de Résilience

### 5.1 Haute Disponibilité (HA)

Le gestionnaire de ressources  de Proxmox est configuré pour surveiller les VMs critiques.

- **Groupe HA :** `critical-services`
- Tentative de migration, sinon redémarrage.
- **Membres :** VM 105 (DNS), VM 106 (Web).
#### 5.1.1 Gestion du Quorum et Watchdog

La stabilité du cluster repose sur le concept de **Quorum** (Majorité de votes).

- **Vote :** Nous avons 3 nœuds, donc 3 votes. La majorité requise est de 2.
- **Scénario de Panne :** Si le Nœud 1 tombe, les Nœuds 2 et 3 conservent le Quorum (2 votes > 1.5). Ils décident alors de "tuer" (Fence) le Nœud 1 pour éviter qu'il ne corrompe les données, puis redémarrent ses services.
- **Protection Watchdog :** Un mécanisme de "Watchdog" matériel (émulé ici par le noyau Linux `softdog`) est actif sur chaque nœud. Si le nœud perd le contact avec le cluster pendant plus de 60 secondes, le Watchdog force un redémarrage brutal du serveur (Hard Reset) pour garantir l'intégrité du système (Self-Fencing).

### 5.2 Plan de Reprise d'Activité

En cas de perte totale du nœud hébergeant le PBS, la réplication ZFS permet le redémarrage du service de sauvegarde sur le nœud voisin sans restauration longue, garantissant l'accès aux archives de backup.

---

# PARTIE 2 : DOSSIER TECHNIQUE - HYPER-V (Romain)

**Responsable :** Romain

## 1. Introduction et Architecture Globale

Cette partie détaille la méthodologie de mise en œuvre d'une infrastructure de virtualisation à haute disponibilité (HA).

Pour simuler un environnement avec plusieurs nœuds avec notre matériel, nous avons déployé une architecture en **Virtualisation Imbriquée (Nested Virtualization)** :

- **Niveau 0 (Physique) :** Serveur Dell exécutant Windows Server Datacenter.
- **Niveau 1 (Logique) :** Cluster de 3 machines virtuelles agissant comme nœuds.
- **Stockage :** Agrégation logicielle des disques via la technologie **Storage Spaces Direct (S2D)**.

---

## 2. Préparation de l'Infrastructure Physique (Niveau 0)

### 2.1 Matériel

L'infrastructure repose sur un serveur Dell (Switch 6). La première étape a consisté à faciliter l'administration via la carte de gestion à distance (iDRAC).

- **Adressage iDRAC :** `10.202.6.216 /16`
- **Passerelle :** `10.202.255.254`

L'accès iDRAC nous a permis de piloter l'alimentation, de monter les images ISO virtuellement et d'accéder à la console distante pour l'installation de l'OS.

### 2.2 Stockage Physique

Pour garantir le bon fonctionnement du futur S2D, il était important de présenter des disques vierges. Nous avons procédé à un nettoyage avant l'installation de l'OS, convertissant les disques au format GPT.

---

## 3. Configuration de l'Hyperviseur Hôte

### 3.1 Système d'Exploitation et Licences

Le choix s'est porté sur Windows Server édition Datacenter.

Contrairement à l'édition Standard, la version Datacenter est un prérequis pour l'activation de la fonctionnalité S2D et permet une virtualisation illimitée.

### 3.2 Activation du "MAC Address Spoofing"

C'est un point de configuration assez important pour le Nested Clustering. Par défaut, un port de vSwitch Hyper-V n'accepte que les trames provenant de l'adresse MAC de la VM connectée.

Or ici, les VMs imbriquées génèrent des trames avec leurs propres adresses MAC.

Nous avons activé l'usurpation d'adresse MAC sur les cartes réseaux virtuelles des 3 nœuds pour autoriser le transit de ces paquets :

PowerShell

```
Get-VMNetworkAdapter -VMName "Node*" | Set-VMNetworkAdapter -MacAddressSpoofing On
```

Sans cette commande, aucune communication réseau vers les VMs finales n'aurait été possible.

---

## 4. Implémentation du Cluster S2D (Cœur du système)

### 4.1 Storage Spaces Direct (S2D)

S2D permet de créer un stockage hautement disponible en utilisant des disques locaux attachés à chaque serveur, éliminant le besoin d'une baie SAN physique.

#### 4.1.1 Activation

Nous avons activé S2D (`Enable-ClusterS2D`) pour qu'il prenne le contrôle des disques, permettant à chaque nœud de lire et écrire sur les disques des voisins via le réseau.

#### 4.1.2 Stratégie de Résilience : Miroir à 3 Voies

Nous avons configuré le pool de stockage en **Miroir à 3 voies**.

- **Fonctionnement :** Chaque bloc de donnée est écrit simultanément sur 3 disques situés sur 3 nœuds différents.
- Cette méthode offre la résilience maximale. Elle permet de tolérer la panne simultanée de **2 nœuds** (ou 2 disques) tout en garantissant l'intégrité des données.

#### 4.1.3 Système de Fichiers ReFS

Les volumes partagés de cluster ont été formatés en ReFS (Resilient File System).

Avantages :

1. Détection et correction automatique de la corruption de données.
2. Optimisé pour la virtualisation et est donc plus performant.

#### 4.1.4 Gestion des vDisk

Lors de la création du volume vDisk_S2D (30 Go pour accueillir la VM), nous avons laissé 50 Go d'espace non alloué dans ce même pool.

Cet espace de réserve permet au système de lancer une reconstruction automatique en cas de perte d'un disque.

---

## 5. Déploiement des Services et Validation

### 5.1 Tests de Résilience et Résultats

Une série de tests a validé le bon fonctionnement de la solution :

|**Test Effectué**|**Méthodologie Technique**|**Résultat**|
|---|---|---|
|**Live Migration**|Déplacement d'une VM active entre Nœud 1 et Nœud 2 via `Move-ClusterVirtualMachineRole`.|**Succès :** 0 perte de ping. Continuité de service assurée.|
|**Failover (HA)**|Simulation d'une panne matérielle du Nœud hébergeant le service Web.|**Succès :** Le cluster a détecté la perte de signal et a redémarré la VM sur un autre nœud.|
|**Intégrité Stockage**|Simulation de déconnexion d'un disque physique du pool S2D.|**Succès :** Le volume est resté "Online" en mode dégradé. Les VMs n'ont subi aucune interruption.|

---

# PARTIE 3 : COMPTE RENDU D'ACTIVITÉ JOURNALIER (Commun)

Cette partie rend compte de l'avancement chronologique des travaux.

---

## Jour 1 : Préparation et Premiers Déploiements

### 1. Réorganisation de la Baie (Commun)

Avant toute chose, nous avons, avec l’aide d’autres groupes (notamment celui de Valentin, de Pierre et de Soyfoudine), réorganisé les baies de la salle pour avoir un **câblage propre** et identifier clairement les branchements de chaque groupe.

![[Pasted image 20251219200311.png]]

_Incident :_ Durant cette phase, une erreur de câblage a provoqué une boucle réseau, générant une tempête de broadcast d'environ **650 Go** sur le VLAN. L'incident a été identifié et corrigé grâce à l'intervention de Maxine.

Une fois le câblage fini, chaque groupe a pu connecter les cartes iDRAC de ses serveurs pour travailler à distance.

### 2. Configuration d’iDRAC et Adressage (Commun)

Nous avons configuré les cartes iDRAC de nos 2 serveurs (Switch 6 et 7).

Après analyse du plan d'adressage global (10.202.0.0/16), nous avons attribué une plage d'adresse à chaque groupe, et nous avons utilisé la plage 10.202.6.0/16.

|**Paramètre**|**Serveur Hyper-V (Romain)**|**Serveur Proxmox (Alexandre)**|
|---|---|---|
|**IP iDRAC**|`10.202.6.216`|`10.202.6.17`|
|**Masque**|`255.255.0.0`|`255.255.0.0`|
|**Passerelle**|`10.202.255.254`|`10.202.255.254`|

#### Test de connexion

Nous nous sommes connectés via l'interface WEB iDRAC.

![[Pasted image 20251219200334.png]]

### 3. Installation de l'Hyperviseur Proxmox (Alexandre)

Ici, je me suis occupé du 7eme serveur en partant du haut.

J'ai établi un premier cahier des charges : un hyperviseur principal (Bare Metal) hébergeant une infrastructure virtualisée (Nested) pour simuler un cluster, ce qui signifiait abandonner un des deux disques de 1TO7 qui m'étaient attribués. Je me suis rendu sur la console en ligne de l'idrac et j'ai chargé un média virtuel, j'ai chargé l'ISO de proxmox, j'ai ensuite redémarré le serveur et j'ai pu configurer mon proxmox,

1. **Installation :** Boot sur l'ISO Proxmox.
2. **Stockage :** Installation standard sur les disques disponibles (le RAID matériel n'était pas encore configuré à ce stade du projet).

![[Pasted image 20251219200651.png]]
![[Pasted image 20251219200820.png]]

Une fois terminé, on peut se rentre  sur l'IP qu'on a addressé sur la configuration du proxmox pendant son installation;
![[Pasted image 20251219200856.png]]
Une fois Proxmox opérationnel, j'ai déployé 3 VMs qui serviront de nœuds pour notre futur cluster Ceph.


**Optimisation et Choix Techniques Ceph :** La configuration à 3 nœuds permet de tolérer la perte d'un nœud sans interruption de service (N+1 redondance). L'algorithme CRUSH de Ceph assure une distribution pseudo-aléatoire mais déterministe des données, garantissant un équilibrage de charge automatique. _Note :_ Dans un environnement de production critique, il est recommandé de séparer physiquement le réseau public Ceph (accès VMs) du réseau de cluster (réplication) pour éviter les goulots d'étranglement lors des opérations de rebalancing. Dans notre architecture labo, ces flux cohabitent sur l'interface `eno1` via une segmentation logique
**Tableau des VMs Proxmox (Nested) :**

| **ID VM** | **Nom** | **OS**     | **vCPU** | **RAM** | **Rôle** | IP           |
| --------- | ------- | ---------- | -------- | ------- | -------- | ------------ |
| 100       | PVE1    | Proxmox VE | 8 (Host) | 32 Go   | Nœud 1   | 10.202.6.220 |
| 101       | PVE2    | Proxmox VE | 8 (Host) | 32 Go   | Nœud 2   | 10.202.6.221 |
| 102       | PVE3    | Proxmox VE | 8 (Host) | 32 Go   | Nœud 3   | 10.202.6.222 |

---

## Jour 2 : Installation Hyper-V et Cluster Proxmox HA

### 1. Installation de Windows Server et Hyper-V (Romain)

De mon côté, j'ai installé la solution Microsoft sur le serveur 6.
#### Difficultés rencontrées

1. **Stockage :** Nécessité de nettoyer les anciennes partitions des groupes précédents.
2. **Partitionnement :** Les disques avaient un formatage bloquant l'installateur Windows.
3. **ISO :** Incompatibilité de la première ISO testée.

![[Pasted image 20251219200634.png]]
Une fois Windows Server installé, j'ai configuré les **vSwitchs**. Après une tentative en mode Externe (Bridge) causant des problèmes APIPA, j'ai basculé vers un vSwitch Interne (NAT) pour stabiliser le réseau.

### 2. Mise en place de la Haute Disponibilité sur Proxmox (Alexandre)

Pendant ce temps, sur le serveur Proxmox, j'ai finalisé la configuration du cluster pour la **Haute Disponibilité (HA)**.

- **Technologie utilisée :** J'ai couplé **Ceph (RBD)** pour le stockage distribué et **Corosync** pour la gestion du cluster.
- **Objectif :** Si le nœud virtuel `pve1` tombe, les services doivent redémarrer automatiquement sur `pve2` ou `pve3`.
- **Réalisation :** Le cluster a été initialisé via l'interface web et les premières VMs de test (Alpine Linux) ont été déployées sur le pool de stockage partagé `pool-vms`.
J'ai configuré les Vms Alpines, Alpine est une vm qui est extrêmement légère, et quad on boot dessus c'est un ISO live, donc pas de configuration tres longue, il faut juste faire un 
**setup-alpine**
suivre le déroulement, 
setup de clavier, setup de l’adressage IP.... 
Une configuration rapide, pour une petite VM très suffisante pour le cadre de la SAE
La mise en place de la haute disponibilité était un succès, comme nous pouvons le remarquer sur ces deux captures d'écran,

Toutes les VMs proxmox en nested étaient bridgées de l'eno1 vers le VMBR0 comme le montre cette capture d'écran
**![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAlwAAADOCAYAAADrLdzRAAAQAElEQVR4AeydCbyNxRvHn3OtSdaSqCwV0UYKUdmXiNAiZCtLWbKkSMpS+gtJKpUiIrImSYlslSxlX0NR2ZJ9iWz/852a03uPc+85555771nu4+M57zvrO/ObeWd+8zwz740rUqTIeRXFQPuA9gHtA9oHtA9oH9A+kHJ9IE70nyKgCCgCioAiEHYEtACKQGwjoIQrtttXa6cIKAKKgCKgCCgCEYBAXL58+URFMdA+EPl9QNtI20j7gPYB7QPR2wfidu3aJSqKgfYB7QPaB7QPaB/QPqB9IOX6QAyZFCNAX6hFUAQUAUVAEVAEFAFFwAcCSrh8gKJeioAioAgoAopAkhHQhIqADwSUcPkARb0UAUVAEVAEFAFFQBFITgSUcCUnmpqXIqAIBIKAxlEEFAFFIM0hoIQrzTW5VlgRUAQUAUVAEVAEUhuBgAhXgQIFpGTJklKqVKk0LWAAFsndSNmzZ5e8efP+93mOKP1UB3WgLoHic/XVV0dMv6JtKU80lj2l38tgsQkEQ/oJ/UWPuEfvEfeE2o52pX0D6QcaRxFISwj4JVwQjIsuukj27dsnu3fvTtMCBmABJsnVSRiYMmTIICdOnJCjR49GtVAH6kKd/OEDucmSJUvE9CvalvJQrmgre0q/l8Fg4w87wukf9BP6S7T3eS3/hWMW7Ur70s60t0p0I6ClTz4E/BKuXLlyybFjx+TcuXPJ99QozQkMwAJMkqsKELi///5bzp8/n1xZhi0f6kBdqJO/QuTOnTui+pVtW8oVbWX3V95Qw4PBJpBn0T/oJ/SXQOJrnOhCgHalfWnn6Cq5llYRSFkE/BKuuLg4JVuONmDyAROHV0i35MUAFVImEZSYulAnf0UiDlj6i5ea4ZSHcvl7JnGI6y9eLIVTX+qdHHUiH/pJcuQV/jy0BL4QoH1pZ19h6qcIpFUE/BKutAqM1lsRUAQUAUVAEVAEFIHkQkAJV3Ih6SOfihUr+vBNW15a27SFgPb5tNXeWltFQBEIHAElXIFjpTEVAUVAEVAEFAFFIDoRCHupk5VwXXHFFfLSSy/JRx99JB9//LH0799fKlWqlKyVLFy4sNx2223Jmmc0ZMbR/8aNG4uVhg0bSrVq1QTMU6P8OXPmNJ+tSI1nOZ/Rpk0bmTFjxgUyePBgZzTPfb9+/aR27drGPWXKFLnyyislXbp0Hj8TkEo/3mWfOnWqDBw40HwKI5WKEBOP4VTwAw88INddd13I9bnpppukfPnyQeWTPn16KVu2rNSrV08eeughqVGjhtxyyy1B5eEdmVN899xzjzRq1EiyZctm8uXqHS+53eyrKlKkiCdb6pMaz/U8UG8UgTSMQLIRrksuuUQGDBggf/75p/Tq1UueeeYZ2bZtm3Ts2FFuvfXWZIO4TJkyUrx48WTLL5oy2rFjh4wfP97IJ598Ilu2bDGTR+bMmVO8GhCXyy67LMWf4+sB3377rdStWzeedOvWzVfUeH5M0r///rvpL3feeWe8sNRyOMvevHlzmTVrlnk3cuTIkVpFiPrnFCxYUNasWSMQr3BU5q677hLa6/vvv5fPPvtMtm/fLtdee63cfPPNSS4On0zgFN+ECRPkyJEjMmnSJHNNcoYBJuQddn76JN5zA8xDoykCikDSEEg2wtWgQQPZuXOnDB06VH755Rf59ddfZdSoUTJs2DA5efKk3HDDDfLWW29Jjx495PXXXzelhYi98cYbgqAZu/TSS40/P0xOpH333XfNBAWpQLNVq1YtYZ9IixYtiCYPPvigvP3220aefPJJo80wATH+w7Hr7e6B/8yZM2I/U4G2C3yQypUrC9+VAjdW0dxbSCCtt99+u3HSLvfee68g+LMCJgDNJFo12pUPX6JdKFiwoJQoUYLgiJE8efII2i76yosvvihoDmzh0HAVLVpUOnToIIUKFZLevXvboLBc+aTIggUL5NSpU2bCphAJvQMPP/ywPPvss0KdPvjgA3nllVcEjQgastGjR8fT2DVp0kSGDx8ub775plnsgEmFChXMO8EzrBAH8kCf6Nq1q3kfeXfq1Kljo0TcFVKCBuann34Syn3xxRd7ykgfv+OOO6Rq1aqGjEOqbfszLtB/CUMz5U2OCrr7sne9eW/QoHse4L7hneL9mj9/vuzdu1f++usv2bx5s3zzzTdy+PBhdwwxfQ6tGenRrjJO2fcI0s+z0Ubfd999RjNGPUqXLm3SVa9eXaiT1TSRrly5ckLZ0KTRP6gXD6IuvIfcI043z0HrRj6ZMmUSPm8CPpSnZs2ahqziz3Nz5MghFStWJIt4mjXKSXzqcffdd5tyEYmxgDpRB8pFGFpjwlQUAUUgcATiAo+aeEwm7iVLllwQaYF7gtmwYYOcPXtWIFRbt24VBnsmhe7du8uIESOMFgxNRNu2bU161P685H379pUuXbqYwQL3Dz/8ID/++KOQJ5MOkweTUM+ePaVTp05SsGBBM/CaTBL5adq0qTD4eEfBjzBv/0h0Q6DAiQmGFTKDNgMz+KBFwY9BErL7xx9/iHNVmz9/fkOI0RiwUv/666/liy++MKt4CAr15Vg3gzZhK1eulF27dpmV/apVqwiOGGnVqpXRpEK2MWPb8tsCHj9+XDDlsQigP1n/JFxDTgKelvzS3xN7B8CfugwZMkQ6d+4stBnvGO/MmDFjpH79+qY8TKpM0PijTWYibNasmSxdutRMurQxEWl/iANampYtW5q2hoiy0KFMEFLiRZpAgFjIUa7ffvtNrrnmGm6NgBH9fu7cuab/YvamP5tA9w+LEcIWLlxotJyEu73NfxaEvO+0AR5o6Mlrh1uLjNsKX02HaEGSrR9X3ikbF1JEWXjveF9oK2c5MUnOmTNHIG1o5xkLGct4N7/66iuhj5InAqGiLOS1fPlys1Agb8L8CUTu008/NYQe8oW14fPPPzfvB27qwFh86NAhM4Y686P9r7rqKgEvxgKIH2mIw/MhaZSfcnFPHQlTUQQUgcARSDbCxWRy8ODBRJ/MS4zWgQGHldbPP/8sa9euNWnwR5PicrmMX4sWLWT//v3mC+xMlpdffrmJ5/xhdQv54rlofBgMyNcZx9f9nj17zN4JBkIbzj0TD2HWL9KuTJ52Dxerdv60BittNCcMgODA5EC5GViZLFwul9E8Ehd//PimEvEwE253a8lYtdMmmCjJh3gIWNhVPO5wCUTSex+XJRxMYEyolG39+vXCpMx9pIiz7Giq0CyytxFs6asJvQOUnzDalDZggl+3bp0w+aFhQetDHLSSmC354jlh8+bNM6SEyRyiDBkjHouTFStWCCSEMkyfPt3kBZmBnBFOvEgT+jzvP+Xiipt7K+DC/enTp82iAFMdboQ+zhUMEQgnboR3gEWEzQ+yQXzeA8KtsLABS+v2dUULxvYJwohLvoyHuBEw5soiiHJC7HD7Et5PyDjlOHDgQFD9mXcZUkW+9APMsNzT16gH9wkJYwEklPT0I7B24gU29B3KRV9D85hQXuqvCCgCvhFINsJ14sQJcQ52vh7HxG79icuK3U6kmB8zZswoDDis8NijM27cOLNZGhW2y+WyST1X8kCVbvNAy8EeBU+EBG5YbbL6w2zjcrnE5XIJ9/gRlkCysHuzorZ7uBjgGRwZTClY5syZhdW6k5BBIrNmzWq0WWgXwZeBlQHdpqENbBo2BjsHZiYH4oVbIBTee7jYw0a5mLwgnNwjznvc4RZn2dEeMOmuXr3aFIv+C/62/zrfASKwiOCKQBCYzO09ixfuIV5ODQn5gwlhECnMatxDshYvXsyt2aTdp08f827x7CpVqpi+YwIj6AfSgjYFLTZ9FLM3Y4Nz8cU7YItMf6WPW7fFCzdkwRmGHwSDBYbL5TIHQnyRddoATRjxExLCneXg3vks8nCmdblcTme8e9JRD+vprIP1S+jqfA4aTXBjXMMM6HIl/Ezyow7O9NQB7TlhCPhxRSBktv/hVlEEFIHAEEg2woV2xNcqGU2EL38mBlbg3hMpfxcOTRMvNOY9wq0Gw7tK5AEBIY4VTEze8Xy5Z86caf60DIQNYaLGz1fcSPRDMwi5tJorBkiwAw+nsBrlb5uxWmZyYTXOREOdSMMq2BmfCZiwaBHIhpMkQjAjteyc3kUjZ0kQ/TehdyDQOpCHJVikgYChEeMegsXEi2kSMm5N/qRhL6V9Z7iyD440kSSYE9HUOvsnWj62DthyQlDsPQSBPm3dzjAWH97kBc2Ty+Uy5nbIry/ChZadRaC3Rgc/a3Ljmc5nQV68n2XL5O8K6aGsNh4LKXsP0bH3XJ3xcFuhrGhPIdyY2QN5p33VAT+bZ7RdtbyKQCQikGyEC3Mex43ZS8L+BSZ3Nr4jvl5cBgMGVCYDgGEisnu40MKgwWGAgVCwmZPBlHistOxA89133wlkjsGSMFZybELlPhBhIGLPC8J9IGkiJQ4kCpJrN7GjtWKPCposyggZYw8X9wiTC+3icrnEmmEgXphU7KBOWxCH+N6ChgUS7O0fbjdmNzaIUw60RfQd7p3iPYk5w1Lznsmb94Q9VDw3sXeA8EBk2bJlgtkS0ulyuQRtFWSctEz6aNMwz7O3j3cHf56Ltsjl+kfr0a5dO7n++usJihihr0EWvUkQWl7amHAKy/jA2IBwj5Yaf4R9SVx5L5B9+/bh9AjjC+Y/iBPvhMXHE8F9w/Pwr1SpkuTNm9ds3GefFZi7XP/gx0LH7h2D7FAOq3l2ZxHUf0yftn6UmQWSzYDFBRo/3Cws0PZx7y34874eOnTIBDEuu1wuc6AIk6DFThz/GB/AGxxdLpfZO4YZ0RFFbxUBRSBEBJKNcLEZnhNVqPtfe+01c0KqYsWKwqlFJgXvcjLAEcbGXU5gtW7d2myIJx6aJvYosermW0bs72LA4yQQEwinbiB2mGvYw8XnKBBMj2gMyCMQYVBiBYhwH0iaSIrDnqXMblMiJInBGA0GK1tOE6FFYTKx5WXigIQxsFo/CBf7PjjthEC+EpooGHyZVJhobPrUuvJMCLFT2IPE86dNm2a+R/bOO+/I/fffL2zq955Q0JIwWY4cOZIkYRWO4TNpsjBI7B0ItJBs0qbdBw0aJKNHjzYb5TkkYNNDriCibJa3fmPGjDF7uTjVyHvDBM2+MBseCVf2VGFag6Q6y4P2DiIJOcCffsk4g/kMgkE/xx9BuwvOEHLeFbTY+DuF+JAXb2Jn40BQ2A9FWsYgTu2WKFFCGO/oa8RjbxyLNk73seiD9PFuERaskC8ED0LMHlXn+8oitGDBguY7YMWKFfMsnLyfwfPBgvIwFoADWk3GR0gn9eXEpDMdCxcWbeDF2AuB37hxozOK3isCikCICCQb4aIcDDycQMREgbCSX7BgAUGyadMm4fSUcfz7w6qbk1WQJ04jcnKHIAga5kT2cfXp08d8+wYzIxotNolz9BmSRtzJkyfLE088IZhInn76aWFAkRj8B1bU31k1JiQIBwMx/hAsMHcP/QAAEABJREFUNCjsQ/vyyy/NJmL8ETRifPOHfHBbYSKC4LK/yHliinZzDrhMTGANybVpU+PKKVb6krcwKfB8Jj1Mwo8//rjw0VNIP99KIgx/JhGIDf3psccewzvVhLLzGQfnA2mHRx55RMAaf9rD1zswceJEgQwRB+FdoE24hxRTN+6RsWPHSvv27QVtMnk5SQrtCnaYF4mLQFg4/UiaHj16mE9OoO0hLFKE/gbB9lUeTt6xUCAMPOnvxGVsgKzgj9D24Aw5t1o/rs73iIUKGtCECBf5QFgWLVokvD+YN3kP6HeEIbyHPJt3D0LtzJ/FIlor4iHWTZ/kRCF+COkgRdQHgkd9yI/FA+UjDmMbz549e7ZwgpGtFmi5CbP5cs/ikTx4r8GGONxzypD6MmbYZ9vnko7FLNiyP5JnUxb86XfOscDbTRwVRUAR8I9AshIu/4+LjhhaSkVAEYh9BNhrhbYK8oYmKxJqjJa5Zs2agmkS8x5aaYhWJJRNy6AIKAKhIaCEKzT8NLUioAhEIQIcLsAEjemMjfiRUgW0cmgo+b4aHz5FmwkhjJTyaTlSHQF9YAwhoIQrBRsT1XsKZq9ZKwIRh0Bq9nlMZHyB3hcImN4w0fkKw48wzIOY5zCx4hcJgqYNcyGmPcyAmJwjoVxaBkVAEQgdASVcoWOoOSgCikC4ENDnKgKKgCIQJQj4JVxswGTjZpTUJ8WLCRZgklwPIi+Xy5Vc2YU9H5fLJdTJX0GIA5b+4qVmOOWhXP6eSRzi+osXS+HUl3onR53Ix+WKnT6fHJjEWh4uV2DjQKzVW+ujCCSGgF/CxQczOUbMgJtYRmkhDAzAAkySq758fZ+PJrpcKTIBJVcxA8rH5XIJdaFO/hKwTwUswdRf3NQIpxyUh3L5ex5xiEsaf3FjIZx6Ul/qnRz1oX/QT1yu6O/zyYFHrOXhcgU+DsRa3bU+ikBiCMQlFkgYx7MZIDktw0f40rKAAViACdgkh/BdIY6Vs3mX7yFFs1AH6kKd/GHDd4o4dg6mkdCnKAfloVzRVvaUxi8YbPxhRzj9g35Cf4nm/q5lv0R8YUC70r60M+2togj8g4D++iVcQATB4IOibOBMywIGYAEmySkMTJxG4jta0SzUgboEig3kBkwjoU9RDsoTjWVPafyCxSYQDOkn9Jdo7u9a9l3mW3/eONCutG8g/UDjKAJpCYGACFdaAkTrqggoAopAOBDQZyoCikBsI6CEK7bbV2unCCgCioAioAgoAhGAgBKuCGgELUIgCGgcRUARUAQUAUUgehFQwhW9baclVwQUAUVAEVAEFIHURiCJz1PClUTgNJkioAgoAoqAIqAIKAKBIhCXL18+UVEMtA9oH9A+oH0gmfqAzik6r2of8NEH4raczCoqkYEBLNn7iLW6fR89j0ZctH1Da0vFT/EL53uv/S+0/pfcbReN7aEmRVpNRRFITQT0WYqAIqAIKAJpDgElXGmuybXCioAioAgoAoqAIpDaCEQi4UptDPR5ioAioAgoAoqAIqAIpCgCSrhSFF7NXBFQBBQBRSB6EdCSKwLJh4ASruTDUnNSBBQBRUARUAQUAUXAJwJKuHzCop6KgCIQCAIaRxFQBBQBRSAwBGKOcF18USa5/fqCwjUwCDSWIqAIKAKKgCKgCCgCKYtATBEuSFbpYoWkQsmiwhV3ysLnL3cNVwQUAUVAEVAEFAFFQCRmCBfkCpKVLi5OsmTOKOni4pR0aQ9XBBQBRUARUARAQCXsCARFuDo0vV82fjVeqpS7LV7Ba9xVRr78YEg8v9R0OMnWsg0/m0dzTRcXu6SrdOnS8sMPPyQoRYsWNTjoT/QjcOmll8rs2bPlgw8+CLgy+fPnl7p163rif/HFF1KlShWPOy3dJAU/8Clbtqzcdlv8sQ7/YKVly5bBJvHEDyWtJ5Mk3Nx5553y/fffS65cuS5I/eCDD8rcuXMlXbp0smjRIqlevfoFcZwe1apVk3nz5knnzp2d3uae9B999JG5d/6MHTtW6tWr5/SKuvs2bdrEG58XL14sH374odSoUUOS+i+Y/jB+/Hi54447zKPatWsnY8aMMe3FWPDGG2/IPffcY8IKFy5symkcMfzj3R7gMGzYMClZsmSy15p3YubMmRfkGxThIvX233fLc+2bS5aLMuOMCGHPVro4lyxzk60jJ06aMnHFjT/hxjOGfpYtW2YmAyaEJk2amJrde++9Hr/NmzcbP/2JfgTq168vX331lWTNmlWuv/76gCpUrlw5qVChgicug+vXX3/tcaelmwDxuwCSOnXqSLFixS7wD8bj6quvlocffjiYJJ64oaT1ZJLEm2+//Vb27dsnYOedRe3atWXWrFly9uxZ7yCf7vvuu0+Y2JiEIGnekbJkySLNmjXz9o4J908//eQZk+kHa9askb59+wokJ9gKBtMfiMuia+nSpVKrVi1p2LChsGCjT7dt21ZWr14tL774YsDjSbBljdT4zvZo3769HD9+XLp3755qxQ2KcJ13F2vzL7/Kxq3bpVurRm6X7/8tH6gt097+n6z87AOZMLSvWyNWykQseOUVsmrmGOncsqF8/HpfWTJ1hHR5tKH06fSYTH7zJZnz4evSqE5VE5efm4teY/zJa9aoV6Vu1bvwvkCO/3VKlm74RSBZzkDc+BPu9I/1e1b0vGilSv2DO/VlosaPsC+//FK6desmI0eOlOnTp8vbb78tBQsWJJpZtXbt2lWmTZsmkydPliFDhvhc5ZrI+pMqCDBI0k6QrgYNGsR75q233ioTJkyQb775xrRnmTJlBLL16KOPSokSJeTNN9808VnNoeFCm8BKz3i6fzJkyGC0FVWrVo3Ztk8IvxtuuOGClf2UKVMEzUpLt1bqrrvuErQ5Xbp0cSMlcvPNNxsNBXh//PHH0qFDB+PPT+PGjeXdd981k9rw4cPNJAdBfu211yR79uxGu8Ake9VVVwl+kN8ZM2ZI7969hTYgj//973/y7LPPCmUYNGiQiedMS5zUFMoHuXI+kzoUL15cJk2a5PRO8P7KK6+U6667Tsjr559/vkAbdv78eQGvFi1ayOWXX55gPs4AtG8jRowwfX7ixInSvHlzZ3DE3v/6669mPD127JhQBwqaWJ/6/PPPBc0UfaVp06YX9AdffY48EfJfsWKFnDt3TiBfO3bskAULFsjhw4eFcrz//vvCGLFlyxaixxPS+sL3mWeekbfeeiteXIg0/nhC7Oi7aNY+dGvy6Cf4Z8yYUZ5++mkzPjHnMKfcdNNNBIVV6I8LFy6Uyy67zFOOxNrjs88+M+0xatQoMz+CIe8DiXlP0Rp+8skn5l1HEYK/twRFuEjMCuXFN0dLnSp3yo1FCuMVT64vXEAwPb426mOp2qyzrNm8TZ5q1djEOXv2nGTKmEH2HzwsD3fqLcPGTJGWbnL20y+/yYMdesn4GV/JYw/VMXFzZc8m77z0jHw65xtp8MSz8vyQ9+Slrq3l2gJXmnDnz/JN2+Xov5otpz/3+BPOfVqRP//8Uxa71ddOkxKrSwgXYQxyrH4ee+wxM6FkdL8QvNjgw+oHNTQTDqsil8tlXhYhUCXVEbj77rvlwIEDsm3bNpk6dapUqlRJMmXKZMqRM2dOYWL+9NNPBS3C8uXLpU+fPoZEYMJZtWpVPFJAIgZv8uQeQQvmcrmMyScW2566JoQf9U9I0AYwSbHogCCBNVdMXY0aNRImGbQ/EOCLL75YIGWQJ96bXr16mfY4ffq0IRNMcpACBnjes7///luYLCFXTG5oHykHGiPevYEDB5p3DiLiTEuc1BQWXVdccYVQJvtcxgS2Mvz+++/WK9ErhBVzOBM/WjHIr3cCcCZOjx49vIMucDNWEW/JkiXGZM7k17FjR8mbN+8FcSPdI7E+RdnpDxCTVq1aCf3O2R/27t3rs8/ZseH2228XtJTkA6nIly+fPP/882b8sFpbtG08gzhWEsOXRRtEgnITH5LBAg+SwRaXnj17mvGHvs34Qz+Oi4uTihUrmj7EfINAynhvyCOcctFFFwmLKvoS5aBeCb3jhDNvsnh44oknhPK7XC5zJaxTp05mwfrAAw8I/dGSTcKcEhThcrlTusQle/btl/c+niH9Ordy+8T/v+nnHVLqvkflm+WrDbFauHSl5MtzabxIc75bbtwr1m+WDOnTy+fzFxv3yg0/Se4c2c195XKl5JR7wBr36Wzj/nHdJln841q5r+qdxq0/iSPAy8HkzGRATCZWBjXuEdvJmBTwL1KkCN6CpoOJnYGeQZIVCWl5cUwE/UlVBOrVqye0JQ+FOKxfv16s1oF2OXLkiKBtIYxVJS8+EzrxfQmrNNoawk047c1+HNqa+1hr+8Two/6BClhjfpgzZ45JgpYAsz4rYhahhEG2ChUqZAgy5PXUqVMmrvMHkoUJgwmTtvzll18ELZCNs3HjRiFf6w7nlT7FZG0Xbkzm7MdCWxVIucDFaj2IzzjD3lK0fLgRl8s9o7gFzQmTFJMz/gkJfZutExCtgwcPmr2N+BUoUCChJBHjD+nh/UTzCRlKrE/ZQqO5po9Yt72CbUJ9jrEazff8+fNNdPoUpI39eFwZJ9CY+zLjgmVC+K5du9Zox+gDZMxCgUUEWjLGJOq0bt06gowmOHPmzMI+SDR6efLkkUceecRoe1EG9HWbVU3EJP4kNRljHwsGBGwhtPQl8gukPb777jux7zXmSasdg4gyNkBgGZOxIpGntwRFuJyJ3584Q9KnTyetGv6jkbJhmTNllOc7tJB544aZDfYjBzwrcXEuG2yux46fMFc0XtwcPnqMi5xza8DSpfunSJflzCF5L81l8mCjPlKhTEnJnzePias/iSPAywaZ4qXAnIgpkYnVpmKwsveHDh0y+4Nww/IxN9IhkdGjRwurnmgY0Ch/LAkmFjQgqONpCwS31RLkzp1bGMxsnU+cOCEMAtbt67pv3z7B1MAAyQRKfpAw4sZa2/vDjzoHKmDNIsQZHzerfAZYyJbL5TJ7ldCKOc2NzjRoBBjgmXRoTybGuLh/xjzikRfXSBG0F5CgbNmymU3WTDaYtgMpH5vD6VNoyqgrGnbwQgvgnZ5+/OqrrxrNIXu6vMOd7oceesiYNCGm5Mv45MTQGTfc984JnsUM7Y0mFKKSWJ+y5WZstvfOK/0koT5HH9u5c6ch/jbN9u3bjUaM/b7ly5cXCAHkC824jWOvieGL5or+QNzKlSsLJIN72pkFG+2B0DY5cuQQyDV9nTkFckNfGDdu3AWmZfJIDWF8hBwhaOIgXZgGWYAG0h6MsbacLFLTuxVGuHk/aBPuEef8itvKf2+69Qnwes5te+877ANp3fA+yXrxRZ5UTzSpL8WvLSR123SXYtUbS8vuL3vCgrnZd/CQbNux0+RBPla69h8WTDZpNi5ki5eBlwBzIgSMwdICcskll9hb4cVg8sCDjsKAQId0iq9VFvFVUg6B+++/3+xTcbYD5h32DUmStaIAABAASURBVCD79+8Xq8GkFOnSpTPqe38TFhMmqzn6xq5du8SuSmOt7f3hd+bMGWBzLwj/GwadeJrAf3/AGrLwr9NccGOix8H78fLLLwtk+N133zUmQ8w6hFlhUMbMwh4w9tnRrkxONjz5rsmXEyQJ8yFaD8YRzIJMNIE8gcn89ddf92wap77sH0Tr5Ysg0S+3bt1qzOBoCnw9A40JeWBWxIxFnmhlfMWNBD/nBM+7y14sNH2UzV+fIk5iklCf49129iuIb4kSJTxZgdc777wjK1euFDSOngD3jT982VfG3mDGHwiUPYnH2MHWBtrDKeyxc2crWFTQ7NasWdOcfsW86asPEDe1hLZhSwZzHxiF0h5Hjx71KC0oPwSUq7f8N9J4hwTgxsw3e9ESafXQf1ou9lj9tP03OXbiL8mUMYM0rF3FDGiYDgPI0hNl3uIfJXu2rJ6N8kwmg3q0FzbSeyLpTaIIoLlgRcVAac1SNgEdjImZDbvcY88nDJLGCofJBDcbrdlEz71K6iHAYMTkzZ4r51Mh0qjuOfHEkXrIMgMscdhfA1kmDuSatsXfWxjwMWMxiTq1nrHU9oHgB5Fg8mHiACMmCtvvcRNmMcS0BhljVU8YpkMmJ1bIt9xyi9noTjhhrO4hcwzCf/31l9g8MCkRBxJDPAgZ2ueMGTPivECcaS8ITEUPxhHGASbtQDfLMyGDC5oUZ1HRrp48eTJBDcfgwYPNgQPMX8509h6NEaZONET4sfHb5XIZLTxuhP1011xzDbdGWrduLZTdONw/aDbY2+e+Dev/xPqUr4I5+wPYsheK/kRcZ5+jH/NJD/wRNF7sr2L/EW4Eskq/t30RP8QfvpjSWaChsaIt//jjD5KZU6tgagkcG/UhM5QPkvnKK6+YeIxNED00RYESd5MwhX74JAR9jfkv2PZwFmnDhg3mcx+MO9nc2mDmXGe4vY+zN0m9DnpvvGS/JKsn+dhPvpRq5W+XSW+8KB8MfE7mfrfc7OUaPeg5T5xAbg4cPiLtXxgsTevVkGnDX5Zxr74gZ8+dM5vwRf8FhAB7RFgx0gm8XyxWHGzCxPzB5MwmYTLldApHhnFzzyZHtGOExYpEQz3Yfwch9iZclB0/tFPHjx83JpjHH39caF9W/px0Y1CjfdlfxIrUTvikRUhHfFaqTKb4IbR3rLR9IPhBijCZUW/2tUCAWPW6XP9sgYBM0f+ZLFjBM5FjxkFDxb45CCqkF8xmzZpljtljnsBkgmzatMloEVhBM5gzEENwSY+pno23XCHWmHhpA6cwMdm0TJDOsNS8nz59ukAM0Zqw98z72Wj2CLOCVgPtIoc2MGF7xwczuy/MO4wJffz48QI59Q7DDc5cIX60GSYdyMVTTz1lTkMSxqEGTFncI2jaIBLcIywwmWi5D6ck1qd8lcvZH9gOABZ82sHZ5yAy7Cvi/bZ50H8hZC+88ILRmNMH6ccvvfSS0C9tPK7kyTUxfBl/6I8s3IiL8Dw2nPfr108wmffv39/0fcYaNF+Y2yDT9He0kwMGDCBZqgv9wPZT9mOxwd0eNAq2PZyFp16QTTR+zJ1g7HL9M4444wVFuN4cO1U69HnVmV6OHj8h5R5sKzVbdjX+S1atN+6HOj4vjTv3MRviKzRqL0269JXfdu81JkK0X0Te4taEYSrkHuFE4821mnFrBDenFxu06ymNOveWHgPfNv76Ex8BvrnFqmbPnj3xA9wu/Jh03bfx/u/YsUNatGgh9erVM0ddGeiIgCqfY7tMBkzgrAZ50QlTST0EGNTQXEGGvZ/KPgo0LYQxeDDBsIqtWLGi+V4X8RlgcTORQ8DYy0eehCGcsmPQ3L17N04jsdT21DUQ/FiFgx0biPk8SosWLQQSBiCjR48W9rtgCsHNKpjVusUbooE/wiSDJph9MZAJSBz+aCU4zUhZaBMmG9qE57CBnmdADnlHOd3IREU6xDstfuEQJktwePLJJy94PFoNxh6noGkFVxYCFyRwexDGaU33rZCe8Yt7K2BHftPdRM/62SsmXDTwCG1GXHAHczZvE49+vWDBAm6NoMmFQBiH+6d58+aCqdN9m6L/KRvjZ2IPSaxPUSdLgMjDuz/46nOM4/Q1p/aIcQKyQ9/lVB6LNU7aMY6QL9pC8OY+EHzHuwkx8b0PT1BW2h6tI88iHnnSf+jXaMXo99QrHIt42oNyW6FPU1aLA2UNpj3oxxB90nEwAS1wrVq15P777zeHBnjPCXNKnNPxz73+xgICmGBpfAYfVM+xUCetgyKgCCgCioAiEK0IxCThWrjqp2htj2QrN6ZCVuac/PFlBki2B2lGioAioAikFAKaryIQQwjEJuFauTmGmihpVcEkiGrZmkecuXibl5xheq8IKAKKgCKgCCgCyY9ATBKu5IdJc1QEIhIBLZQioAgoAopAlCCghCtKGkqLqQgoAoqAIqAIKALRi0BsE67obRctuSKgCCgCioAioAjEEAJKuGKoMbUqioAioAgoApGJgJZKEVDCpX1AEVAEFAFFQBFQBBSBFEZACVcKA6zZKwKKQCAIaBxFQBFQBGIbgbjrMh8TlcjAgK7Gn7RQyWf+tEes4aDtG1q7Kn6KXzjHBO1/ofW/5G67aGwP1XDRalEgWkRFQBFQBBQBRUARiF4E4nbt2iUqioH2Ae0D2ge0D2gf0D4QQB9QzpBE3qQarugly1pyRUARUAQUAUVAEYgSBJRwRUlDaTEVAUUgShDQYioCioAi4AMBJVw+QFEvRUARUAQUAUVAEVAEkhMBJVzJiabmFQgCGkcRUAQUAUVAEUhzCCjhSnNNrhVWBBQBRUARUAQUAZHUxUAJV+rirU9TBBQBRUARUAQUgTSIgBKuNNjoWmVFQBFQBAJBQOMoAopA8iEQMYSrdOnS8sUXX8iwYcOSr3YxnlObNm3khx9+kIoVK4rzX5UqVWTatGlOrwTvGzRoIHny5EkwPNSAxo0by9ixY/1mc+utt5q6UB/k66+/lnHjxkm1atX8pvUXIdAy+MsnNcOzZs1q3gfa2Pu548ePlxdeeMHbW91eCIAdfSmU98MryzTlVPzSVHNrZVMBgYggXPfdd588/vjjsnTp0lSocmw94tdff5Wnn35asmTJkqSKtWjRQnLmzJmktCmRqEKFCgL5ZrCfMWOGPPnkk9K8efOQHgVBadq0aUh5+E6ccr7Hjh2T/v37C+1TsGBBz4Nw015Dhgzx+OlNwgiE+n4knHPaCFH80kY7ay1TB4GgCFe6dOmka9euRnsyefJkYdDPlSuXKelDDz0kI0aMkAEDBsioUaPk008/lVatWpkwfm6++Wb58MMPZcKECfLxxx9Lhw4d8DaydetWad26tRw8eNC49SdwBLZs2SKbN2+Wjh07JpioVq1aMmXKFIF40AbFixc3cZnQ+ftWffr0ESbyxYsXS8aMGU1YjRo1jMYJzRMeaMGWLVsm2bJlk8Ta8vPPP5d27doJGir6BGmdQp9Bi0lfcvo778+dOyfbtm2TSZMmGTLZsmVLueKKK0wU+ht5EIYWr0uXLhIXFyfPPPOMvPXWWyaO/eE5+HtruB588EH57LPPZMGCBfLaa69J3rx5TZKE8jaBYfj59ttvZc6cOdKrVy/z9Pz585t3inaDkN1www3mnaJdp06dKrSziej+oc7vvvuufPDBBzJ8+PB4Ye7gNPM/kPejbt26ZkwCR7SqtWvXTjP4+Kuo4ucPIQ1XBAJHICjC1bZtW7njjjuECbBhw4bicrnMhMjjzp49KzfeeKMhY48++qi88sorgpYCzQsrciY2TEuNGjUyk2P9+vUFcxZp169fL6TnXiU4BCAuYM1ka4mUMwe0RT179pQ+ffoIk/C8efNk4MCBhqT07dvXRCVs9OjR8ueff0qpUqWMH9cNGzbIbbfdZtxly5YV3DwvsbakHW+66SZDDCBFJvG/P926dTOEjSvx/vVO9LJp0yY5dOiQ6XdEhNCfOnVKIHMQ+kqVKgkECnM0ZaWvES979uxSpkwZ+eSTT3B6hP4LuR88eLDJgwDKwzWhvAkLl0Aur7rqKlPWHj16CO0HEaOer7/+ukBwadeXXnpJnn/+eSlcuLBcfPHFAhHt3bu3eVchbGiRM2XKFK5qhO259NfE3o9y5cpJp06dzJgEjmPGjBFwK1iwYNjKHEkPVvwiqTW0LNGOQFCEq2rVqsJK+vDhw4IWYuTIkYIJCA0DQDBhowXhftWqVWZSRzNBnOPHj5vVOmGoqYmHpgS3StIRcLlcsnfvXhk9erQ899xzF2TEap0Jet26dSYMDVfmzJkFAmU8HD8rVqyQkiVLGp8SJUqYvVdc8eC6fPly097+2vKbb76RX375hWRy/vx5I5AiCBGT299//23CAv3Zv3+/XHnllYImCo3be++9Z5IeOHDAECr2ea1du1boV9wTeM8998jPP/8srNBxW6lZs6YsWbJEFi5cKH/88YchnxDQxPK2acNx5V2DMIDbddddJ6+++qopBu8UOE6cONG4V65caUzytDeTJG3UsmVLKVSokIATiyWIqomchn5crsTfDzS533//vek7wIJGkX5xyy234Ezz4nLFDH5pvi0VgPAjEBThYlWNNoCNqAiTPCaoAgUKmJr89ddf5srPmTNnuEj69Okld+7cwsRhPP79wY0W4l+nXkJEYMyYMQZr7/1OtBlEmfZCILo5cuQQtCbejyQMEnz55ZebvBYsWCBM8hBq/DE5BtKWaKRs3i6XSzCFYWbEHyJgwwK9XnbZZYY0YNYkDZoz6oK0b9/eY25E+1OxYkWiSOXKlT0E33j8+wMeR48e/dclsnv3bkO8/OXtSRCGG+q1c+dOYU8b7w1FoB0oMxhYufPOOwUT8ZEjR4xmy+VymUMomP+dJnzSpzVJ7P2wmFpMwI93xLr1KqL4aS9QBEJHICjCxR4r1O1oKpxitRkJFQcNhTe5wo1GLKE06h8cAmgcMYm1aNFCOOFmU9Nm7Kdzthf3VjNi43H97rvvpFixYoKZBTPv6dOnZceOHVKxYkVhAkKLkpS2PHnypDH7QRAwM/OsQAWyCOFiTxiaB9JVr17dmDqpB4JWB3/Ma5hCMath1pw5cybe8QQ8nPhQJrR9/vKOl0kYHCxgEPto2gENHvV3yrPPPmui8E6+/PLLUqdOHWEvF+ay22+/3YT5/Ilxz8TeD8YiZ/XZp7hv3z6nV5q/V/zSfBdQAJIBgaAIF+p29s7YAYrPD7CJ3l85FrrNN+wrQetAXMwcTHKYnnCrJA8CEKK5c+eKU8s1a9Ysufvuu6Vo0aLmIVdffbUMGjTI7PNhHxVi9/aw0meiZpLGJEyCNWvWmPzQouBOSlti0oJcY/J85JFHBLMgeSUmaNYwg/Xp08eYNtFE7dmzR3788UdzotWmJQ6bnnFjUsR0ihYW86glUYRZmT17tpQvX95o+DJkyGAOgTRp0kT85W3TR8qVduA9ZO8eZcKMyD4uNtKw6hfdAAAQAElEQVRjDuOQBO8cYWguIWtOzR7+aU18vR/0B/b1FSxY0MCByRnCxeLDeOiPBwHFzwOF3igCSUIgKMLFKcTVq1ebk0/cP/bYYzJ//ny/D0arwCZe9pTYU4qQt0WLFpm0bKZnQufoPtoV7jGfmMDI+ImaUrCRmgnDFphPbbDJvV+/fub0KCfcGDgx7UG2FixYIG+//bYxQZEGooL5kH0tuCFeTOLs38Ltry2Jk5Bs3LhRMENDDJxldMaHSND+9BPIPfsEOWVn46DBQeMFoUBzw6EA9qjZcDRh+DGRWj/nFbMoe8A40UgdIaJDhw41UfzlbSJFyA/twGLn4Ycflo8++kioE1oINJO8oxDtF198Ud5//33zPTNO33EAIUKKH7ZieL8f9Ic33njD7OWjT3HAgAMVLD7CVsgIfrDiF8GNo0WLeASCIlxM0Jya4nQhpiHMFEze1JLN9Jxc5B5hQy+mDj5ZgBtNCYSKU4qcHsPcgT+CP3GdYrUWhKv4RgDS+9RTT8UL5HMBmOFoIxvA5EvbcHoUrDn+bsO6d+9uND58PgA/PqVAO+zatQunQIpx077Gw/2TWFvSbjzPHc3851k80zjcP3wyBK0M+2TcTs9/iB7PcQoaVFsuGxFtGeT9gQceELRbkH78bDjPIw8nYcfPWQbMqZSTeJyW5RMUpCefxPImTriEd432dj4fctWsWTNBQ0fbOj+GCs4QMk5yUlfvtM58YvWeOgfyfkyfPt2cAqVPsQ/O2XdiFZtA6qX4BYJSOOLoM6MVgaAIV7RWUsutCCgCioAioAgoAopAOBFQwhVO9PXZioAikOwIaIaKgCKgCEQiAkq4IrFVtEyKgCKgCCgCioAiEFMIKOGKqeYMpDIaRxFQBBQBRUARUARSGwElXKmNuD5PEVAEFAFFQBFQBETSGAZKuNJYg2t1FQFFQBFQBBQBRSD1EYjjT4Go5DN/EkVxUBy0D2gfiKA+oONSPu2P2h9jpw/E/fTTT6KiGGgf0D6gfUD7gPYB7QPaB1KuD6hJMfW1ivrE5EJA81EEFAFFQBFQBKIEASVcUdJQWkxFQBFQBBQBRUARiEwEAimVEq5AUNI4ioAioAgoAoqAIqAIhICAEq4QwNOkioAioAgoAoEgoHEUAUVACZf2AUVAEVAEFAFFQBFQBFIYASVcKQywZq8IBIKAxlEEFAFFQBGIbQSUcMV2+2rtFAFFQBFQBBQBRSACEIgIwnXZZZfJkCFDZOnSpbJw4ULp2bOnpEuXzgGP3vpCoEOHDrJp0yapWrWqOP/VqFFDZs+e7fRK8L5hw4Zy+eWXJxgeakDz5s1l6tSpfrO5/fbbTV2oD7JkyRKZNm2a3HPPPX7T+osQaBn85ZPa4cnRvqld5kh6nuIXWmsofqHhp6kVAW8EIoJwvfrqq3L8+HGpW7eutG3bVkqVKmWuov/8IrB9+3Z57rnnJEuWLH7j+orQpk0byZUrl6+gsPjddtttUrx4cWnatKkhak8//bS0bt06pLKMGTNG7r///pDyCFfiUNs3XOWOlOcqfqG1hOLnAz/1UgSSiEBQhAutU48ePeTLL7+Uzz//XIYPHy65c+c2j27SpImMHTtWhg4dKhMmTJA5c+ZIu3btTBg/JUuWlClTpsinn34qM2bMkK5du+JthLgjRoyQvXv3Gi3HypUr5cYbbzRh+pM4Aps3b5aNGzdKt27dEox43333yaxZs+STTz4xbWCxhejmz59fBgwYIBCvNWvWSKZMmUw+tWvXNm2B5gkPtGDr16+X7NmzS2JtOX/+fOncubPRVtInSOsU+sx7772XqAbz3LlzsmXLFvnoo4+EVTZl489bkA/97e2335aZM2caLV737t0lLi5Onn/+eRk5ciRRPMJzevXqJd4arsaNG8u8efNk+fLl8s4778gVV1xh0iSUtwkM008g7QuZ5J2aPn260QrWq1cvTKWNvMcqfqG1ieIXGn6aWhFwIhAU4erYsaPcdddd8vDDD0udOnXMRMeERoZnz56VW265RSZOnCiNGjWSF198Udq3b280L2hQmNiYEJn8n3zyScGUhZAWovbbb79xa+TWW2+VVatWmXv9SRwBSDBYox20RMqZoly5ctKnTx959tlnpX79+vLVV1/JsGHDTNthuiVuDzeJhvDu27dPLMEqU6aMrF27VrgSp3z58rJu3TpDlBJrS/pBiRIlBFIDYSKtFTRxEDZIFPGsf2LXDRs2yMGDB+XOO+800SD0J0+elHvvvdc8o1q1auYK4ShbtqxHW5cjRw6h7pMnTzbp7A/50C/79+9v8sCfcnFNKG/CwiX+2pf3ES0g7xREC5JJ3QoXLhyuIkfUcxW/0JpD8QsNP02tCDgRCIpwsZ/m448/lkOHDglaCDQNVapUMZM3mTJhf//999zKDz/8YCZnNCjEOXr0qHzxxRcmbLvbDLZ48WJhYjYe//5cdNFFgtaFuJCzf731kggCLpdLdu/eLRCmfv36XRATIrZw4UJZvXq1CXv//fcFnCFQxsPxs2zZMg/hgvSOGjXKmHeJgqmPfVWBtCVarm3btpFMzp8/bwQCBnnDZHzq1CkTFujPn3/+KVdffbXRREEI33rrLZN0//79MmnSJLPPi/rRr+ijBLIg2Lp1q7BCx20Fovbtt9/K119/bTSqYAZhRcuVUN42bTiuLlfi7YsmkvpQd8rHO4ammPbDndbF5VL8QukDLpfiFwp+mlYRcCIQFOHKmTOn2S/EpmYEbVbGjBmlUKFCJs8TJ06YKz9Wg5E+fXq59NJL5fDhw3h7BNJGftYjh1sjgUYE8xCaNJvehsfMNYUqApHKkCHDBfudMJPVrFlTaC8EjRG4FyhQ4IKSQKggwZAP2m3u3LlStGhRQ6jx/+abbwJqSzRSNnOXyyVXXXWVMTPif+zYMRsU8BVzJuQqb968Jg3mROqCYJq25ka0d/YAQfXq1T0E3yT69wc8IPT/OmXXrl2GePnL28YP1zWx9uVdcpbryJEjQhs7/dL6veIXWg9Q/ELDT1MrAiAQFOFiwsT8dP3114tTrDaDDH0JGgpMSc6wHG6C9ccffxgvTI680GgrunTpIkwYJkB/AkYAjWPfvn0N4brkkks86SAq7J1zthf348aN88SxN2jCbrjhBmM2xpx4+vRp+eWXX8wpSCZwtJb+2tLm5bxCxNHE5HUTJsyJzjB/9zXdZDFPnjxmv9aePXtMdLRz1MFKpUqVjP/06dOldOnScu211woEkT1rJsDxAx5Zs2b1+EDmyM9f3p4EYbpJrH1zuN8lZ7GyZctmSKTTL63fK36h9QDFLzT8IjW1lit1EQiKcLHxGtOQHeD5/AAEzF+RMd9AAtA6EPeaa64RJrkFCxbglMGDBwv3mCuNh/4kCQEIEQcaWrVq5UnP3qbKlStLsWLFjB+arTfeeEMgHWfOnBE0iXajPJoSyHODBg3kxx9/NPFXrVol5McnO/Dw15bE8ZYDBw4I5ma0UY8++qjHbOkdz+lGs4am83//+5/ZDI8mCtMpZs9OnTp5ohKH8uKxY8cOYzplTxbxMK3h7xT6cIUKFQQc0Aiyt61ly5bGLEuahPJ25hGue1/ty+EV9qXZPVuYTFncLFq0KFzFjNjnKn6hNY3iFxp+mloRCIpwvfnmm7JixQqBGLHR/fHHHxfMTv5gZMIlLvt37ClF9ppwUgzzFZub0XxgIrIS6Hek/D07rYUPGjTInCS09Wav3CuvvCIDBw40p0f53hlkCtMeZAsCNXr0aHNKkTQMqmiHMB/iJu7NN98smBtxJ9aWhCcmnHJkrxkEGy2Mr7g8nz5AP+GUI/sE2cxu46IBReMFcfrwww/ljjvuMN9us+GYFfEj3Po5rxAR9oCxPw0tHkQUfIjjL2/ihFu825d2Yt8jByGoM/vR2DQPeQ53WSPx+cHhF4k1CG+ZFL/w4q9Pj24EgiJcTNB8QgAzD99J4tQbEyQQQMLYqMw9wsZoTD58sgA3n3rg+DqnFDFb9e7dG2+jWSCet6A9MxH0J0EEIMCcuHNGYH8ShMOJH+SFtuH0KG0AwbJpON0GwYII4ceASlvs3LkTp/l8Am7a13i4fxJqS3eQMT+iVeMeGTNmTLxvYHHCEQ2Tt9mYTzTwHKdw6tCWi7wQTIKQ91q1akmzZs3MCUX8CEOoG3k4P7bqXQbMqWz+Jx448QkK0pJPYnkTJzUl0PblJCaaLTDhkx18MDY1yxmpz1L8QmsZxS80/DS1IuCNQFCEyzuxutM2Alp7RUARUAQUAUVAEQgMASVcgeGksRQBRUARUAQUAUUgMhGIilIp4YqKZtJCKgKKgCKgCCgCikA0I6CEK5pbT8uuCCgCikAgCGgcRUARCDsCSrjC3gRaAEVAEVAEFAFFQBGIdQSUcMV6C2v9AkFA4ygCioAioAgoAimKQFyRIkVERTHQPqB9QPuA9gHtA9oHtA+kXB+I4wvefmXXLvM35zSe4qB9QPuA9gHtA9oHtA9oHwi+D6hJMUUViJq5IqAIKAKKQHIioHkpAtGKgBKuaG05LbcioAgoAoqAIqAIRA0CSriipqm0oIpAIAhoHEVAEVAEFIFIREAJVyS2ipZJEVAEFAFFQBFQBGIKgTRHuGKq9bQyioAioAgoAoqAIhAVCCjhiopm0kIqAoqAIqAIxBgCWp00hoASrjTW4FpdRUARUAQUAUVAEUh9BCKCcF155ZXy4Ycfyvz58+MhEBcXJyNGjJAlS5ZI48aN5aKLLpLXXntNFi1aJLNnz5Z27drFi59UxzXXXCPjxo2TxYsXy7Rp06Rq1apJzSpV07Vp00Z++OEHqVixojj/ValSxdTD6ZfQfYMGDSRPnjwJBYfsT7uNHTvWbz633nqrqQv1Qb7++mvTJtWqVfOb1l+EQMvgL5/UDk+O9k3tMifr80LMTPELDUDFLzT8NLUi4I1A2AlX0aJFZdiwYYbseBcOIsBEXK9ePRk/frw8/fTTJgoT6LPPPis1a9aU6tWrG79Qfl555RXz/Lp168qoUaPkueeeS1ESEkpZvdP++uuvBpcsWbJ4BwXkbtGiheTMmTOguKkRqUKFClK6dGlhsJ8xY4Y8+eST0rx585AeTd9p2rRpSHmEK3Go7RuuckfKcxW/0FpC8QsNP02tCDgRCIpwpUuXTrp27Wq0J5MnT5YhQ4ZIrly5TH4PPfSQ0UYNGDDAkJZPP/1UWrVqZcIyZsxoSMHIkSMFId1NN91kwv766y8zua5bt8647U/27Nnl1VdfNc6BAwfKPffcI2hu0HD9/vvvsmLFCpkyZYrce++9Js7LL78sL7zwgikD2qqPP/5YKleubMKaNGkiH3zwgckPQsVEDtEgsFSpUpI5c2YZPny4/PnnnzJz5kxZuXKlJ1/iRKiYYm3ZskU2b94sHTt2NG5fP7Vq1TJYQTzQJBYvXtxE69+/v+TLl0/69Okj4IGGj7YisEaNGkbjBOHFDfldtmyZjRj0kgAAEABJREFUZMuWTW6++WajkZwwYYKAc4cOHYhi5PPPPzeaRzRU9Anj6fih7SHY9CWHd7zbc+fOybZt22TSpEmm37Rs2VKuuOIKE4f+Rh6EoY3s0qWLoAl95pln5K233jJx7A/PwR+C7tSyPfjgg/LZZ5/JggULjMY0b968JklCeZvAMP0E0r4sFGgH2pe+X7t27TCVNvIeq/iF1iaKX2j4aWpFwIlAUISrbdu2cscddwgTYMOGDcXlcpkJkQzPnj0rN954oyFjjz76qKA1QkuB5qVixYom3WOPPSYIRAlTFulYQUF0uHfK4cOHpXv37sYLMvHTTz8JZID4xtP9A/EqXLiw+06ESfq2226T559/Xh555BGjsYJEEHj+/HmB4EH2KBsEDpLA34xCw7Z7926ieYQ/WXDdddd53JF8A3EBa0iVJVLO8qIt6tmzp/Tp08eYZefNmyfUH5LSt29fE5Ww0aNHG8IJAcWT64YNGwRMcZctW1Zw8zxILwSmUaNGAqGpX7++2PakH4A1ZBtSRFor3bp1M4SNK/Gsf2LXTZs2yaFDh0z/IR6E/tSpUwKZ4xmVKlUSCNQXX3xhymq1dRD2MmXKyCeffEIyj9B/W7duLYMHDzZ5EEB5uCaUN2HhEvBOrH3LlSsnnTp1Mu0AsRwzZoz07t1bChYsGK4iR9RzFb/QmkPxCw2/tJNaaxoIAkERLvY2TZ06VSBDEBwIDCYgJm8eBnFCC8L9qlWrjOYBzcSxY8eMiQ4ixESIJsVO9sQNRHLnzi0nT56MF5V80bhYz7Vr18revXuNk4n6sssuM/f84A9h4P7bb7+VPXv2GIKYI0cOQcuGv5Xjx48L5bTuSL66XC5T59GjRxtTqHdZ0XZQX6tBRMOFRg8C5R0XrWHJkiWNd4kSJQRSxRUPrsuXLxfaG3zmzJmDt0CAaXO0XsbD/fPNN9/IL7/84r4TgewikCLIG+Tg77//NmGB/uzfv1/Y54cmCo3be++9Z5IeOHDAECr2edH2lIV7AtGI/vzzz8IKHbcVzNDsCVy4cKH88ccfhnxCQBPL26YNx9XlSrx90UR+//33ph0oH+1CvW655RacaV5cLsUvlE7gcil+oeCnaRUBJwJBES60B2gD2NSMMMmjdSpQoIDJ00lczpw5Y/zSp09vtE2kQ/OBGQizR7B7ryB5F198scnT/kC2IF3WfeLECXsraFAsEcST9FytHDlyxGhbjh49Kt75XnLJJUK4jRsNVzQbYO2934k2gyjTXgjkCJJ51VVXXVAtwiBOl19+uZDXggULBE0fOOIPUYb4emOJ20lQ0UjZzF0ul+TPn9+YGfGHrNmwQK8QZ8gVZk3SoDmjLkj79u095ka0dxUrViSKMSdDPozD8QMetLn1QrsJQfGXt40frmti7Qv+znLRd2ljp19avw83ftGOv+IX7S2o5Y8EBIIiXAcPHjTmCjQVTrHajMQqhFYBEyEaBlbkmP6YyBNL4wzbuXOn0ZbccMMNHm9OF2JW9HgkcpM1a9Z4oZA1JvHffvtNChYsGC8MN/7xPCPcgcYRkxhmVGddaTP20znbi/uJEydeUKPvvvtOihUrJpip1q9fL6dPn5YdO3YIJIYJnL1taJuc5IpMcKPd5N6XoJlEwwWpwczsK05CfpBFCBd7wiBGxIOsUwcraPHwZ/8YplDMzJB79uPh7xTwcOJDmdD2+cvbmUc47hNrX/B3lom+vW/fPqdXmr9X/ELrAopfaPhpakUABIIiXGgM2DtjB3g2sbOJnowSE06IsQ+FOEziTNxoo3iJ8QtE0GSx0Zm9Y8RnUsGc4os4EO4tl156qdF64H/nnXcaEycaEj4xAYlgHxJh7EO79tprzSZz3NEk4Dp37lxxarlmzZold999t7BXjbpcffXVMmjQIKPVQwuIZMqUiSBjKoY816lTRzAJ47lmzRqTH1jhxhSHRtAeSChUqJBAWDAjEu5LILYQMk5/YlbGLBg/3oUuNGvsGezTp48xbaKJwgz8448/yuOPP+5JQBw2jeOBSRHTKdpUzKOWRBFmhc+JlC9fXtDwZciQwRwC4VCFv7xt+nBefbUv9WFfGosEysaChncD8oxb5T8EFL//sEjKneKXFNQ0jSLwHwJBES6+ibV69Wpz4o97NsB7fzvrv6z/u0PDgpmDjcqYIXv06CFoY4jBpx6YzIcOHSqY8rhH2LNDuFM4icjE+OWXX8qbb75pBM2HM05C91u3bjWaG1TjTMikZ3M8pI9Jmz1K1IVN9Zip2POVUF6R7P/6668bU6kt49KlS81JvH79+pnTo5xMZODEtAfZwmz49ttvm4MQpIGoYD5EC4kb4oVWkf1buNEQcTKwZcuWYk8pQsQhroQnJhs3bhTa/6WXXopXRmcaCB3tT96Qe/YJcoLUxuFzIGi8OHjx7rvvmk9IsEfNhtMfOCgAEbF+zitmUfaAcaKROkJE6XvE8Zc3ccIt3u1Lfd544w2zFw1MevXqZQ4EeJsZw13uSHm+4hdaSyh+oeGnqaMAgRQsYlCEiwmaI/mcSMM0xKkoJm/Kx2Z6q33CzcZoTD58sgCyxUQP0WnRooWgkYDcEA9tC/G8BVMhgr+dPCBHTL6s4ps1aybs2SEPhImGZ3CPMPGiAeMecblcwkSP9qdevXrmswb4I5AITnZVqlTJaDy8N1oTJxIF0vvUU0/FKxqaQMxwtJENQMtF20Am0Tby+QAbhpkXjQ+fzcCPTymAOWQUN0QKN+2LG0HrRT6cUuQkIEQYf4S25XncIzyLuNwjfJaDE5X0CdxWIHo8xyloUG25bDy0ZRC+Bx54QCDKkH78bDjPIw8+/eH0c5YBrSjlJB6aTT5BQVzySSxv4qSmBNq+06dPNycuwQTtrbPuqVneSHuW4hdaiyh+oeGnqRUBbwSCIlzeidWtCCgCioAikKYQ0MoqAopAEhFQwpVE4DSZIqAIKAKKgCKgCCgCgSKQJggXZianSSlQcDSeIhA0AppAEVAEFAFFQBHwgUCaIFw+6q1eioAioAgoAoqAIqAIpBoCqU24Uq1i+iBFQBFQBBQBRUARUAQiBQElXJHSEloORUARUAQUgVREQB+lCKQuAnH58uUTFcVA+4D2Ae0D2ge0D2gf0D6Qcn0g7qeffhIVxUD7gPYB7z6gbu0T2ge0D2gfSL4+oCbF1NUo6tMUAUVAEVAEFAFFIA0ioIQryY2uCRUBRUARUAQUAUVAEQgMASVcgeGksRQBRUARUAQUgchEQEsVFQgo4YqKZtJCKgKKgCKgCCgCikA0I6CEK5pbT8uuCCgCgSCgcRQBRUARCDsCSrjC3gRaAEVAEVAEFAFFQBGIdQSUcMV6CwdSP42jCCgCioAioAgoAimKQEQQrquvvlomT54sy5Yti1fZuLg4GTt2rKxbt06aN29uwsqVKyeLFi2S9957z7iT66dly5ayatUqeeKJJ5IryxTPp0OHDrJp0yapWrWqOP/VqFFDZs+e7fRK8L5hw4Zy+eWXJxgeagDtNnXqVL/Z3H777aYu1AdZsmSJTJs2Te655x6/af1FCLQM/vJJ7fDkaN/ULnMkPU/xC601FL/Q8NPUSUMgllOFnXAVK1ZMRowYId98880FOEMEmIirVasmY8aMkQceeEA6duwo33333QVxQ/H43//+J0WKFJEtW7aEkk1Y0m7fvl2ee+45yZIlS5Ke36ZNG8mVK1eS0qZEottuu02KFy8uTZs2FYja008/La1btw7pUfSd+++/P6Q8wpU41PYNV7kj5bmKX2gtofiFhp+mVgScCARFuNKlSyc9evSQL7/8Uj7//HMZPny45M6d2+TXpEkTo40aOnSoTJgwQebMmSPt2rUzYZkyZZJevXrJ+PHjjZDulltuMWF//fWXmVzXrFlj3PYnR44c8tZbbxnnG2+8IbVr1zZfxH/kkUfk4MGDxt/5M2TIEOnfv78pA5qRGTNmSPXq1U0UtFcff/yxyY+yzZ07VyAaJtD9Q9izzz4rZ86ccbui6//mzZtl48aN0q1btwQLft9998msWbPkk08+kSlTpsiNN95o4r766quSP39+GTBggMGDNqCtCARvNE0QXtyQ3/Xr10v27NmlZMmSJp9PP/1UwLlr165EMTJ//nzp3LmzLF26VOgTxtPxQ9ujnaQvObzj3Z47d86Q348++khYZdNW/LkJItHf3n77bZk5c6bR4nXv3l3QhD7//PMycuRIoniE59DvvDVcjRs3lnnz5sny5cvlnXfekSuuuMKkSShvEximn0DaFzJJO0yfPt1oBevVqxem0kbeY2MPv9TFWPFLXbz1abGNQFCEC+3SXXfdJQ8//LDUqVPHTHRMaEB09uxZgURNnDhRGjVqJC+++KK0b9/eaF6qVKkipGOiQyA45EG67W4Nzb59+7iNJ4cOHTITN56tWrUyBA9CwHPw8xYm6bJly8ozzzwjDRo0MGZHJmriEVaiRAlhoqZsL730kkAS0K4Rvnr1ai5RKRAXsK5bt66HSDkrggm2T58+AqGsX7++fPXVVzJs2DDTdj179jRRe7hJNFpG2sESrDJlysjatWuFK5HKly9vTLs8D5ICuYHIPfnkk4JZEiEe7QPWtDOECT8raOIgbJAo4ln/xK4bNmwwBPvOO+800SD0J0+elHvvvVd4BtpPrhAO2t9q63K4CTt1x1RtEv77Qz70S8g5eeBNubgmlDdh4RLwTqx9ea/QAtIOEC1IJnUrXLhwuIocUc9V/EJrDsUvNPw0tSLgRCAowsV+GsgSZAgSA4GBTKFhIFMm7O+//55b+eGHH4SXFQ3K0aNHJU+ePNKyZUthImQPFgTAREzGH/Zg7d692+TIRM0zjcP9gz97wdy3smDBAtm1a5fcdNNNOKNaXC6XUDcIU79+/S6oC0Rs4cKFYknl+++/LxdddJFAoLwjs4fOEq5bb71VRo0aJaVKlTLRMPWxr4r2pj2/+OIL4w9hXrx4sUCyjIf7By3Xtm3b3Hci58+fNwIpgry1bdtWTp06ZcIC/fnzzz+FfX5ooiif1Xzu379fJk2aZPZ5UT/KQh8lXxYEW7duFVbouK1Asr799lv5+uuvZe/evQJmEJrE8rZpw3F1uRJvXzSR1Ie6Uz7ahXrRfrjTurhcil8ofcDlUvxCwU/TKgJOBIIiXDlz5jT7hTA1IWizMmbMKIUKFTJ5njhxwlz5sRqM9OnTm/1ZaDUwRWGOxOTHREG85JTjx497suP5lgjiCUnkauXIkSOG/Fl3tF8hUhkyZLhgvxNmspo1awrthUBEaccCBQpcUGUIFcQJ8kG7YXotWrSo0Ybhzz67Sy+9VA4fPhwvLdiSp/V0mnxdLpdcddVVRluJ/7Fjx2y0gK+YMyFXefPmNWkwJ1IXBE2lNTeivbMHCDAnQz5MAscPeEAYrRfEG4LiL28bP1zXxNoX/J3lom8728MZllbvFb/QWl7xCw0/TZ0mEbig0kERLiZMzMrvpjkAABAASURBVE/XX3+9OMVqMy7I3eHBRnfMHphAmLgx6zkJkSNqitxecskl8fLNli2boDmJ5xnFDjSOffv2NYTLWVeICvu2nO3F/bhx4y6oLZqwG264wZh/MSeePn1afvnlF3MKkgkcrSWYYRZ0Js7hNt/98ccfTq949xBxCHZeN2GCeMcL9OOo6SaLaCo5dblnzx4TG+0cdbBSqVIl4z99+nQpXbq0XHvttQJBZM+aCXD8gEfWrFk9PpA58vOXtydBmG4Sa98cbvydxaJvQyKdfmn9XvELrQcofqHhp6kVARAIinCx8RrTkB3g+fwABIyMEpNHH31UXn/9dROFSfzHH38UtFG8xMYzFX6YtNF68KiKFSuaTyGwsRt3rAiECA0ie95sndjbVLlyZbH71dBscQgB0sEhATSBdqM8mhLIM3vgaCPyWLVqlZCfxQpTHITOYnnNNdcY8yRmWuL7kgMHDgjmZrRR9AXMgr7iOf3QrLFnkBOk7BdDE4XpFLNnp06dPFGJQ3nx2LFjhzGdsieLeL5IB324QoUKAg5oBDFtY+r2lzf5h1t8tS+HV9iXZvdsYTKFEGO2D3d5I+35il9oLZLs+IVWHE2tCEQdAkERrjfffFNWrFgh7OPi+1iPP/64YHbyV2s0LJihSI8Z8oUXXhC0MaRj0z2mITZiszLnHmHPDuFO4TMBhDFpoynjPpDnkwd7eUjDnh82i3OqcefOnQLZIB8EkyeTOfeDBw8mWdTJoEGDzElCW3D2V73yyisycOBAc3qUekOmMO1BtiBQo0ePNqcUScOginYILSRu4t58882CuRE35Il2Zy+WPaWI6Y5Tf4QnJpxyZK8Z2NLWvuLyfPAnb045sk+Qzew2bpcuXcx+QIjThx9+KHfccYegmbPhmBXxI9z6Oa8QEfaAsT8NLR5EFHyI4y9v4oRbvNuXduK0KQchqDP70dg0D3kOd1kj8fmKX2itoviFhp+mTtsIBEW4mKD5hABmHr6TxKk3JkgghISxUZl7hI3RmHz4ZAF7SiBZmJM4zcY+Gz4bQTxMi8Tzll9//VUQ/O3kwfF33E4hL/Lhswg8g3sEExQrf+4RzJd8OuChhx4yn4tAa4K/LaczT+7Jj/BIFggsJ+6cZWR/EoQD7aP1h7zQNpzQBEMIlg3DzAvBggjhx4BK/SGjuCFSuGlf3MjKlSuFfDiliAmyd+/eeBuhPdCqGYf7Z8yYMSau+9b8h1ijYaJPGI9/f/hEA89xCqcObbn+jSaYBCF8tWrVkmbNmpmTivjZcOpGHpBz6+ddBsypbP4nHjjZ76+RT2J52/xS6SqBti8nMdFsgQmLBvZIplYZI/k5il9oraP4hYafplYEvBEIinB5J1a3IqAIKAKKgCKgCCgCioB/BKKXcPmvm8ZQBBQBRUARUAQUAUUgIhBIE4TL26QUEchrIRQBRUARUARiAgGthCIQCAJpgnAFAoTGUQQUAUVAEVAEFAFFIKUQUMKVUshqvoqAIvAvAnpRBBQBRUARUMKlfUARUAQUAUVAEVAEFIEURiCuSJEiohJeDBR/xV/7gPYB7QPaB7QPxHYfiOPv5qlcIYqBYqB9QPuA9gHtA2m8D+hceEXKvQNqUkxhFaJmrwgoAoqAIqAIKAKKgBIu7QOKgCKgCASKgMZTBBQBRSCJCCjhSiJwmkwRUAQUAUVAEVAEFIFAEVDCFShSGi8QBDSOIqAIKAKKgCKgCPhAQAmXD1DUSxFQBBQBRUARUASiGYHIK7sSrshrEy2RIqAIKAKKgCKgCMQYAkq4YqxBtTqKgCKgCASCgMZRBBSB1EUg1QlXlSpVpGfPngHXMlOmTNK+fXsZNmyYDBo0SOrVqxdw2liPWKdOHXn++ed9VjNYnH1mEoRn586d5d133zXyzjvvSL9+/aRVq1aSLVu2BHN55ZVX5NZbb/UZTr0qVarkMywteF500UUCPrSxd3179eolzZo18/ZWtwMBxc8BRhJv6Xu80yVKlIiXA+/siy++GM9PHamLQLFixcz48OSTT6bug/VpISGQ6oQr2NI2bNjQJOEFHzFihNx+++1GjKf+JIjA119/LS+//HKC4SkRsHDhQmnbtq106NBBxowZI1deeaXUrVs3wUd1795dVqxYkWB4eAPC+/S//vpLxo0bJzVq1JC8efN6ClOzZk255JJLZPLkyR4/vbkQAcXvQkyS4rN37155+OGHhYVvUtJrmuRHoHz58gIZ3rhxY/JnrjmmKAIBE67bbrtNhg4dKunTp/cUqFatWvLcc8/J5ZdfLm+++aagfWISHTJkiLlv0qSJ0Wb1799fKlSo4EnHTfPmzU1+rOLJBz++8Es+9913n8kvf/78UqpUKZkyZYrs27dPtmzZIosWLZKyZcsSXSURBJwartatWxuNSLdu3Ux7vfDCCx7NEhN4jx49PDllz57daKly5cplJns0VbbNyRMSh/bAk8DHzZkzZ2Tbtm2mvXLkyGFioK3q2rWrdOzYUegPeNL2rJa5L1eunCGIlOXxxx/HyyO33HKL9O7d28hTTz0l9CvqRIS4uDh58MEHjUatT58+0q5dO0NICIt2Wbt2rfz444/StGlTU5VLL71UateubYgYhKJgwYLm/ULjRTs53wvaCqx4H7t06ZIm3xnFz3SbJP+cP39edu7cKb/++qs0aNAgwXwgAIwpaKWZD5z9MMFEaSAgsbGpYsWKwnjcpk0beeaZZ8yYyLttYbnmmmvMuw2mYFu/fn0bZNpk8ODBcvToUY+f3kQHAnGBFnPVqlVy7tw5YfIjDVKyZEn54YcfjH+GDBnkyJEjRs05Y8YMqVatmukYTNALFiwwkzdpEDQff/zxh/RxT5CzZ88WCFaBAgWEiZrJnQmdiYIXHjerLNIhpMuXLx+3KgEiQLtdf/31MnLkSPNir1u3TiBa/pLTNseOHTOTPETs3nvvlbFjxwqTvb+0EK1rr71WmPSIe/bsWaGNeTbkCT8rl112mSFR5D1gwACTxrYx7Q/hQBPWt29foS/dcccdNqlZ6d1www2m3xHucrnMitwTIcpvJk2aJHny5BEIa+PGjWXlypUGH7RckNfvv/9eXnrpJdMu4ARumTNnNiT0gw8+MLjQ7kyKvKNRDkfQxVf8goYsXgJIw4QJEwxh5/2NF+h28O7df//9ZpGGFYIxg8W0UyvrjpYm/6OFAh8Wlt5jE2NyoUKFjAJh4MCBAsaMr2gSebfZRvPVV18JmGLWveuuu+Tuu+82OG7fvt3MucahP1GFQMCECzIE6bIaCbRREKclS5Z4KsxkgAPtBhPl8uXLcRpth3Mvz+HDh+WLL76QQ4cOybx58+TAgQNmMiayy+UynfD48ePCJP/333/j7REm+yxZsnjcehMYAj///LMcPHjQRGbFCrbG4ecHEoR2smXLlkbbkpgam3gMDgiDDO3r7B8nT56U+fPnG2LtfCz7EegDNu/vvvtOaH/ioNVhAEKziRuNj42HG80rYcSHoM+aNUtKlCghLpeL4KgX6sVgzKTG+zZx4kRTJxY+p0+fNnjigfYXXMq6tb9MkmB9zz33mL+LxkLo1VdfFeITNy2J4pf01na5XOY9Ytz48ssv5ZFHHrkgs9KlS8v69evFLopZgPPeo6G5IHLKeURkzv7GJubBTZs2mbLz/vLe5s6d2yg1mOfAkkCw5d0uXLgwTpUoRiBgwkUdly1bJjB2yFSZMmVkw4YNQqchDKGTcEWbwZXBjits3uVycWvkxIkT5mp/cF988cXW6ckT7QraLk+A+4Z49jlup/4PEAEmYBuV9uDltu7Errt375Y1a9YIGrKZM2cmFlUWLlxo9nBhEuSQA5Exa3FFaGeu3gKh8m5T2p54WbNm5eLpEzicfY5w9vlB8hBMkvTPvI59T6SJZkG7h0ndSURZwOTMmdNoFqg3ctNNNwkDNjhDeKkzWjA0yU6TBP5pSRS/0FsbwsV7xZ5CZ268u3act/70P95L606rVzBIbGw6deqUBxrGZBzp0qUzB428McXN3EcclehFICjCBRvnZUKDgFgNVrDVx+ThTIPGyjmJ2rA///xT0FoULFjQegkmE/w9HnoTEgK86C7Xf2TY+6W++uqrzYqLVWxi+zichaDNiD969GijufS3MmMw8e4TDFbkafcpOMvl1M5BzDCdsVnfKRBF0seK0E6IrQ9aK+rorDP37733nolC2EcffWT2gXz22WfCni5IswlMgz9gh9iqK34WicCuvNPjx48XtiI4F8G8n853k9wYz9FycZ+WJaljE33TG1PcvubItIxvNNY9KMJFBTHpYE9m0uMev2AlR44cnk3brMpxb968+YJs0HosXrzY7F8hkBeZU4qYIXGrhI4A6mo2yPNCkxumKq4Ie34wJWKmGzVqlBQtWlRQkxPmT1wulzlNyqCza9euRKNv3brVHLwoUqSIicd+Izuos2ePwRtzJYGYH2083KjdK1asKLb8mLzZRE9YLMvq1atNncu6TYjUE43lY489JixOMOf07dtXLInFHIHWmcUScVVEFL/gewFmL8Z8p5bLWj2sRhkTI+M0ezWDf0JspUjq2ETfZPxjLAMRtu8UL17c7N3ErRK9CARNuDBrsFJmP1dS9oS4XC5hPxFEi0kBlev06dNl//79PlHkaDx7CNhY2LlzZyEuJgKfkZPmGdWp2NeDOckpmHsDrRTmQvDlcAOn2SA3rGZdLpc5mQRhmjNnjtlTxX4uNm5jtvKVP6TIloNTNBC0119/XZzmTF/pfv/9d6Ff8Xz6BBox9pkRl7LwCQSIBRtIOcnjbP/P3Nob9gxyyIJTP5x4pW+SNpYFzcJbb70llStXNidPn376abORlg214MHeuUcffVTw5+TY3LlzzWmzWMYkmLopfsGg9V/cqVOnGqJvfdBkT5s2zWwl4N3l4Ab7DNFa2zhp9ZrUscn2TbSJ9pQi5A0iBpZ8x5Jxtnr16maLD/f25DfhKpGLQNCEa8+ePeblGjNmjKdW7C/BnGEnVjQauG0EJgFOXeBm4GcDL+k5rcaRdk62EGbzcb6sTLiQLI7OQgqcky1p0rLwQoOztyxdulSc3+HilBrE1WIFhkzEuMEXXDt16iSvvfaacOqNPVgQYAZO2op4CKtWPu1AGG6nDB061PQLWxb2bkEILHFiYzsDsjMNJImy4Pfhhx/KE088YT79ALFjDxIb7AljsOFoNIMPRA5zo+0jmIkgZIQTxok9VuKkiyWBbNLezjpt377dfEqDwRa8MK3acA6lDB8+3HwsGMLlndbGSytXxS/4lqbP0IecKbE68G7zLlr/b7/9Vni3Gc/ZM4hVwoal5WtiY5P3eIjygrHzt99+M5CxaGJcpt8yLrI9wAS4f/AnrlN4x91B+j/CEQiacEV4fbR4MYgAZB3NGmYzTNlowHyZoGOw6lolRUARUAQUgRhBQAlXjDRkLFeDE1J8q4vTdmjYFi5caD5REct11rr5RkA9lLc/AAABWElEQVR9FQFFQBGIVgSUcEVry6WhcqNeZy8YZkPMFnxYNw1VX6uqCCgCioAiEAMIKOGKgUb8rwp6pwgoAoqAIqAIKAKRiIASrkhsFS2TIqAIKAKKgCIQzQho2S9AQAnXBZCohyKgCCgCioAioAgoAsmLQBxfpFbZLYqBYqB9QPtAKvYBHXN2a3/T/pa2+kAc38xS2SWKgWKgfUD7gPYB7QPaB7QPpFQfUJNi8moMNbfkQkDzUQQUAUVAEVAEYggBJVwx1JhaFUVAEVAEFAFFQBFIXgSSKzclXMmFpOajCCgCioAioAgoAopAAggo4UoAGPVWBBQBRUARCAQBjaMIKAKBIKCEKxCUNI4ioAgoAoqAIqAIKAIhIKCEKwTwNKkiEAgCGkcRUAQUAUVAEVDCpX1AEVAEFAFFQBFQBBSBFEYgAghXCtdQs1cEFAFFQBFQBBQBRSDMCPwfAAD//54BTc8AAAAGSURBVAMAw89Q9qYM8YYAAAAASUVORK5CYII=)


---

## Jour 3 : Réseau Hyper-V et Échec PBS

### 1. Résolution des problèmes réseaux Hyper-V (Romain)

J'ai passé une heure à debuguer une erreur **"Destination Host Unreachable"** sur ses VMs.

- **Cause :** Conflit d'IP entre le vSwitch et la carte physique, et confusion dans le nommage des interfaces.
- **Solution :** Réinitialisation de la couche réseau et renommage propre des adaptateurs. Tout est rentré dans l'ordre.

Par la suite, j'ai mis en place un **Storage Space (Espace de stockage)** en mode miroir sur Windows avec 2 SSD du serveur pour créer un disque **(J:)** sécurisé.

### 2. Commencement des livrables (Romain)

Après avoir réglé mes problèmes, j'ai décidé de prendre une pause sur les serveurs et consacrer du temps sur la réalisation des livrables (Compte rendu journalier et Bilan financier).

### 3. Tentative d'intégration Proxmox Backup Server (Alexandre)

J'ai tenté d'intégrer **Proxmox Backup Server (PBS)** pour gérer les sauvegardes.

- **Problème critique :** L'importation de l'ISO dans le stockage local des nœuds virtuels (Nested) provoquait un redémarrage brutal (Kernel Panic) de l'hyperviseur parent.
- **Action :** Malgré une après-midi de tests avec M. Toulliou, le bug a persisté. J'ai pris la décision de préparer une réinstallation complète pour le lendemain afin de repartir sur des bases saines.
  Un nouveau cahier des charges a été commencé ce jour là mais toujours dans l'espérance que l'importation de l'ISO du PBS allait fonctionner

---

## Jour 4 : Diagnostic et Changement d'Architecture

### 1. Refonte de l'Architecture Proxmox (Alexandre)

Après de nouveaux tests infructueux avec M. Pouchoulon dans la matinée, j'ai validé la stratégie de refonte totale de la partie Proxmox pour adopter une approche **Hybride** :

- **Ceph (RBD) :** Maintenu pour les VMs critiques (Web, DNS) afin de garder la HA.
- **ZFS Local :** Introduit pour la VM "Client" pour gagner en performance disque.
- **ZFS Réplication :** Prévu pour le serveur de sauvegarde (PBS) afin d'assurer un Plan de Reprise d'Activité (PRA) sans dépendre du cluster Ceph.
- Création d'un raid level 1 pour réplication sur les deux disques 1.7TO, comme cela, il n'y aura pas de disque inutilisés
Voici le schéma de ma nouvelle construction;


schéma physique
![[Pasted image 20251219204922.png]]
schéma réseau;
![[Pasted image 20251219205033.png]]
### 2. Préparation du Cluster S2D Hyper-V (Romain)

J'ai profité de cette demi-journée pour préparer le terrain pour le **Cluster S2D (Storage Spaces Direct)** : validation des prérequis réseaux, documentation des IPs et croquis du schéma réseau.

---

## Jour 4.5 (Romain/Week-end) : Cluster S2D à distance

### Travail à distance (VPN Tailscale)

Pour travailler à distance pendant mon week-end, j'ai installé le logiciel **Tailscale** sur mon serveur Windows. Une fois authentifié, j'ai pu prendre le contrôle total du bureau de mon serveur à distance via RDP.

### Implémentation du Cluster S2D

- **Mise à niveau :** Passage de Windows Server Standard à **Datacenter** (via clé KMS) pour débloquer la fonction S2D.
- **Stockage :** Création d'un pool de stockage partagé utilisant les disques locaux des 3 nœuds (Miroir à 3 voies, ReFS).
- Test de résilience :
    
    Pour faire ce test, nous regardons d'abord sur quel nœud se trouve notre VM. 
    ![[Pasted image 20251219205153.png]]
	Étant sur le nœud 2, nous l'éteignons pour voir comment réagit l'installation.
	![[Pasted image 20251219205212.png]]
	Résultat : Le cluster détecte automatiquement que le nœud est tombé. Après quelques secondes, la VM a redémarré sur le nœud 1 (SRV1).
    ![[Pasted image 20251219205231.png]]
- **Migration à chaud :** On active le "MAC Address Spoofing" pour permettre le déplacement des VMs sans coupure réseau, puis, pour réaliser ce test, rienn de plus simple. On prend notre VM, et on appuie sur move -> live migration -> best possible node. Avant cela, on lance notre VM avec un ping infini vers google. Puis, on lance la migration dynamique. On voit qu'un ping est à 12ms (au lieu de 6 ou 5 pour tout les autres) quand on clique sur la migration, mais ils reprennent de manière normale juste après, sans coupure :
    ![[Pasted image 20251219205246.png]]
    ![[Pasted image 20251219205312.png]]
    
    Résultat : Un ping légèrement plus haut (12ms) pendant la bascule, mais aucune coupure réseau.
---

## Jour 5 : La recofiguration entière de Proxmox et dernière retouche Hyper-V

### 1. Refonte totale de la partie Proxmox (Alexandre)

Ayant validé la nouvelle architecture théorique, j'ai procédé à la réinstallation complète de l'environnement Proxmox avec un temps assez limité :

1. **Configuration iDRAC :** Création du Virtual Disk en **RAID 1 (Miroir)** sur les SSD de 1.7 To pour la redondance physique.
2. **![[Pasted image 20251219205447.png]]

3. **Déploiement :** Réinstallation de l'hôte et création des 3 nœuds virtuels, toujours les memes.
4. **Stockage :** Configuration immédiate des pools Ceph et ZFS, et ZFS raid.
J'ai donc créé chaque 3 neouds virtuels avec 4 disques, un pour l'OS, 32 Go, et 100Go chacun pour un ZFS, ZFS raid et pour le CEPH, j'ai donc une architecture de ce type là

### 2. Peaufinage de Hyper-V (Romain)

J'ai pris un peu de temps pour m'assurer du fait que la migration dynamique se faisait bien de tous les serveurs vers les autres pour valider le cluster à basculement.

### 3. Continuité des livrables (Commun)

Pendant qu'Alexandre faisait sa partie Proxmox plus longue que prévue, à cause du crash, Romain a avancé le comparatif financier. Alexandre a ensuite documenté sa refonte.

---

## Jour 6 : Déploiement des Services et Dimensionnement

### Alexandre :

J'ai finalisé l'infrastructure Proxmox en déployant les services finaux. J'ai choisi **Alpine Linux** pour sa légèreté.

### 1. Déploiement et Configuration Détaillée des Services (VMs)

Pour cette étape, j'ai fait le choix technique d'utiliser Alpine Linux.

Pourquoi ? C'est une distribution orientée sécurité et ultra-légère. Une VM Alpine consomme environ 50 Mo de RAM au repos, ce qui est crucial étant donné que nos ressources RAM sont limitées sur le cluster virtuel.

Configuration standard des VMs : **2 vCores, 512 Mo RAM, 8 Go Disque**.

#### A. VM 105 : Le Serveur DNS (Bind9)

**IP :** `10.202.6.53` | **Stockage :** Ceph RBD (Haute Disponibilité)

Le rôle de cette VM est de traduire les noms de domaine (`www.sae.lan`) en adresses IP. J'ai choisi **ISC Bind9**, le standard de l'industrie.

1. Installation et Configuration Réseau :

Après avoir installé l'OS, j'ai fixé l'IP statique dans /etc/network/interfaces puis installé le paquet :

```
apk add bind
```

2. Déclaration de la Zone (named.conf.local) :

J'ai configuré Bind pour qu'il soit "Maître" de la zone sae.lan. J'ai édité le fichier de configuration pour déclarer cette nouvelle zone :

Bash

```
zone "sae.lan" {
    type master;
    file "/etc/bind/db.sae.lan";
};
```

3. Création du Fichier de Zone (db.sae.lan) :

C'est l'étape la plus technique. J'ai créé le fichier de base de données DNS contenant les enregistrements (Records).

- **SOA (Start of Authority) :** Définit les paramètres globaux (TTL, Email admin).
- **NS (Name Server) :** Indique qui est le serveur DNS (lui-même).
- **A (Address) :** Fait le lien IP ↔ Nom.

Voici la configuration exacte que j'ai injectée :

Bash

```
$TTL 604800
@       IN      SOA     ns1.sae.lan. admin.sae.lan. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; Serveur de Noms
@       IN      NS      ns1.sae.lan.

; Enregistrements A (Nos services)
@       IN      A       10.202.6.53
ns1     IN      A       10.202.6.53
www     IN      A       10.202.6.54     ; Pointeur vers le Web
pbs     IN      A       10.202.6.255    ; Pointeur vers le Backup
```

Une fois configuré, j'ai activé le service au démarrage : `rc-update add named default && rc-service named start`.

#### B. VM 106 : Le Serveur Web (Nginx)

**IP :** `10.202.6.54` | **Stockage :** Ceph RBD (Haute Disponibilité)

Pour le serveur Web, j'ai opté pour **Nginx** plutôt qu'Apache. Nginx est réputé pour sa capacité à gérer beaucoup de connexions simultanées avec très peu de mémoire, ce qui s'aligne avec notre philosophie "Alpine".

**1. Installation :**

Bash

```
apk add nginx
```

2. Configuration du Site :

J'ai modifié le fichier /var/lib/nginx/html/index.html pour créer une page d'accueil personnalisée prouvant que le serveur répond bien.

J'ai ensuite vérifié la configuration du vHost dans /etc/nginx/http.d/default.conf pour m'assurer qu'il écoutait bien sur le port 80.

3. Test local :

Un curl localhost sur la VM m'a confirmé que le serveur web tournait correctement avant même de tester depuis le client.

#### C. VM 107 : Le Client de Test

**IP :** DHCP | **Stockage :** ZFS Local

Cette machine simule un utilisateur du réseau.

- **Particularité Stockage :** J'ai placé son disque sur le stockage **ZFS Local** (et non Ceph). Comme c'est une machine de test jetable, elle n'a pas besoin de Haute Disponibilité, mais elle bénéficie ainsi de meilleures performances disques (I/O) pour lancer des benchmarks si besoin.
- **Outils installés :** `bind-tools` (pour avoir `nslookup` et `dig`) et `curl` (pour tester le web).

![[Pasted image 20251219205818.png]]
### 2. Le Serveur de Sauvegarde (PBS)

Installation de **Proxmox Backup Server** (VM 110) sur un stockage **ZFS Répliqué** (pour la sécurité).
- **Ressources :** 4 Go RAM (pour la déduplication), 20 Go Disque.
- **IP :** `10.202.6.255`
- **Liaison :** Le Datacenter PVE a été connecté au PBS pour permettre les backups.
![[Pasted image 20251219212942.png]]

### Romain :

Ayant fini ma partie technique, j'ai pu me consacrer à 100% aux livrables, j'ai mis au propre et en markdown ce compte rendu. J'ai ensuite mis au propre mes idées par rapport aux bilans financiers sur Excel.

---

## Jour 7 : Supervision et Tests Finaux

### 1. Supervision Centralisée "Datacenter Manager" (Alexandre)

Pour assurer la veille technologique et la gestion globale, j'ai installé une solution de gestion centralisée (Datacenter Manager) sur une VM dédiée. Cela nous permet de visualiser la charge du cluster en temps réel via une interface unifiée.

![[Pasted image 20251219213027.png]]

### 2. Validation Fonctionnelle (Alexandre)

Nous avons clôturé la semaine par les tests de validation :

- Test Service : Le client accède bien au site web www.sae.lan via le DNS interne.
    
![[Pasted image 20251219205934.png]]

La page de base du localhost d'un service Nginx correspond exactement a ce fichier HTML, c'est une réussite, cela veut dire que mon serveur nginx et mon service DNS sont tous les deux opérationnels


### 3. Bilan Financier et Documentation (Commun)

En cette fin de projet, nous avons mis en commun nos rapports et nos expériences. Bien que nous faisions des choses assez proches, le fait que l'un soit sur Proxmox et l'autre sur Hyper-V a enrichi notre vision globale.

---

# PARTIE 4 : BILAN FINANCIER & ANALYSE DE RENTABILITÉ 

## 1. Introduction et Paramètres de l'Étude

Afin de départager les solutions **Proxmox VE** et **Microsoft Hyper-V**, nous ne nous sommes pas limités à la technique. Nous avons réalisé une projection financière sur **6 ans (72 mois)** pour une infrastructure type composée de **3 nœuds physiques**.

L'indicateur clé utilisé est le **TCO (Total Cost of Ownership)**, qui englobe le coût d'achat (CAPEX) et le coût de fonctionnement (OPEX).

### Hypothèses de calcul :

- **Infrastructure :** 3 Serveurs Physiques (Nécessaire pour le quorum Ceph/S2D).
- **Charge de travail :** Scénario de base avec **15 Machines Virtuelles** (Windows Server).
- **Coûts Humains :** Salaire administrateur chargé à 45€/h.
- **Énergie :** 0,25€/kWh.

---

## 2. Investissement Initial (CAPEX)

Ce tableau représente la trésorerie à sortir "Jour 1" pour monter l'infrastructure.

|**Postes de Coûts (Achat)**|**PROXMOX (Cluster Ceph)**|**HYPER-V (S2D)**|**Analyse & Explication**|
|---|---|---|---|
|**Matériel (3 Serveurs)**|30 000 €|34 000 €|Hyper-V nécessite souvent du matériel certifié/validé (HCL) plus strict et cher.|
|**Réseau (Switchs 10/25GbE)**|4 000 €|4 000 €|Idem pour les deux : le stockage distribué (Ceph ou S2D) exige du réseau rapide.|
|**Licences OS (Hôtes)**|**0 €**|**18 900 €**|**L'écart majeur.** Proxmox est gratuit. Hyper-V nécessite 3 licences "Datacenter" (6300€/CPU).|
|**Logiciel de Sauvegarde**|0 €|5 000 €|Proxmox Backup Server est gratuit. Hyper-V requiert souvent Veeam (payant).|
|**Prestation d'Installation**|3 000 €|2 000 €|Ceph demande une configuration plus fine au démarrage que l'assistant S2D Microsoft.|
|**Formation Équipe**|1 500 €|0 €|Budget provisionné pour la montée en compétence Linux de l'équipe (si nécessaire).|
|**TOTAL CAPEX**|**38 500 €**|**63 900 €**|🔴 **Hyper-V est 65% plus cher au démarrage.**|

---

## 3. Coûts de Fonctionnement (OPEX)

Ce tableau représente ce que l'infrastructure coûte chaque jour pendant 6 ans (Maintenance, Électricité, Support).

|**Postes de Coûts (sur 6 ans)**|**PROXMOX**|**HYPER-V**|**Analyse & Explication**|
|---|---|---|---|
|**Support / Maintenance**|1 350 €|3 000 €|Comparatif entre l'abonnement "Community" Proxmox et la Software Assurance Microsoft.|
|**Électricité**|2 700 €|3 000 €|Windows Server (avec interface graphique) consomme légèrement plus de ressources "à vide".|
|**Main d'œuvre (Maintenance)**|1 800 €|2 700 €|**Le temps homme.** Les mises à jour de clusters Windows (S2D) sont lourdes, lentes et demandent plus de reboots que Linux.|
|**TOTAL OPEX (6 ANS)**|**35 100 €**|**52 200 €**|🟢 **Proxmox coûte moins cher à maintenir dans le temps.**|

---

## 4. Verdict Final : Le Coût Total (TCO)

C'est ici que se joue la décision finale. Nous comparons le coût global pour héberger nos **15 VMs**.

- **Stratégie Hyper-V :** On paie très cher les hôtes physiques (Licence Datacenter), mais cela donne le droit de créer des VMs Windows **illimitées** gratuitement.
- **Stratégie Proxmox :** L'hôte est gratuit, mais on doit acheter une licence Windows Standard (750€) pour **chaque VM** créée.

### Bilan pour 15 VMs :

|**Type de Coût**|**PROXMOX**|**HYPER-V**|**Notes**|
|---|---|---|---|
|**Investissement (Capex)**|38 500 €|63 900 €||
|**Fonctionnement (Opex)**|35 100 €|52 200 €||
|**Coût Licences VMs (15 VMs)**|**11 250 €**|**Inclus (0€)**|15 x 750€ pour Proxmox. Gratuit pour Hyper-V (car Datacenter).|
|**COÛT TOTAL (6 ANS)**|**84 850 €**|**116 100 €**||
|**ÉCONOMIE RÉALISÉE**|🟢 **31 250 €**||**Proxmox est nettement plus rentable.**|

---

## 5. Analyse de Sensibilité : Quand Hyper-V devient-il rentable ?

Une question subsiste : _À partir de combien de VMs la licence forfaitaire "Datacenter" de Microsoft devient-elle plus intéressante que l'achat unitaire de licences sur Proxmox ?_

Nous avons modélisé l'évolution des coûts en fonction de la densité de VMs :

|**SCÉNARIO (Nombre de VMs)**|**COÛT TOTAL PROXMOX**|**COÛT TOTAL HYPER-V**|**RÉSULTAT**|
|---|---|---|---|
|**3 VMs** (Infrastructure minime)|75 850 €|116 100 €|🟢 Proxmox beaucoup moins cher.|
|**15 VMs** (Notre cas)|84 850 €|116 100 €|🟢 Toujours gagnant (Gain : ~31k€).|
|**40 VMs** (Forte densité)|103 600 €|116 100 €|🟢 Toujours gagnant (Gain : ~12k€).|
|**58 VMs** (**Point de bascule**)|**117 100 €**|**116 100 €**|🔴 **Hyper-V devient plus rentable ici.**|

### Conclusion Financière

Tant que l'infrastructure héberge **moins de 58 VMs Windows**, la solution **Proxmox VE** est économiquement supérieure. Dans notre cas d'école (15 à 30 VMs), l'économie se chiffre en dizaines de milliers d'euros, ce qui valide définitivement le choix de l'Open Source.
