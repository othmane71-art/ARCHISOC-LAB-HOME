<div align="center">

# 🛡️ ARCHISOC
### Architecture de Cybersécurité Virtualisée — Mini-SOC d'Entreprise

*Laboratoire complet de défense en profondeur : segmentation réseau, double pare-feu FortiGate + pfSense/Suricata, honeypots DMZ, Active Directory durci, SIEM Wazuh.*

![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Platform](https://img.shields.io/badge/platform-VMware%20Workstation%20Pro-informational)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.14-orange)
![Firewall](https://img.shields.io/badge/Firewall-FortiGate%20%7C%20pfSense-blue)
![IDS](https://img.shields.io/badge/IDS-Suricata-red)
![Standard](https://img.shields.io/badge/aligned%20with-ISO%2FIEC%2027001-2d2d2d)
![Framework](https://img.shields.io/badge/mapped%20to-MITRE%20ATT%26CK-9c27b0)

</div>

---

## 🎯 Objectifs et cadre de référence

Ce projet reproduit une architecture de sécurité d'entreprise dans un environnement de laboratoire entièrement virtualisé, avec une approche de défense en profondeur alignée sur des référentiels reconnus du secteur.

- **ISO/IEC 27001** — les principes de segmentation réseau (zones DMZ, LAN SOC, LAN AD), de gestion des accès, de journalisation et de supervision continue mis en œuvre dans ce laboratoire s'inspirent des exigences du référentiel de management de la sécurité de l'information ISO/IEC 27001, notamment sur le contrôle des accès (A.9), la sécurité des communications (A.13) et la gestion des incidents (A.16).
- **MITRE ATT&CK** — les scénarios de démonstration (injection SQL, XSS, brute force SSH, reconnaissance réseau) sont cartographiés sur la matrice MITRE ATT&CK afin de valider la couverture de détection de Wazuh face à des techniques d'attaque documentées (Initial Access, Execution, Discovery, Credential Access).
- **Défense en profondeur** — chaque couche du laboratoire (périmétrique, réseau, système, applicatif) dispose de son propre mécanisme de détection, sans dépendance unique sur un point de contrôle.

---

## 🏗️ Vue d'ensemble de l'architecture

| Composant | Rôle | Technologie |
|---|---|---|
| Pare-feu périmétrique | Filtrage entrant/sortant WAN | FortiGate |
| Pare-feu interne | Segmentation LAN, IDS | pfSense + Suricata |
| Zone exposée | Services leurres et vulnérables | DVWA, Cowrie (honeypot SSH) |
| Annuaire d'entreprise | Authentification, GPO, audit | Active Directory |
| Supervision centralisée | Collecte et corrélation des logs | Wazuh SIEM |

---

## 📖 Table des matières

- [🎯 Objectifs et cadre de référence](#-objectifs-et-cadre-de-référence)
- [🏗️ Vue d'ensemble de l'architecture](#️-vue-densemble-de-larchitecture)
1. [Plan d'adressage](#1-plan-dadressage)
2. [Configuration VMware](#2-configuration-vmware)
3. [Installation et configuration de FortiGate](#3-installation-et-configuration-de-fortigate)
4. [Installation de pfSense](#4-installation-de-pfsense)
5. [Routes statiques FortiGate](#5-routes-statiques-fortigate)
6. [Configuration GUI pfSense](#6-configuration-gui-pfsense)
7. [VM Ubuntu DMZ (DVWA + Cowrie)](#7-vm-ubuntu-dmz-dvwa--cowrie)
8. [VM Wazuh Server sur LAN SOC](#8-vm-wazuh-server-sur-lan-soc)
9. [Connectivité DMZ to SOC](#9-connectivité-dmz-to-soc)
10. [Installation de l'agent Wazuh sur VM DMZ](#10-installation-de-lagent-wazuh-sur-vm-dmz)
11. [Intégration des logs FortiGate dans Wazuh](#11-intégration-des-logs-fortigate-dans-wazuh)
12. [Installation de Suricata et envoi des logs pfSense/Suricata vers Wazuh](#12-installation-de-suricata-et-envoi-des-logs-pfsensesuricata-vers-wazuh)
13. [Activation des catégories de règles Suricata (ET Open Rules)](#13-activation-des-catégories-de-règles-suricata-et-open-rules)
14. [LAN AD & Client](#14-lan-ad--client)
15. [Installer l'agent Wazuh sur le DC et le client Windows](#15-installer-lagent-wazuh-sur-le-dc-et-le-client-windows)
16. [Collecte des logs AD et Windows Client](#16-collecte-des-logs-ad-et-windows-client)
17. [Démonstration](#17-démonstration)
18. [Conclusion générale](#conclusion-générale)

## 🏗️ Architecture globale

<img width="1162" height="900" alt="image" src="https://github.com/user-attachments/assets/5e63292d-51c3-4bc9-bc86-a4a4bd40bc55" />

*Schéma Draw.io complet montrant FortiGate, pfSense/Suricata, la DMZ (honeypot SSH + DVWA), le LAN AD+Client, et le LAN SOC (Wazuh) avec toutes les IP de chaque interface.*

## 1. Plan d'adressage

| Zone | Réseau | Rôle |
|---|---|---|
| WAN (NAT VMware) | 10.0.0.0/24 | Accès Internet simulé |
| DMZ | 10.10.14.0/24 | DVWA + Cowrie (serveur Ubuntu) |
| LAN SOC | 10.10.20.0/24 | Wazuh |
| LAN AD | 10.10.30.0/24 | Active Directory + client Windows |
| Transit FortiGate↔pfSense | 10.10.50.0/24 | Lien inter-pare-feux |

| VM | OS | RAM | vCPU | Rôle |
|---|---|---|---|---|
| FortiGate | FortiOS VM | 2 Go | 2 | Pare-feu externe (périmètre) |
| pfSense | FreeBSD | 1 Go | 1 | Pare-feu interne + Suricata |
| Ubuntu-DMZ | Ubuntu 22.04/24.04 | 2 Go | 2 | DVWA + Cowrie (Docker) |
| Wazuh Server | Ubuntu 22.04 | 4 Go | 2 | SIEM/XDR — collecte logs |
| Windows AD | Windows Server 2019 | 3 Go | 2 | DC + client Windows |

## 2. Configuration VMware

La plateforme VMware Workstation Pro a été utilisée pour segmenter le laboratoire en plusieurs réseaux virtuels indépendants. Chaque VMnet représente une zone de sécurité précise, ce qui permet de reproduire une architecture réaliste tout en gardant l’environnement totalement isolé.

### Réseaux virtuels

| VMnet | Type | Subnet | Usage |
|---|---|---|---|
| VMnet17 | NAT | 192.168.100.0/24 | WAN (accès Internet via l’hôte) |
| VMnet10 | Host-only | 10.10.14.0/24 | DMZ (DVWA + Cowrie) |
| VMnet2 | Host-only | 10.10.20.0/24 | LAN SOC (Wazuh ) |
| VMnet3 | Host-only | 10.10.30.0/24 | LAN AD |
| VMnet4 | Host-only | 10.10.50.0/24 | Transit FortiGate ↔ pfSense |

Cette configuration permet d’isoler les flux entre la zone exposée, la zone de supervision et le réseau interne. Le réseau de transit dédié entre FortiGate et pfSense facilite aussi le routage et l’application de politiques de sécurité distinctes entre les deux pare-feu.

### Attribution des interfaces réseau

Les machines principales disposent de plusieurs interfaces afin de relier chaque zone au bon segment virtuel. FortiGate est connecté au WAN, au réseau de transit et à la DMZ, tandis que pfSense est connecté au transit, au LAN SOC et au LAN AD.

- **FortiGate VM**
  - NIC 1 → VMnet17 → WAN
  - NIC 2 → VMnet4 → Transit vers pfSense
  - NIC 3 → VMnet10 → DMZ
 <img width="377" height="308" alt="image" src="https://github.com/user-attachments/assets/65615126-ea16-4ab5-ac6e-dac7eb4fbca8" />
 
 *Attribution des cartes réseau vm FortiGate.*

- **pfSense VM**
  - NIC 1 → VMnet4 → WAN pfSense / Transit
  - NIC 2 → VMnet2 → LAN SOC
  - NIC 3 → VMnet3 → LAN AD
<img width="366" height="300" alt="image" src="https://github.com/user-attachments/assets/49964879-3dd7-4202-b5a6-0b0ce3ec89eb" />

*Attribution des cartes réseau vm PfSense.*

- **Ubuntu-DMZ**
  - NIC 1 → VMnet10

- **Wazuh Server**
  - NIC 1 → VMnet2
<img width="357" height="251" alt="image" src="https://github.com/user-attachments/assets/9d3f3b6b-22f8-422a-a0bd-360b13e728b6" />

*Attribution des cartes réseau vm wazuh.*

- **Windows AD / Client**
  - NIC 1 → VMnet3

### Objectif de cette étape

Cette étape pose toute la base du projet. Si les VMnets ou les cartes réseau sont mal attachés, les routes, les règles firewall, les agents Wazuh et les tests d’attaque ne fonctionneront pas correctement.

<img width="664" height="211" alt="image" src="https://github.com/user-attachments/assets/24841879-3433-4c07-ad0b-92d494d3a45f" />

*Configuration des réseaux virtuels VMware utilisée pour segmenter le laboratoire en zones WAN, DMZ, LAN SOC, LAN AD et Transit.*

## 3. Installation et configuration de FortiGate

FortiGate constitue le pare-feu périmétrique principal de l’architecture. Il joue le rôle de point d’entrée entre le réseau externe simulé, la DMZ et le réseau de transit vers pfSense.[file:1340]

Dans cette maquette, la VM FortiGate a été déployée avec **3 interfaces réseau** afin de séparer clairement les différentes zones du laboratoire.[file:1340]

### Interfaces FortiGate

| Interface | Rôle | Connexion VMware |
|---|---|---|
| port1 | WAN | VMnet17 (NAT) |
| port2 | Transit vers pfSense | VMnet4 |
| port3 | DMZ | VMnet10 |

<img width="478" height="137" alt="image" src="https://github.com/user-attachments/assets/5b796213-c839-418a-82a8-c0a15be812d6" />

*Configuration en ligne de commande de l'interface FortiGate : WAN.*

<img width="810" height="495" alt="image" src="https://github.com/user-attachments/assets/06d922ee-17d8-4727-881b-05f06f6d1524" />

*Vue de l'interface FortiGate utilisée pour segmenter le réseau de transit.*

<img width="720" height="493" alt="image" src="https://github.com/user-attachments/assets/2dc70695-9df6-4f92-966b-dc6dd5d3dd29" />

*Vue de l'interface FortiGate utilisée pour segmenter le réseau DMZ.*

La configuration des interfaces a été réalisée en **CLI** afin de définir précisément les rôles de chaque port et de préparer les étapes suivantes, notamment le routage, les politiques de filtrage et l’accès aux services hébergés dans la DMZ.

### Rôle de chaque interface

- **port1 (WAN)** : connectée au réseau NAT VMware, elle permet à FortiGate d’accéder à Internet et de jouer le rôle de pare-feu externe.
- **port2 (Transit)** : utilisée pour la communication entre FortiGate et pfSense via le réseau 10.10.50.0/24.
- **port3 (DMZ)** : reliée à la zone DMZ contenant la VM Ubuntu qui héberge DVWA et Cowrie.

### Objectif de cette étape

Cette étape permet de mettre en place la première couche de sécurité du projet. FortiGate servira ensuite à :
- relier la DMZ au reste de l’infrastructure ;
- faire transiter le trafic vers pfSense ;
- appliquer les futures routes statiques vers le LAN SOC et le LAN AD ;
- contrôler les flux entre Internet simulé, la DMZ et les réseaux internes.

Cette configuration prépare FortiGate à jouer un double rôle : pare-feu d’exposition pour les services en DMZ et routeur de transit vers les réseaux supervisés par pfSense.


## 4. Installation de pfSense

pfSense a été déployé comme **pare-feu interne** afin d’ajouter une seconde couche de contrôle entre les différentes zones du laboratoire. Il est placé entre FortiGate et les réseaux internes pour assurer le routage, le filtrage et, plus tard, l’intégration de Suricata et l’envoi des logs vers Wazuh.

La VM pfSense a été configurée avec **3 interfaces réseau**, chacune reliée à une zone précise de l’architecture.

### Interfaces pfSense

| Interface | Rôle | Réseau | Connexion VMware |
|---|---|---|---|
| WAN | Transit depuis FortiGate | 10.10.50.0/24 | VMnet4 |
| LAN | LAN SOC | 10.10.20.0/24 | VMnet2 |
| OPT1 | LAN AD | 10.10.30.0/24 | VMnet3 |

Cette répartition permet à pfSense de recevoir le trafic depuis FortiGate sur son interface WAN, puis de le distribuer soit vers la zone SOC, soit vers le réseau Active Directory selon les routes et règles de filtrage appliquées ensuite.

### Adresses configurées

- **WAN** : `10.10.50.2`
- **LAN** : `10.10.20.1`
- **OPT1** : `10.10.30.1`

L’interface web de pfSense est accessible depuis le réseau SOC via l’adresse **https://10.10.20.1**, ce qui permet d’effectuer l’administration graphique depuis les machines de supervision.

<img width="1013" height="703" alt="image" src="https://github.com/user-attachments/assets/ba8843c0-b9fc-4595-b23f-380283a4ac21" />

*Dashboard pfSense affichant les interfaces réseau actives et leurs adresses IP dans l’architecture interne.*

### Objectif de cette étape

Cette étape met en place le point de passage entre les réseaux internes et la couche de sécurité externe. Elle prépare aussi les prochaines phases du projet :
- configuration GUI avancée ;
- ajout des règles firewall ;
- configuration des routes vers la DMZ ;
- déploiement de Suricata et remontée des logs vers Wazuh.

Avec cette configuration, pfSense devient le point central de distribution et de contrôle des flux entre la supervision SOC et le réseau Active Directory.

## 5. Routes statiques FortiGate

Après la configuration initiale des interfaces, FortiGate a été complété par des **routes statiques** pour permettre la communication avec les réseaux situés derrière pfSense. Cette étape est indispensable pour que le pare-feu périmétrique sache comment joindre le LAN SOC et le LAN AD via le réseau de transit.

### Routes configurées

<img width="1597" height="302" alt="image" src="https://github.com/user-attachments/assets/e482f2f7-4d5d-457d-9fb0-393d4983e35c" />

*Table de routage statique de FortiGate permettant de joindre les réseaux LAN SOC et LAN AD via pfSense.*

| Destination | Rôle | Passage |
|---|---|---|
| LAN SOC | Accéder à Wazuh | via pfSense |
| LAN AD | Accéder au réseau Active Directory | via pfSense |
| Internet (défaut) | Sortie Internet | passerelle par défaut |

Ces routes permettent à FortiGate de transmettre correctement les flux destinés aux réseaux internes plutôt que de les considérer comme inconnus. Elles assurent ainsi la continuité entre la couche périmétrique et la couche interne de sécurité.

### Politiques associées

En complément des routes, des politiques firewall ont été ajoutées pour autoriser certains flux nécessaires au fonctionnement du laboratoire.

- **Transit → Internet** : permet aux machines internes, notamment dans le LAN SOC et le LAN AD, d’accéder à Internet à travers FortiGate.
- **DMZ → Internet** : permet à la machine Ubuntu de la DMZ d’accéder aux ressources nécessaires, par exemple pour l’installation de paquets ou la mise à jour des composants.

<img width="620" height="666" alt="image" src="https://github.com/user-attachments/assets/af759852-650c-48f5-86f4-8055047c4e30" />
<img width="600" height="533" alt="image" src="https://github.com/user-attachments/assets/454b05a6-dbe9-417e-8d90-841b07269c91" />

*Politiques FortiGate autorisant la sortie Internet depuis le réseau de transit et depuis la DMZ.*

### Objectif de cette étape

Cette étape garantit que FortiGate ne joue pas seulement un rôle de filtrage, mais aussi un rôle de **routage contrôlé** vers les réseaux internes. Elle prépare directement :
- la supervision Wazuh sur le LAN SOC ;
- la communication avec le LAN AD ;
- l’installation des composants nécessitant un accès Internet dans la DMZ et les réseaux internes.

Grâce à cette configuration, FortiGate devient le point de passage logique entre l’extérieur, la DMZ et les réseaux protégés situés derrière pfSense.


## 6. Configuration GUI pfSense

Après l’installation de base, pfSense a été configuré depuis son interface graphique afin d’assurer le routage correct entre les zones internes, de contrôler les flux autorisés et de préparer l’envoi des journaux de sécurité vers la plateforme Wazuh.

Cette phase comprend l’activation de l’interface **OPT1** pour le LAN AD, l’activation du **DNS Resolver**, l’ajout de règles firewall sur les interfaces **LAN SOC**, **LAN AD** et **WAN**, ainsi que la création d’une **route statique vers la DMZ**.

### Éléments configurés

- **Activation de l’interface OPT1** pour permettre le routage vers le réseau Active Directory.
- **Activation du DNS Resolver** afin que les machines internes puissent résoudre les noms de domaine et télécharger les paquets nécessaires.
- **Règles LAN SOC** pour autoriser les communications du réseau de supervision vers les autres segments.
- **Règles LAN AD** pour permettre aux machines Windows d’accéder à Internet et d’envoyer leurs logs vers Wazuh.
- **Règles WAN** pour n’autoriser que les connexions légitimes provenant du lien FortiGate ↔ pfSense.
- **Route statique vers la DMZ** pour que pfSense sache joindre le réseau 10.10.14.0/24 via FortiGate.

<img width="852" height="585" alt="image" src="https://github.com/user-attachments/assets/0799218a-ce14-4978-a5aa-5b96725bc7fa" />

*Activation de l’interface OPT1 dans pfSense pour relier et router le trafic vers le LAN Active Directory.*


### Logique de sécurité

La configuration appliquée sur pfSense suit une logique de cloisonnement. Le SOC doit pouvoir superviser l’ensemble de l’infrastructure, le LAN AD doit pouvoir communiquer avec la supervision et Internet, tandis que l’interface WAN de pfSense ne doit accepter que le trafic prévu depuis FortiGate.

<img width="857" height="646" alt="image" src="https://github.com/user-attachments/assets/9ea30030-a244-44b1-af06-da815b89b5cd" />
<img width="858" height="647" alt="image" src="https://github.com/user-attachments/assets/fb7deed5-1f20-4080-8008-1b4ea5e5ef4f" />

*Règles de filtrage pfSense appliquées aux interfaces internes afin d’autoriser la supervision SOC et les communications nécessaires du réseau AD.*

Sans la route statique vers la DMZ, pfSense ne pourrait pas renvoyer correctement le trafic vers le serveur Ubuntu hébergeant DVWA et Cowrie. Cette route est donc indispensable pour que les logs de la DMZ puissent remonter jusqu’à Wazuh.

<img width="858" height="285" alt="image" src="https://github.com/user-attachments/assets/25cf9afc-9a03-46bb-87ee-cb062b832862" />

*Route statique configurée sur pfSense pour joindre la DMZ via FortiGate et assurer le retour des flux vers le serveur Ubuntu exposé.*

### Objectif de cette étape

Cette étape transforme pfSense en véritable pare-feu interne de contrôle. Elle prépare directement :
- la connectivité entre la DMZ et le SOC ;
- l’enregistrement des agents Wazuh ;
- la remontée des logs de sécurité ;
- l’activation ultérieure de Suricata et son intégration avec Wazuh.

À ce stade, pfSense assure à la fois le routage interne, le cloisonnement des flux et la préparation de la collecte centralisée des événements de sécurité.

## 7. VM Ubuntu DMZ (DVWA + Cowrie)

La VM Ubuntu DMZ constitue le cœur du volet honeypot du projet. Elle héberge deux services intentionnellement exposés — **DVWA** (application web vulnérable) et **Cowrie** (honeypot SSH) — afin de capturer et d’analyser des tentatives d’attaques réelles depuis un environnement isolé.

### Caractéristiques de la VM

| Paramètre | Valeur |
|---|---|
| OS | Ubuntu 22.04 LTS Server |
| RAM | 2 Go |
| vCPU | 2 |
| Disque | 40 Go |
| NIC 1 | VMnet10 (DMZ) |

Une adresse IP statique a été configurée sur cette VM pour garantir un adressage cohérent au sein du réseau DMZ (10.10.14.0/24).

### Docker et isolation des services

Docker et Docker Compose ont été installés pour déployer DVWA dans un environnement isolé. Cette approche garantit que si un attaquant compromet l’application, il reste confiné dans le conteneur sans pouvoir directement atteindre le système hôte.

### Déploiement de DVWA avec ModSecurity

DVWA est déployée derrière un reverse proxy basé sur l’image officielle **OWASP ModSecurity** intégrant les règles **CRS (Core Rule Set)**. Ce composant analyse tout le trafic entrant avant de le transmettre à DVWA.

Le choix stratégique de cette configuration repose sur le mode **DetectionOnly** : ModSecurity n’intercepte ni ne bloque aucune requête, il se contente de journaliser les attaques. Cette approche est essentielle dans une logique de honeypot, où l’objectif est de laisser l’attaquant progresser tout en enregistrant l’intégralité de son activité pour alimenter Wazuh.

<img width="1217" height="586" alt="image" src="https://github.com/user-attachments/assets/ef831f89-fb5d-4ddb-810b-570f4df7739d" />

*Accès à DVWA via le reverse proxy ModSecurity, configuré en mode détection uniquement.*

**Paramètres clés de la configuration ModSecurity :**

- Niveau de sensibilité des règles CRS réglé pour un bon équilibre entre détection et faux positifs.
- Seuils de score d’anomalie définis avant déclenchement d’un log.
- Format de logs **JSON**, indispensable pour permettre à Wazuh de parser efficacement les événements.

### Architecture des conteneurs

| Conteneur | Rôle |
|---|---|
| db (MariaDB) | Base de données de DVWA (utilisateurs, sessions, challenges) |
| dvwa | Application vulnérable, exposée sur l’hôte DMZ via le port 8080 |
| modsecurity | Reverse proxy Nginx + CRS, exposé sur l’hôte DMZ via le port 80 |

<img width="1210" height="162" alt="image" src="https://github.com/user-attachments/assets/d822ff16-2348-4aa5-9104-17cb277a0333" />

*Conteneurs Docker actifs sur la VM DMZ : base de données MariaDB, application DVWA et reverse proxy ModSecurity.*

Le chemin des logs ModSecurity a été identifié précisément afin de configurer par la suite la collecte via l’agent Wazuh.

<img width="935" height="395" alt="image" src="https://github.com/user-attachments/assets/ef3d8024-fcbb-4e77-ac8f-8376298a45ff" />

*Extrait du fichier de logs ModSecurity au format JSON, utilisé plus tard par l’agent Wazuh.*

### Installation de Cowrie (honeypot SSH)

Cowrie simule un faux serveur SSH exposé sur le **port 22**. Chaque tentative de connexion, chaque commande tapée, et chaque fichier téléchargé par un attaquant est intégralement enregistré, ce qui permet de reconstituer le comportement complet d’un intrus.

Pour libérer le port 22 au profit de Cowrie, le véritable service SSH d’administration a été déplacé sur le **port 42333**, garantissant un accès administrateur discret et sécurisé.

Les logs générés par Cowrie sont stockés au format JSON dans un volume Docker dédié, chemin identifié et retenu pour la configuration ultérieure de l’agent Wazuh.

<img width="1650" height="315" alt="image" src="https://github.com/user-attachments/assets/b23c14bf-3634-4409-b315-8dbe8eb82239" />
<img width="1075" height="601" alt="image" src="https://github.com/user-attachments/assets/e71d05a4-7d9a-45ce-ae2e-396b22fbfd9b" />

*Honeypot Cowrie actif sur le port 22, avec ses logs JSON prêts à être collectés par Wazuh; et modif port ssh de la machine DMZ afin de tromper attaquant*

### Objectif de cette étape

Cette VM constitue le point d’attraction principal du laboratoire. Elle prépare directement :
- la génération d’alertes de sécurité réalistes (SQLi, XSS, brute force SSH) ;
- l’alimentation du SIEM Wazuh via les logs ModSecurity et Cowrie ;
- les scénarios de démonstration d’attaque présentés en fin de projet.

## 8. VM Wazuh Server sur LAN SOC

Le serveur Wazuh constitue le **cœur du SIEM** de cette architecture. Il centralise l’ensemble des logs de sécurité provenant de la DMZ, de FortiGate, de pfSense/Suricata et des machines Windows du LAN AD, afin de détecter, corréler et visualiser les événements de sécurité.

### Caractéristiques de la VM

La VM a été créée sous Ubuntu et configurée avec une adresse IP statique sur le réseau LAN SOC (10.10.20.0/24), garantissant un accès stable pour l’ensemble des agents qui s’y connecteront.

<img width="846" height="232" alt="image" src="https://github.com/user-attachments/assets/455a192b-5f06-4db7-9b80-10b403ea8827" />

*Configuration reseau de la vm wazuh.*


Avant l’installation, la connectivité de la VM a été validée dans les deux sens : vers pfSense (pour confirmer le bon fonctionnement du routage interne) et vers Internet (pour permettre le téléchargement des paquets nécessaires à l’installation).

### Installation de Wazuh All-in-One

L’installation a été réalisée avec le **script officiel Wazuh** (version 4.14), qui déploie en une seule opération les trois composants principaux de la stack :

- **Wazuh Manager** — moteur de collecte, d’analyse et de corrélation des logs.
- **Wazuh Indexer** — moteur d’indexation basé sur OpenSearch, chargé du stockage des événements.
- **Wazuh Dashboard** — interface web de visualisation et de gestion des agents.

<img width="830" height="357" alt="image" src="https://github.com/user-attachments/assets/e720c861-4bd0-4a81-98a7-afff7fc90dec" />
<img width="1297" height="185" alt="image" src="https://github.com/user-attachments/assets/6ee9f32c-15b5-44a4-8242-9ad00fb44734" />
<img width="1292" height="167" alt="image" src="https://github.com/user-attachments/assets/2bf85288-702b-414e-999b-a0e4c2cb70d4" />

*Vérification de l’état des services Wazuh (manager, indexer, dashboard), tous actifs et opérationnels.*

```bash
sudo apt update && sudo apt upgrade -y
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```
<img width="1310" height="707" alt="image" src="https://github.com/user-attachments/assets/d4ea0824-a566-4571-adfb-1d47029672b6" />
<img width="1215" height="787" alt="image" src="https://github.com/user-attachments/assets/87f531e6-1d13-4dab-bd5f-8bd8696927db" />

*Installation Wazuh all-in-one (script officiel).*


### Vérification des services

Une fois l’installation terminée, l’état des trois services critiques a été vérifié via `systemctl status`. Les trois services sont apparus **actifs (running)**, confirmant le bon déploiement de la stack :

- `wazuh-manager` — actif, avec l’ensemble des processus internes (analysisd, remoted, logcollector, etc.) démarrés.
- `wazuh-indexer` — actif, avec le moteur Java/OpenSearch opérationnel.
- `wazuh-dashboard` — actif, servant l’interface web sur le port HTTPS par défaut.

### Accès au Dashboard Wazuh

L’interface web a ensuite été testée depuis un navigateur. À ce stade, aucun agent n’était encore enregistré — le tableau de bord affichait donc l’écran d’accueil standard avec les compteurs d’alertes à zéro et l’invitation à déployer un premier agent.

<img width="1712" height="902" alt="image" src="https://github.com/user-attachments/assets/93068587-7635-40b9-a3f9-23433b21fee3" />

*Première connexion au Wazuh Dashboard, avant l’enregistrement de tout agent — aucune alerte critique ou élevée détectée.*

### Objectif de cette étape

Cette VM devient le point central de supervision du laboratoire. Les étapes suivantes consisteront à :
- valider la connectivité DMZ → SOC ;
- déployer les agents Wazuh sur chaque machine du laboratoire ;
- intégrer les logs FortiGate, pfSense/Suricata et Windows AD ;
- exploiter le Dashboard pour visualiser les alertes de sécurité en temps réel.

## 9. Connectivité DMZ to SOC

Avant d’enregistrer l’agent Wazuh sur la VM DMZ, il est indispensable de valider la connectivité complète entre la DMZ et le LAN SOC. L’agent doit pouvoir atteindre le manager Wazuh (10.10.20.10) sur les ports **TCP 1514** (communication agent↔manager) et **TCP 1515** (enregistrement initial).

### Test de connectivité initial

Un premier test de connectivité a été effectué depuis la VM DMZ en testant successivement chaque saut du chemin réseau :

| Étape | Cible | Résultat initial |
|---|---|---|
| 1 | 10.10.14.1 (FortiGate, interface DMZ) | ✅ OK |
| 2 | 10.10.50.2 (pfSense WAN via FortiGate) | ❌ KO |
| 3 | 10.10.20.1 (pfSense LAN) | ❌ KO |
| 4 | 10.10.20.10 (Wazuh) | ❌ KO |

Ce résultat a révélé que seul le premier saut fonctionnait : la DMZ atteignait bien FortiGate, mais le trafic ne parvenait pas à traverser pfSense pour rejoindre le SOC.

### Correction côté FortiGate

Une troisième politique pare-feu, **DMZ_to_SOC_Wazuh**, a été créée pour autoriser le trafic entrant depuis l’interface DMZ (port3) vers l’interface de transit TRANSIT_PFSENSE (port2), avec pour destination le sous-réseau SUB_SOC.

<img width="642" height="530" alt="image" src="https://github.com/user-attachments/assets/59e4736b-5085-4201-9ca2-fb864db3a05f" />

*Politique FortiGate autorisant le trafic depuis la DMZ (port3) vers le réseau de transit (port2), destination SUB_SOC.*

### Correction côté pfSense

Deux règles supplémentaires ont été nécessaires sur pfSense pour compléter la chaîne de connectivité :

- **Règle sur l’interface WAN, source 10.10.50.1** : autoriser le trafic provenant de FortiGate, arrivant sur le WAN de pfSense et routé vers les sous-réseaux LAN.

<img width="912" height="737" alt="image" src="https://github.com/user-attachments/assets/f2a42f49-80dd-4313-8cd9-9788591de14d" />

*Règle WAN pfSense autorisant FortiGate (10.10.50.1) à atteindre le LAN SOC.*

- **Règle sur l’interface WAN, source 10.10.14.0/24** : autoriser explicitement le réseau DMZ à atteindre les sous-réseaux LAN via FortiGate.

<img width="912" height="737" alt="image" src="https://github.com/user-attachments/assets/5327643b-33ba-4c71-9680-16477eb0f9c4" />

*Règle WAN pfSense autorisant le réseau DMZ (10.10.14.0/24) à atteindre le LAN SOC via FortiGate.*

### Explication technique du blocage

Le problème identifié provenait de l’absence de NAT sur le chemin DMZ → SOC. Lorsque la VM DMZ envoie un ping vers 10.10.20.10, FortiGate route le paquet vers pfSense **en conservant l’adresse IP source d’origine** (10.10.14.10) plutôt que de la traduire. pfSense voyait donc arriver, sur son interface WAN, un paquet avec une adresse source (10.10.14.10) qui ne correspondait à aucun réseau directement connecté à cette interface — d’où le blocage initial. Les deux nouvelles règles WAN ont permis d’autoriser explicitement ce chemin.

### Résultat final

Après application des correctifs sur FortiGate et pfSense, l’ensemble de la chaîne de connectivité DMZ → SOC a été validée avec succès : les tests ping vers pfSense LAN (10.10.20.1) et vers le manager Wazuh (10.10.20.10) aboutissent désormais sans aucune perte de paquets.

<img width="687" height="362" alt="image" src="https://github.com/user-attachments/assets/2a288804-7502-44b4-b8fd-ded3f3ddf51a" />

*Ping depuis srvdmz vers pfSense LAN (10.10.20.1) et vers le manager Wazuh (10.10.20.10) : connectivité DMZ → SOC pleinement opérationnelle.*

### Objectif de cette étape

Cette étape est une validation critique de bout en bout. Elle garantit que :
- le routage entre la DMZ et le SOC fonctionne dans les deux sens ;
- les règles de filtrage sur FortiGate et pfSense sont cohérentes avec l’absence de NAT ;
- l’agent Wazuh pourra effectivement s’enregistrer et transmettre ses logs.

## 10. Installation de l'agent Wazuh sur VM DMZ

Une fois la connectivité DMZ → SOC validée, l'agent Wazuh a été déployé sur la VM Ubuntu DMZ afin de collecter les journaux générés par ModSecurity (DVWA) et Cowrie, et de les transmettre au manager pour analyse et corrélation.

### Déploiement de l'agent

La commande d'installation a été générée directement depuis le Dashboard Wazuh, via l'assistant **Deploy new agent** : sélection du package Linux (DEB amd64), adresse du manager (10.10.14.10) et nom d'agent (SRV-DMZ).

<img width="1586" height="1705" alt="image" src="https://github.com/user-attachments/assets/f9dc6779-01a2-4192-b520-48c6d296d01f" />

*Assistant Wazuh générant la commande d'installation pour l'agent SRV-DMZ.*

L'agent a ensuite été téléchargé et installé sur la VM Ubuntu DMZ, puis démarré via systemd. Le service `wazuh-agent` apparaît actif avec l'ensemble de ses sous-processus (agentd, syscheckd, logcollector, modulesd).

<img width="1141" height="646" alt="image" src="https://github.com/user-attachments/assets/77af4962-a87e-4fc1-bf39-2833f82d1c8e" />

*Installation du paquet `.deb`, démarrage du service et confirmation de l'état `active (running)`.*

La connexion de l'agent a été confirmée depuis le Dashboard : l'agent **SRV-DMZ** (10.10.14.10) apparaît avec le statut **Active**.

<img width="1715" height="597" alt="image" src="https://github.com/user-attachments/assets/339f77b3-01d2-4c03-97e0-4aa7b0064db2" />

*Confirmation côté Dashboard Wazuh : l'agent SRV-DMZ est enregistré et actif.*

### Collecte des logs ModSecurity (DVWA)

Pour que Wazuh puisse exploiter les alertes générées par ModSecurity, plusieurs étapes ont été nécessaires :

1. **Vérification du fichier de log** : confirmation que `modsec_audit.log` existe bien dans le volume Docker et est lisible.

<img width="997" height="112" alt="image" src="https://github.com/user-attachments/assets/3fdb360a-415c-4281-869b-97f5814cfd40" />

*Le fichier `modsec_audit.log` est présent et accessible dans le volume Docker `dvwa_modsec_logs`.*

2. **Configuration de l'agent** : ajout d'un bloc `<localfile>` dans `ossec.conf` pointant vers le log ModSecurity au format JSON, avec un label `source=modsecurity-dvwa`.

<img width="1227" height="757" alt="image" src="https://github.com/user-attachments/assets/226160f1-89ca-4be9-9fa3-c45809187aeb" />

*Configuration du `<localfile>` ModSecurity dans `ossec.conf`, format JSON, avec label de source dédié.*

3. **Règles personnalisées côté manager** : création de règles dans `local_rules.xml` (groupe `modsecurity,dvwa,webattack`) pour détecter spécifiquement les attaques XSS, SQLi, LFI, RCE et les dépassements de score d'anomalie, avec mapping MITRE ATT&CK.

<img width="877" height="805" alt="image" src="https://github.com/user-attachments/assets/975c3d6a-c80d-48de-a2d2-5e2951496e85" />

*Règles personnalisées `local_rules.xml` générant des alertes distinctes par type d'attaque détectée (XSS, SQLi, LFI, RCE).*

### Collecte des logs Cowrie (JSON)

La même logique a été appliquée pour les logs du honeypot SSH Cowrie :

1. **Ajout du `<localfile>`** correspondant dans `ossec.conf`, pointant vers le fichier `cowrie.json` du volume Docker, avec le label `source=cowrie-ssh`.

<img width="1238" height="742" alt="image" src="https://github.com/user-attachments/assets/dc8a2c85-f0c1-43e7-b66a-2e92cda4f4ee" />

*Configuration du `<localfile>` Cowrie dans `ossec.conf`, ajoutée juste après le bloc ModSecurity.*

2. **Règles personnalisées côté manager** pour transformer les événements Cowrie bruts en alertes exploitables : activité SSH générale, login réussi/échoué, commande exécutée/inconnue, et téléchargement de fichier, chacune avec un niveau de sévérité et un mapping MITRE adapté.

<img width="897" height="827" alt="image" src="https://github.com/user-attachments/assets/35d723b4-4113-45b4-bd38-28ddb7e6d46e" />

*Règles personnalisées `local_rules.xml` pour Cowrie : login SSH réussi (niveau 10), commande exécutée (niveau 8), téléchargement de fichier (niveau 12).*

### Objectif de cette étape

Cette étape transforme la VM DMZ en véritable source de télémétrie pour le SOC. Elle prépare directement :
- la détection en temps réel des attaques SQLi/XSS sur DVWA ;
- la capture complète des sessions d'attaquants sur le honeypot Cowrie ;
- les scénarios de démonstration présentés en fin de projet.

## 11. Intégration des logs FortiGate dans Wazuh

Afin de superviser également le pare-feu périmétrique, les logs FortiGate ont été redirigés vers Wazuh via syslog UDP, en s'appuyant sur le lien de transit dédié entre FortiGate et pfSense.

### Étape 1 — Configurer FortiGate pour envoyer les logs vers Wazuh

Le paramétrage a été effectué depuis la CLI FortiGate, avec pour objectif d'envoyer les logs vers le serveur Wazuh (10.10.20.10) en UDP sur le port 514, en forçant la source IP à l'adresse du lien de transit (10.10.50.1) afin de respecter la topologie réseau du laboratoire.

<img width="646" height="321" alt="image" src="https://github.com/user-attachments/assets/be3f9aa7-0e0a-4df6-8fd0-cd3dc1c8f72b" />

*Vérification de la configuration syslog sur FortiGate via `show full-configuration log syslogd setting` : serveur 10.10.20.10, mode UDP, port 514, source-ip 10.10.50.1.*

### Étape 2 — Configuration côté Wazuh

#### 2.1 Activer la réception syslog dans Wazuh

Un bloc `<remote>` a été ajouté dans `ossec.conf` pour que le manager Wazuh écoute en syslog UDP sur le port 514 et n'accepte que les logs provenant de l'adresse FortiGate autorisée (10.10.50.1), via `<allowed-ips>`. Cette restriction garantit que seul le pare-feu légitime peut injecter des événements dans le pipeline de logs.

<img width="1014" height="792" alt="image" src="https://github.com/user-attachments/assets/2b0d6b93-5b37-4e3e-bda8-b26ad34a39ad" />

*Ajout du bloc `<remote>` en écoute syslog UDP/514, avec restriction `allowed-ips` à l'adresse du lien Transit FortiGate (10.10.50.1).*

#### 2.2 Vérifier que Wazuh reçoit bien les logs FortiGate

La réception a été confirmée directement dans le Dashboard Wazuh, où plusieurs types d'événements FortiGate apparaissent correctement décodés : détection d'attaque, détection de virus, session nettoyée, ou blocage d'URL appartenant à une catégorie interdite. Chaque événement est associé à un niveau de sévérité et un identifiant de règle Wazuh dédié (816xx).

<img width="1116" height="242" alt="image" src="https://github.com/user-attachments/assets/a03b8657-b22d-46db-8f17-5ea1d4d777ac" />

*Logs FortiGate décodés et affichés dans le Dashboard Wazuh, avec règles dédiées (attack detected, virus detected, session cleared, blocked URL).*

### Objectif de cette étape

Cette intégration permet de centraliser dans Wazuh la visibilité sur les attaques bloquées en amont par FortiGate (antivirus, IPS, filtrage d'URL), en complément des logs applicatifs de la DMZ et du honeypot. Le SOC dispose ainsi d'une vue corrélée entre le périmètre réseau et les couches applicatives.

## 12. Installation de Suricata et envoi des logs pfSense/Suricata vers Wazuh

Suricata a été installé comme IDS sur pfSense afin d'inspecter le trafic transitant entre FortiGate (10.10.50.1) et l'interface WAN de pfSense (10.10.50.2). Les alertes générées sont ensuite relayées vers Wazuh via syslog, au même titre que les logs firewall natifs de pfSense.

### Étape 1 — Installation du paquet Suricata sur pfSense

Le paquet `pfSense-pkg-suricata` a été installé depuis le gestionnaire de paquets de pfSense.

<img width="927" height="612" alt="image" src="https://github.com/user-attachments/assets/95724e87-459c-4c85-b55b-99c018674da7" />

*Confirmation de l'installation réussie du paquet pfSense-pkg-suricata.*

### Étape 2 — Activer Suricata sur l'interface WAN

Suricata a été activé sur l'interface WAN (em0) pour surveiller le trafic entre FortiGate et pfSense. Les alertes sont configurées pour être envoyées vers le journal système (`Send Alerts to System Log`) en facility LOCAL1, avec l'EVE JSON Log activé pour un format d'événement structuré.

<img width="1792" height="3604" alt="image" src="https://github.com/user-attachments/assets/eb2d34d5-2698-4547-91cb-bad57767cc96" />

*Activation de Suricata sur l'interface WAN avec envoi des alertes vers le syslog système et sortie EVE JSON.*

### Étape 3 — Activer l'envoi des logs pfSense (+ Suricata) vers Wazuh

pfSense a été configuré pour envoyer ses System Logs — incluant les alertes Suricata — vers le serveur Wazuh (10.10.20.10) en syslog UDP, avec les catégories "System Events" et "Firewall Events" cochées.

<img width="1247" height="796" alt="image" src="https://github.com/user-attachments/assets/49375149-4618-4a74-9d50-fdb219058ce2" />

*Activation du remote logging vers 10.10.20.10:514, incluant les événements firewall et système.*

### Étape 4 — Vérifier que pfSense envoie bien vers soc-wazuh-srv

Une capture réseau via `tcpdump` côté serveur Wazuh confirme la réception effective des paquets syslog UDP/514 en provenance de pfSense (10.10.20.1).

<img width="765" height="265" alt="image" src="https://github.com/user-attachments/assets/e5ab76ef-80b4-402b-83e1-a4dc5ef9e163" />

*Capture tcpdump confirmant la réception des paquets syslog UDP/514 depuis pfSense.*

### Étape 5 — Acceptation des logs dans Wazuh

Un second bloc `<remote>` a été ajouté dans `ossec.conf` pour accepter les logs syslog en provenance de pfSense (10.10.20.1), en complément du bloc existant pour FortiGate.

<img width="748" height="250" alt="image" src="https://github.com/user-attachments/assets/cbcf754a-de5b-4d88-8587-1e952edefcc5" />

*Ajout du second bloc `<remote>` pour accepter les logs syslog UDP/514 provenant de pfSense (10.10.20.1).*

### Étape 6 — Créer le decoder pfSense/Suricata

pfSense utilise un format de log firewall spécifique (filterlog) aux champs séparés par des virgules, tandis que Suricata journalise ses alertes dans un format distinct. Deux décodeurs personnalisés ont donc été créés dans Wazuh pour extraire correctement les champs (action, interface, IP source/destination, ports, classification, priorité) de ces deux formats.

<img width="1116" height="535" alt="image" src="https://github.com/user-attachments/assets/e21502a6-7c25-45d2-a3f2-29a81c6cf1c1" />
<img width="1027" height="221" alt="image" src="https://github.com/user-attachments/assets/82a0d349-0104-4519-b3db-da569da7d136" />

*Décodeurs personnalisés Wazuh pour extraire les champs des logs Suricata relayés via syslog pfSense.*

### Étape 7 — Créer les règles pfSense

Un jeu de règles a été créé dans `pfsense_rules.xml`, avec des niveaux de sévérité différenciés selon le contexte : trafic bloqué ou autorisé, trafic bloqué depuis la DMZ, trafic bloqué depuis le segment Transit FortiGate, ou connexion vers le LAN SOC depuis une zone inconnue. Sans ces règles à niveau >= 3, les événements n'apparaissent pas dans le dashboard Wazuh.

<img width="975" height="821" alt="image" src="https://github.com/user-attachments/assets/d0448d2f-26bb-4171-84fb-9e44119cd336" />

*Règles personnalisées pfSense dans Wazuh : trafic bloqué/autorisé, trafic DMZ, trafic Transit FortiGate, accès LAN SOC.*

### Vérification finale

Le Dashboard Wazuh confirme la bonne réception et le décodage des événements pfSense, avec les libellés "Traffic ALLOWED on em0" et "Traffic BLOCKED on em0" associés à leurs identifiants de règle respectifs (200101, 200102).

<img width="1170" height="177" alt="image" src="https://github.com/user-attachments/assets/84615027-2526-449b-a62a-10fd34da0942" />

*Événements pfSense (trafic autorisé/bloqué) correctement décodés et affichés dans le Dashboard Wazuh.*

## 13. Activation des catégories de règles Suricata (ET Open Rules)

En complément de l'activation de base de Suricata, un jeu de catégories de règles "Emerging Threats Open" a été sélectionné pour cibler spécifiquement les menaces pertinentes dans l'architecture du laboratoire (reconnaissance vers l'AD, exploitation SMB, attaques web contre DVWA, brute force RDP, etc.).

### Étape — Activer les catégories ET dans WAN Categories

Les catégories de règles ont été activées sur l'interface WAN afin de couvrir les scénarios d'attaque les plus représentatifs du projet : scan de ports (T1046), exploitation EternalBlue/SMB/RPC (T1210), kits d'exploitation Windows, shellcode, DoS vers AD/DNS, malwares en transit, SQL injection (pour DVWA), attaques web server, exploits navigateur, propagation latérale et brute force RDP/VNC.

<img width="926" height="812" alt="image" src="https://github.com/user-attachments/assets/6e113c26-9d72-41be-b3a4-593db1673a7b" />

*Activation des catégories de règles ET Open Rules sur l'interface WAN de Suricata, couvrant scan réseau, exploitation SMB, attaques web et brute force.*

### Objectif de cette étape

Cette sélection ciblée de catégories permet à Suricata de générer des alertes pertinentes pour les scénarios d'attaque simulés dans le laboratoire (reconnaissance, exploitation SMB/RDP, SQLi sur DVWA), sans surcharger le moteur avec des règles hors périmètre.

## 14. LAN AD & Client

Cette section couvre la mise en place du contrôleur de domaine Active Directory ainsi que le rattachement d'un poste client Windows au domaine, sur le segment LAN AD (10.10.30.0/24).

### 1. Configuration du serveur AD

#### 1.1 Configurer l'adresse IP statique

Une adresse IP statique a été attribuée au serveur AD (10.10.30.10), avec 10.10.30.1 comme passerelle et 8.8.8.8 comme DNS temporaire avant la promotion en contrôleur de domaine.

<img width="395" height="452" alt="image" src="https://github.com/user-attachments/assets/f6f1b848-9898-45d1-8ea3-ed2eaf656e4e" />

*Adressage IPv4 statique du serveur AD : 10.10.30.10/24, passerelle 10.10.30.1.*

#### Test de connectivité

La connectivité du DC a été vérifiée vers la passerelle (10.10.30.1), le serveur SOC (10.10.20.10) et Internet (8.8.8.8) avant de poursuivre l'installation des rôles.

<img width="492" height="630" alt="image" src="https://github.com/user-attachments/assets/97a8c4b2-01c5-42ca-a336-d50d31425b67" />

*Vérification de la connectivité du DC vers la passerelle, le LAN SOC et Internet.*

#### 1.2 Installer les rôles AD DS et DNS

Les rôles **Active Directory Domain Services (AD DS)**, **DNS Server** et **Group Policy Management** ont été installés via le gestionnaire de serveur Windows.

<img width="822" height="615" alt="image" src="https://github.com/user-attachments/assets/b4b64028-23bf-4cb8-8fa4-a333bc74859d" />

*Confirmation d'installation des rôles AD DS, DNS Server et Group Policy Management.*

#### 1.3 Promouvoir le serveur en contrôleur de domaine

Le serveur a été promu en contrôleur de domaine, créant la nouvelle forêt `lan.entreprise` (NetBIOS: LAN), avec le rôle DNS Server et le catalogue global activés.

<img width="1027" height="725" alt="image" src="https://github.com/user-attachments/assets/85511c38-3d61-44f5-8951-44829fd02514" />

*Récapitulatif de la promotion en contrôleur de domaine : forêt lan.entreprise, DNS Server activé, catalogue global.*

#### 1.4 Créer quelques comptes utilisateurs

Un compte utilisateur de test (`user01`) a été créé dans l'unité d'organisation `OU_USERS` de l'annuaire Active Directory.

<img width="750" height="526" alt="image" src="https://github.com/user-attachments/assets/ce8483bf-b860-4e75-a75d-c9ffa1126002" />

*Création du compte utilisateur user01 dans l'OU dédiée (lan.entreprise/OU_USERS).*

### 2. Configuration du client "user01"

#### 2.1/2.2 Configuration VM et réseau

Le poste client Windows a été configuré avec une adresse IP statique (10.10.30.20), la passerelle 10.10.30.1 et le serveur AD (10.10.30.10) comme serveur DNS préféré.

<img width="390" height="450" alt="image" src="https://github.com/user-attachments/assets/d5b2ea98-84ef-4377-afed-faeb2b49cfa6" />

*Adressage IPv4 du client : 10.10.30.20/24, DNS pointant vers le contrôleur de domaine (10.10.30.10).*

#### 2.3 Tester la résolution DNS

La résolution DNS du domaine `lan.entreprise` a été validée depuis le poste client via `nslookup`, confirmant la bonne prise en charge par le serveur DNS du DC.

<img width="637" height="317" alt="image" src="https://github.com/user-attachments/assets/602b1c0b-00b6-4bcd-b492-a1b5031fdc2c" />

*Ping vers le DC et résolution DNS réussie du domaine lan.entreprise via nslookup.*

#### 2.4 Joindre le domaine

Le poste client a rejoint avec succès le domaine `lan.entreprise`.

#### Vérification côté DC

La jonction a été confirmée côté contrôleur de domaine : l'objet ordinateur `USER01` apparaît dans le conteneur Computers d'Active Directory Users and Computers, avec son nom DNS complet `user01.lan.entreprise`.

<img width="752" height="436" alt="image" src="https://github.com/user-attachments/assets/39232d52-37b3-4365-a2de-dc6b49ab5cbb" />

*Objet ordinateur USER01 visible dans Active Directory Users and Computers, confirmant la jonction réussie au domaine.*

### Objectif de cette étape

Cette étape établit l'infrastructure d'annuaire du laboratoire, indispensable pour simuler des scénarios d'attaque réalistes contre un environnement Windows d'entreprise (reconnaissance AD, mouvement latéral, brute force RDP) et pour tester la collecte de logs Windows/Sysmon dans les sections suivantes.

## 15. Installer l'agent Wazuh sur le DC et le client Windows

Après la mise en place du domaine Active Directory, les agents Wazuh ont été déployés sur le contrôleur de domaine et sur le poste client Windows, en s'appuyant sur l'assistant de déploiement du Dashboard Wazuh (Agents Management → Summary → Deploy new agent).

### 1. Agent 1 — Domain Controller (AD-SRV)

Le paquet MSI Windows a été sélectionné avec le serveur Wazuh (10.10.20.10) comme adresse de destination et `AD-SRV` comme nom d'agent unique.

<img width="977" height="1637" alt="image" src="https://github.com/user-attachments/assets/f47ec8fd-cf6b-4fa9-a672-dbc6fbe680ac" />

*Assistant de déploiement Wazuh configuré pour le DC : serveur 10.10.20.10, nom d'agent AD-SRV.*

La commande PowerShell générée a été exécutée sur le contrôleur de domaine, téléchargeant et installant l'agent, puis démarrant le service Wazuh avec succès.

<img width="1217" height="226" alt="image" src="https://github.com/user-attachments/assets/bfb7fddd-e4e8-4f00-aa6b-da9df1585ca3" />

*Installation et démarrage réussi du service Wazuh sur le contrôleur de domaine (AD-SRV).*

### 2. Agent 2 — Client Windows (user01-entreprise)

La même procédure a été appliquée au poste client Windows, avec `user01-entreprise` comme nom d'agent, toujours pointé vers le manager 10.10.20.10.

<img width="977" height="1637" alt="image" src="https://github.com/user-attachments/assets/fb4c108a-6820-4bc5-974b-2f19dcc7ba57" />

*Assistant de déploiement Wazuh configuré pour le client Windows : serveur 10.10.20.10, nom d'agent user01-entreprise.*

L'agent a été installé via PowerShell, et le service `WazuhSvc` a été confirmé comme étant à l'état `Running`.

<img width="1217" height="307" alt="image" src="https://github.com/user-attachments/assets/9d9e5cd0-e5b0-484b-b676-f5d7176d7881" />

*Installation de l'agent sur le poste client, avec vérification du service WazuhSvc à l'état Running.*

### 3. Vérification finale côté Manager

La commande `agent_control -l` exécutée sur le serveur Wazuh confirme la présence des trois agents attendus : le serveur lui-même, ainsi que AD-SRV et user01-entreprise à l'état Active.

<img width="666" height="167" alt="image" src="https://github.com/user-attachments/assets/b6cb72a2-070d-4ee4-88bd-d980c1d7c58b" />

*Vérification en ligne de commande des agents enregistrés : AD-SRV et user01-entreprise à l'état Active.*

Le Dashboard Wazuh confirme visuellement le même résultat, avec le détail des systèmes d'exploitation détectés : Ubuntu 24.04.4 LTS pour SRV-DMZ (déconnecté), Windows Server 2019 pour AD-SRV, et Windows 10 Pro pour user01-entreprise, tous les deux actifs.

<img width="1212" height="400" alt="image" src="https://github.com/user-attachments/assets/ce705923-d5c6-4d37-9940-e602ead97fcd" />

*Vue Dashboard des trois agents enregistrés, avec systèmes d'exploitation et statuts respectifs.*

### Objectif de cette étape

Le déploiement des agents sur le DC et le poste client Windows complète la couverture de supervision du laboratoire sur l'ensemble des zones réseau (DMZ, SOC, AD), permettant de collecter les journaux Windows et Sysmon nécessaires à la détection des scénarios d'attaque contre l'annuaire et les postes de travail.


## 16. Collecte des logs AD et Windows Client

Afin d'enrichir la visibilité de Wazuh sur l'annuaire Active Directory et le poste client, trois axes ont été mis en place : le durcissement des politiques d'audit GPO, l'installation de Sysmon pour la télémétrie système, et la configuration des agents Wazuh pour collecter les canaux d'événements pertinents.[file:1355]

### Collecte logs AD

#### Étape 1 — Audit Policies GPO sur le Domain Controller

Les politiques d'audit avancées ont été configurées via la Default Domain Policy afin de journaliser les événements critiques liés à l'authentification, la gestion des comptes et l'accès à l'annuaire.[file:1355]

| Catégorie | Sous-catégorie | Valeur |
|---|---|---|
| Account Logon | Audit Credential Validation | Success + Failure |
| Account Management | Audit User Account Management | Success + Failure |
| Account Management | Audit Security Group Management | Success |
| Logon/Logoff | Audit Logon | Success + Failure |
| Logon/Logoff | Audit Account Lockout | Success + Failure |
| DS Access | Audit Directory Service Access | Success |
| Privilege Use | Audit Sensitive Privilege Use | Success + Failure |
| Policy Change | Audit Audit Policy Change | Success |

<img width="407" height="837" alt="image" src="https://github.com/user-attachments/assets/e6c7ac8c-08de-4e18-b2fd-a5a4525956e9" />

*Arborescence des Advanced Audit Policy Configuration dans le Group Policy Management Editor de la Default Domain Policy.*

#### Étape 2 — Sysmon sur AD-SRV

Sysmon a été téléchargé, extrait, puis installé sur AD-SRV avec la configuration communautaire recommandée SwiftOnSecurity, afin de tracer les processus, connexions réseau et accès fichiers.

<img width="731" height="281" alt="image" src="https://github.com/user-attachments/assets/1a82804e-dc9d-4869-b09a-c61b83f03f90" />

*Téléchargement, extraction et installation de Sysmon avec la configuration SwiftOnSecurity via PowerShell.*
*Sysmon64 et SysmonDrv installés et démarrés avec succès sur AD-SRV.*

L'installation confirme le chargement du schéma de configuration version 4.50 et le démarrage réussi des services SysmonDrv et Sysmon64.


#### Étape 3 — Configuration de l'agent Wazuh pour collecter les logs AD

Le fichier `ossec.conf` de l'agent (`C:\Program Files (x86)\ossec-agent\ossec.conf`) a d'abord été enrichi avec les canaux Security, Sysmon et PowerShell.

<img width="1081" height="777" alt="image" src="https://github.com/user-attachments/assets/1acf5eb4-075e-4980-bef0-6d60e6c0c09d" />

*Configuration initiale de l'agent Wazuh avec les canaux Security, Sysmon-Operational et PowerShell-Operational.*

Le canal Directory Service a ensuite été ajouté pour capturer spécifiquement les événements liés aux opérations sur l'annuaire Active Directory.


### Objectif de cette étape

Le durcissement des politiques d'audit combiné à Sysmon et à la collecte multi-canal via l'agent Wazuh permet de détecter des comportements suspects tels que les tentatives de brute force, la création de comptes, les élévations de privilèges et les modifications de l'annuaire, éléments essentiels pour les scénarios d'attaque ciblant l'Active Directory.

### Collecte logs user01

#### Étape 1 — Sysmon sur user01

La même procédure d'installation de Sysmon (téléchargement, extraction, configuration SwiftOnSecurity) a été appliquée au poste client Windows user01, afin d'obtenir une télémétrie système équivalente à celle du contrôleur de domaine.

<img width="1037" height="511" alt="image" src="https://github.com/user-attachments/assets/08c94916-148d-4fcd-8bf7-ed0353b5d5b0" />

*Installation de Sysmon avec la configuration SwiftOnSecurity sur le poste client user01, services SysmonDrv et Sysmon64 démarrés avec succès.*

#### Étape 2 — Configuration de l'agent Wazuh sur user01

Le fichier `ossec.conf` de l'agent (`C:\Program Files (x86)\ossec-agent\ossec.conf`) a été enrichi avec les canaux Security, Sysmon-Operational et PowerShell-Operational, sans bloc Directory Service puisque user01 n'est pas un contrôleur de domaine.

<img width="1002" height="302" alt="image" src="https://github.com/user-attachments/assets/f3360ac4-01a6-4f95-9077-84aa008dea3c" />

*Configuration de l'agent Wazuh sur user01 avec les canaux Security, Sysmon et PowerShell activés.*

### Objectif de cette étape

La collecte homogène des logs Sysmon, Security et PowerShell sur AD-SRV et user01 permet à Wazuh de corréler les événements entre le contrôleur de domaine et le poste client, un prérequis indispensable pour détecter des scénarios de mouvement latéral ou d'escalade de privilèges dans le laboratoire.


## 17. Démonstration

Cette section présente les résultats concrets du laboratoire à travers cinq scénarios d'attaque exécutés contre la DMZ, l'infrastructure réseau et l'AD, chacun démontrant la capacité de détection de la chaîne de sécurité (ModSecurity, Cowrie, pfSense, FortiGate, Wazuh) mise en place dans les sections précédentes.

### Scénario 1 — Injection SQL

Une injection SQL classique (`' or '1' = '1`) a été soumise via le paramètre `ARGS:id` de DVWA, hébergé derrière le reverse proxy ModSecurity sur srvdmz. La requête malveillante a été interceptée par les règles OWASP CRS `REQUEST-920-PROTOCOL-ENFORCEMENT` et `REQUEST-942-APPLICATION-ATTACK-SQLI`, avec détection par libinjection.

Le détail de l'alerte capturée par Wazuh montre l'adresse IP source de l'attaquant (10.10.14.50), le port cible (8080) et le contenu exact de la charge malveillante détectée.

<img width="1295" height="60" alt="image" src="https://github.com/user-attachments/assets/a00e2729-45fc-47b0-91fa-effd67a0d5e6" />
<img width="937" height="750" alt="image" src="https://github.com/user-attachments/assets/2922526f-92c4-4fd8-9cf6-0177c4b8c474" />


*Détail du document Wazuh montrant la détection SQLi par ModSecurity : IP source 10.10.14.50, règle REQUEST-942-APPLICATION-ATTACK-SQLI, correspondance libinjection.*

### Scénario 2 — Cross-Site Scripting (XSS)

Un payload XSS réfléchi (`<script>alert('Test xss Demo')</script>`) a été injecté dans le champ de saisie de la page de vulnérabilité DVWA dédiée au Reflected XSS, démontrant l'absence de filtrage applicatif natif de l'application volontairement vulnérable.

<img width="895" height="382" alt="image" src="https://github.com/user-attachments/assets/d9aad4fd-6b62-4009-946f-14b508c8e3cd" />
<img width="942" height="756" alt="image" src="https://github.com/user-attachments/assets/8fed14e8-5921-49d2-9e02-f6bb47071926" />


*Soumission d'un payload XSS réfléchi dans le module Reflected Cross Site Scripting de DVWA.*

### Scénario 3 — Tentative SSH sur le honeypot Cowrie

Une connexion SSH root a été tentée depuis Kali vers l'adresse 10.10.14.10, correspondant en réalité au honeypot Cowrie et non à un véritable service SSH. L'attaquant a pu se connecter, exécuter `whoami`, consulter `/etc/passwd`, puis quitter la session — toutes ces actions ayant été journalisées en détail par Cowrie sans jamais compromettre le système réel.

La chronologie des événements Wazuh reconstruit fidèlement la session : connexion SSH réussie (règle 100201, niveau 10), exécution des commandes `ls` et `exit` (règle 100203), puis fermeture de session (règle 100205). Ces alertes sont corrélées dans le temps avec la détection ModSecurity du scénario SQLi, illustrant la vision unifiée apportée par Wazuh sur l'ensemble des surfaces d'attaque de la DMZ.[file:1354]

| Horodatage | Agent | Source | Description de la règle | Niveau | ID règle |
|---|---|---|---|---|---|
| 18:40:48 | SRV-DMZ | cowrie-ssh | Session fermée depuis 10.10.14.10 | 3 | 100205 |
| 18:40:48 | SRV-DMZ | cowrie-ssh | Commande exécutée: exit | 8 | 100203 |
| 18:40:46 | SRV-DMZ | cowrie-ssh | Commande exécutée: ls | 8 | 100203 |
| 18:40:30 | SRV-DMZ | cowrie-ssh | Login SSH réussi | 10 | 100201 |
| 18:40:14 | SRV-DMZ | modsecurity-dvwa | Score anomalie élevé | 10 | 110005 |
| 18:40:01 | SRV-DMZ | modsecurity-dvwa | SQLi détectée | 8 | 110002 |

<img width="707" height="707" alt="image" src="https://github.com/user-attachments/assets/e845ee14-ba6c-42ce-ab96-9a874a1e66da" />

*Terminal Kali interagissant avec le honeypot Cowrie : commandes whoami, cat /etc/passwd, puis exit, toutes journalisées.*

<img width="1247" height="587" alt="image" src="https://github.com/user-attachments/assets/e98ed5bc-38bf-476b-a42b-92da60c9ea2c" />

*Timeline Wazuh corrélant les alertes Cowrie (SSH) et ModSecurity (SQLi) sur la même fenêtre temporelle.*

### Scénario 4 — Journalisation du trafic pfSense (Traffic allowed / blocked)

Ce scénario démontre l'intégration des logs de filtrage pfSense dans Wazuh, permettant de visualiser en temps réel les décisions du pare-feu sur l'interface `em0`. Deux types d'événements sont capturés et différenciés : le trafic autorisé (Traffic allowed on em0) et le trafic bloqué (Traffic blocked on em0), reflétant l'application effective des règles de filtrage configurées sur pfSense.

Cette visibilité permet de corréler les tentatives de connexion rejetées par le pare-feu — par exemple lors d'un scan réseau ou d'une tentative d'accès non autorisée — avec les autres alertes de sécurité générées ailleurs dans l'infrastructure (DMZ, AD), offrant une vue consolidée du périmètre réseau directement depuis le Dashboard Wazuh.

<img width="1170" height="177" alt="image" src="https://github.com/user-attachments/assets/0facc11b-8872-4332-a2c5-a8ddb47b542b" />

*Module Threat Hunting de Wazuh affichant les événements pfSense : Traffic allowed on em0 et Traffic blocked on em0.*

### Scénario 5 — Décodage des logs FortiGate dans Wazuh

Ce dernier scénario illustre l'intégration des logs FortiGate dans Wazuh via des règles de décodage dédiées, permettant de traduire les événements bruts du pare-feu périmétrique en alertes structurées et exploitables. Quatre catégories de règles spécifiques ont été validées : détection d'attaque (attack detected), détection de virus (virus detected), fin de session (session cleared) et blocage d'URL (blocked URL).

Cette catégorisation permet à l'équipe SOC de distinguer rapidement la nature de chaque événement FortiGate sans avoir à interpréter manuellement les logs syslog bruts, et de prioriser les investigations selon la criticité du type d'alerte (une détection de virus ou d'attaque nécessitant une réponse plus rapide qu'un simple blocage d'URL par filtrage de contenu).

<img width="1116" height="242" alt="image" src="https://github.com/user-attachments/assets/e3e65d73-fc6b-4463-a276-ba0caffd4f1e" />

*Dashboard Wazuh affichant les logs FortiGate décodés avec les règles dédiées : attack detected, virus detected, session cleared, blocked URL.*

### Synthèse de la démonstration

Les cinq scénarios valident la chaîne de détection complète du laboratoire à tous les niveaux de l'infrastructure : ModSecurity intercepte les attaques applicatives (SQLi, XSS) au niveau du reverse proxy, Cowrie capture intégralement les tentatives d'intrusion SSH via le honeypot, tandis que pfSense et FortiGate remontent leurs événements de filtrage réseau et de sécurité périmétrique. Wazuh centralise l'ensemble de ces sources hétérogènes — applicative, système et réseau — dans une timeline unifiée, démontrant la capacité du SOC à corréler des événements provenant de couches de sécurité complémentaires pour une détection et une réponse aux incidents plus complètes.

## Conclusion générale

Ce projet a permis de concevoir, déployer et valider un mini-SOC complet reproduisant les composantes essentielles d'une infrastructure d'entreprise moderne : une zone démilitarisée exposée (DMZ) hébergeant des services volontairement vulnérables, un réseau interne structuré autour d'un Active Directory, une double couche de pare-feu (FortiGate en périmétrique, pfSense en interne), et une plateforme de supervision centralisée basée sur Wazuh.

La segmentation réseau opérée via VMware, avec des VMnets dédiés à chaque zone de confiance (WAN, DMZ, LAN SOC, LAN AD, Transit), a permis de reproduire fidèlement les principes de cloisonnement recommandés en environnement professionnel, tout en conservant un laboratoire totalement isolé et maîtrisable. Cette architecture a démontré sa pertinence tout au long du projet, en particulier lors des tests d'intrusion où chaque zone a réagi de manière cohérente avec son rôle attendu.

Le déploiement d'un service vulnérable (DVWA) associé à un reverse proxy ModSecurity, ainsi que d'un honeypot SSH (Cowrie), a permis d'exposer une surface d'attaque réaliste sans jamais mettre en péril l'intégrité des systèmes réels. Les scénarios de démonstration ont confirmé que ces mécanismes de leurre et de filtrage applicatif interceptent efficacement les attaques courantes — injection SQL, XSS, tentatives de compromission SSH — tout en produisant des journaux exploitables pour l'analyse.

Sur le plan de la supervision, l'intégration progressive des agents Wazuh sur l'ensemble des machines (serveur DMZ, contrôleur de domaine, poste client Windows) combinée à un durcissement des politiques d'audit GPO et au déploiement de Sysmon a considérablement enrichi la visibilité sur les événements systèmes et applicatifs. La collecte et le décodage des journaux FortiGate et pfSense ont complété cette visibilité au niveau réseau, permettant une corrélation fine entre les couches applicative, système et périmétrique au sein d'un même tableau de bord.

Les cinq scénarios de démonstration ont validé de bout en bout la chaîne de détection : chaque type d'attaque, qu'elle vise l'application web, le service SSH ou le périmètre réseau, a été détecté, journalisé et correctement catégorisé par Wazuh, avec une granularité allant jusqu'à la commande exacte exécutée par l'attaquant. Cette capacité de reconstruction chronologique démontre la valeur opérationnelle d'un SOC bien instrumenté, tant pour la détection en temps réel que pour l'investigation forensique a posteriori.


### Perspectives d'amélioration

Plusieurs axes pourraient enrichir ce laboratoire dans une itération future :

- Étendre l'automatisation Shuffle à davantage de scénarios (blocage automatique sur FortiGate, désactivation de comptes AD compromis, notification par email/Slack, création de tickets dans un outil de gestion d'incidents)
- Enrichir les workflows Shuffle existants avec des logiques conditionnelles plus poussées (score de gravité, whitelist d'IP, escalade automatique selon le niveau d'alerte Wazuh)
- Ajouter des règles de corrélation Wazuh personnalisées pour détecter des chaînes d'attaque multi-étapes (par exemple reconnaissance réseau suivie d'une tentative de brute force)
- Intégrer un SIEM de threat intelligence (par exemple via des flux MISP) pour enrichir automatiquement les alertes avec du contexte externe
- Simuler des scénarios de mouvement latéral plus avancés au sein de l'Active Directory (Kerberoasting, Pass-the-Hash) pour tester la détection Sysmon/Wazuh sur ce type de menace
- Mettre en place des tableaux de bord dédiés par équipe (SOC Tier 1, Tier 2, management) pour adapter le niveau de détail présenté à chaque profil d'utilisateur
- Migrer une partie de l'infrastructure vers le cloud (AWS, Azure ou GCP) afin de tester la supervision Wazuh sur des ressources cloud natives (VPC, groupes de sécurité, instances managées) et valider la portabilité de l'architecture au-delà du laboratoire local
- Déployer les agents Wazuh sur des services cloud managés (bases de données, conteneurs, fonctions serverless) pour étendre la visibilité à des environnements hybrides on-premise/cloud
- Exploiter les journaux natifs des fournisseurs cloud (CloudTrail sur AWS, Activity Log sur Azure) et les intégrer dans Wazuh pour une corrélation unifiée entre les couches on-premise et cloud


