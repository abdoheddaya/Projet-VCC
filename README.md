
# Atelier_Sécurité des endpoints et supervision SIEM : étude de cas multi-OS (Linux &amp; Windows)


**Filière :** Ingénierie Informatique - Big Data et Cloud Computing (II-BDCC)

**Réalisé par :** Abderrahmane HEDDAYA  
**Encadré par :** Prof. Azeddine KHIAT  
**Année universitaire :** 2025-2026

----------

## Table des matières

1.  [Introduction](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#1-introduction)
2.  [Architecture globale](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#2-architecture-globale)
3.  [Création des instances AWS](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#3-cr%C3%A9ation-des-instances-aws)
4.  [Installation du serveur Wazuh](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#4-installation-du-serveur-wazuh)
5.  [Déploiement des agents](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#5-d%C3%A9ploiement-des-agents)
6.  [Installation et configuration de Sysmon (Windows)](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#6-installation-et-configuration-de-sysmon-windows)
7.  [Tests et génération d'événements](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#7-tests-et-g%C3%A9n%C3%A9ration-d%C3%A9v%C3%A9nements)
8.  [Analyse comparative : SIEM vs EDR](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#8-analyse-comparative-siem-vs-edr)
9.  [Conclusion](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#9-conclusion)
10.  [Annexes](https://claude.ai/chat/292e19f7-1410-4e7d-970e-f2aa3288db45#10-annexes)

----------

## 1. Introduction

### 1.1 Contexte général

Dans un contexte de cybersécurité en constante évolution où les menaces deviennent de plus en plus sophistiquées, la mise en place d'une solution de détection et de réponse aux incidents est devenue indispensable pour toute infrastructure informatique. Les organisations font face quotidiennement à des tentatives d'intrusion, des attaques par déni de service, des ransomwares et d'autres menaces qui peuvent compromettre la confidentialité, l'intégrité et la disponibilité de leurs systèmes.

### 1.2 Objectifs du projet

Ce projet vise la mise en place d'une solution complète de supervision de sécurité basée sur **Wazuh**, un système SIEM (Security Information and Event Management) et EDR (Endpoint Detection and Response) open source, déployée sur l'infrastructure cloud **AWS EC2**.

**Les objectifs spécifiques sont les suivants :**

-   Déployer une architecture de sécurité complète dans le cloud AWS
-   Configurer un serveur Wazuh central pour la collecte et l'analyse des événements
-   Enrôler et superviser des endpoints Linux et Windows
-   Implémenter des mécanismes de détection d'intrusion et de monitoring d'intégrité
-   Tester différents scénarios d'attaque et valider les capacités de détection
-   Développer des compétences en threat hunting et investigation d'incidents

### 1.3 Architecture déployée

L'infrastructure comprend :

-   Un **serveur Wazuh central** (Ubuntu 22.04) hébergeant le Manager, l'Indexer et le Dashboard
-   Un **client Linux** (Ubuntu 22.04) avec l'agent Wazuh
-   Un **client Windows** (Windows Server 2022) avec l'agent Wazuh et Sysmon

----------

## 2. Architecture globale

### 2.1 Vue d'ensemble

L'architecture déployée repose sur le modèle client-serveur où un serveur Wazuh central collecte, analyse et corrèle les événements de sécurité provenant de multiples endpoints.

### 2.2 Schéma d'architecture

![Schéma d'architecture du projet Wazuh SIEM/EDR](https://claude.ai/chat/architecture-schema.png)

_Figure 1 : Schéma d'architecture du projet Wazuh SIEM/EDR_

### 2.3 Composants de l'architecture

#### 2.3.1 Serveur Wazuh (EC2 Ubuntu)

**Rôle :** Serveur central de supervision de sécurité

**Composants installés :**

-   **Wazuh Manager :** Moteur d'analyse et de corrélation des événements
-   **Wazuh Indexer :** Base de données Elasticsearch pour le stockage
-   **Wazuh Dashboard :** Interface web de visualisation et d'investigation

**Spécifications :**

-   Instance type : t3.large (2 vCPUs, 8 GiB RAM)
-   OS : Ubuntu Server 22.04 LTS
-   Stockage : 30 GiB SSD (gp3)

#### 2.3.2 Client Linux (EC2 Ubuntu)

**Rôle :** Endpoint Linux supervisé

**Composants :**

-   Agent Wazuh pour la collecte d'événements
-   Monitoring d'intégrité de fichiers (FIM)
-   Détection d'intrusion (HIDS)

**Spécifications :**

-   Instance type : t2.micro (1 vCPU, 1 GiB RAM)
-   OS : Ubuntu Server 22.04 LTS
-   Stockage : 8 GiB SSD

#### 2.3.3 Client Windows (EC2 Windows Server)

**Rôle :** Endpoint Windows supervisé avec monitoring avancé

**Composants :**

-   Agent Wazuh pour Windows
-   Sysmon (System Monitor) pour les événements système avancés
-   Collecte des événements Windows Security

**Spécifications :**

-   Instance type : t2.medium (2 vCPUs, 4 GiB RAM)
-   OS : Windows Server 2022
-   Stockage : 30 GiB SSD

### 2.4 Flux de communication

#### 2.4.1 Agents vers Serveur

-   **Port 1514/TCP :** Envoi des logs et événements de sécurité
-   **Port 1515/TCP :** Enrôlement initial et configuration des agents
-   **Protocole :** Communication chiffrée avec authentification par certificats

#### 2.4.2 Accès administrateur

-   **Port 443/HTTPS :** Accès au Dashboard Wazuh (interface web)
-   **Port 22/SSH :** Administration du serveur et client Linux
-   **Port 3389/RDP :** Administration du client Windows

----------

## 3. Création des instances AWS

### 3.1 Instance 1 : Serveur Wazuh

#### 3.1.1 Lancement de l'instance

1.  Dans la console AWS, rechercher "EC2" dans la barre de recherche
2.  Cliquer sur "Instances" dans le menu latéral
3.  Cliquer sur "Launch instances" (bouton orange)

#### 3.1.2 Configuration détaillée

**Nom et tags :**

```
Name: Wazuh-Server

```

**Choix de l'AMI (système d'exploitation) :**

-   Application and OS Images : Quick Start
-   Sélectionner : **Ubuntu**
-   Version : Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
-   Architecture : 64-bit (x86)

**Type d'instance :**

```
Instance type: t3.large
Specifications: 2 vCPUs, 8 GiB RAM

```

> **Justification :** Wazuh nécessite des ressources suffisantes pour l'indexation et l'analyse des événements. Le type t3.large assure des performances optimales.

**Paire de clés (Key Pair) :**

-   Cliquer sur "Create new key pair"
-   Nom : `wazuh-key`
-   Type : RSA
-   Format : .pem (pour Windows)
-   Cliquer sur "Create key pair"
-   Le fichier se télécharge automatiquement
-   **Important :** Conserver ce fichier en lieu sûr

**Configuration réseau :**

-   Network settings → Edit
-   VPC : Default VPC
-   Subnet : No preference
-   Auto-assign public IP : **Enable**
-   Firewall : Create security group

#### 3.1.3 Security Group : Wazuh-Server-SG


| Service           | Port | Protocole | Source                           |
|-------------------|------|-----------|----------------------------------|
| SSH               | 22   | TCP       | My IP                            |
| HTTPS Dashboard   | 443  | TCP       | My IP                            |
| Wazuh Logs        | 1514 | TCP       | Linux-Client-SG, Windows-Client-SG |
| Enrollment        | 1515 | TCP       | Linux-Client-SG, Windows-Client-SG |


**Configuration des règles :**

**Règle 1 - SSH :**

-   Type : SSH
-   Port : 22
-   Source : My IP
-   **Justification :** Accès administrateur sécurisé

**Règle 2 - HTTPS Dashboard :**

-   Type : HTTPS
-   Port : 443
-   Source : My IP
-   **Justification :** Accès à l'interface web Wazuh

**Règle 3 - Communication agents (logs) :**

-   Type : Custom TCP
-   Port : 1514
-   Source : Anywhere-IPv4 (0.0.0.0/0) _(à restreindre plus tard)_
-   **Justification :** Réception des logs des agents

**Règle 4 - Enrôlement agents :**

-   Type : Custom TCP
-   Port : 1515
-   Source : Anywhere-IPv4 (0.0.0.0/0) _(à restreindre plus tard)_
-   **Justification :** Inscription initiale des agents

#### 3.1.4 Configuration du stockage

-   Volume 1 (Root) : 30 GiB
-   Type : gp3 (SSD haute performance)
-   **Justification :** Espace suffisant pour les logs et l'indexation

#### 3.1.5 Lancement

1.  Vérifier le récapitulatif dans le panneau de droite
2.  Cliquer sur "Launch instance"
3.  Cliquer sur "View all instances"
4.  Attendre :
    -   Instance state : Running (2-3 minutes)
    -   Status checks : 2/2 checks passed (3-5 minutes)

#### 3.1.6 Noter les informations

Une fois l'instance lancée :

-   Sélectionner l'instance "Wazuh-Server"
-   Dans l'onglet "Details" en bas, noter :
    -   **Private IPv4 address :** exemple 172.31.29.180
    -   **Public IPv4 address :** exemple 54.123.45.67

![Security Group du serveur Wazuh](https://claude.ai/chat/WAZUHSG.png)

_Figure 2 : Security Group du serveur Wazuh_

![Instance Wazuh Server dans AWS EC2](https://claude.ai/chat/1.png)

_Figure 3 : Instance Wazuh Server dans AWS EC2_

### 3.2 Instance 2 : Client Linux

#### 3.2.1 Configuration EC2

-   Name : Linux-Client
-   AMI : Ubuntu Server 22.04 LTS (64-bit x86)
-   Instance Type : t2.micro (1 vCPU, 1 GiB RAM)
-   Key Pair : wazuh-key (utiliser la clé existante)
-   Public IP : Enable
-   Stockage : 8 GiB gp3

#### 3.2.2 Security Group : Linux-Client-SG


| Service | Port | Protocole | Source |
|--------|------|-----------|--------|
| SSH    | 22   | TCP       | My IP  |


> **Note :** Seul le port SSH est nécessaire pour l'administration. L'agent Wazuh initie les connexions sortantes vers le serveur.

![Security Group du client Linux](https://claude.ai/chat/WINDOWSSG.png)

_Figure 4 : Security Group du client Linux_

![Instance Linux Client](https://claude.ai/chat/3.png)

_Figure 5 : Instance Linux Client_

### 3.3 Instance 3 : Client Windows

#### 3.3.1 Configuration EC2

-   Name : Windows-Client
-   AMI : Microsoft Windows Server 2022 Base
-   Instance Type : t2.medium (2 vCPUs, 4 GiB RAM)
-   Key Pair : wazuh-key
-   Public IP : Enable
-   Stockage : 30 GiB gp3

> **Note :** Windows nécessite plus de ressources que Linux, d'où le choix du t2.medium.

#### 3.3.2 Security Group : Windows-Client-SG


| Service | Port | Protocole | Source |
|--------|------|-----------|--------|
| RDP    | 3389 | TCP       | My IP  |

![Security Group du client Windows](https://claude.ai/chat/LINUXSG.png)

_Figure 6 : Security Group du client Windows_

![Instance Windows Client](https://claude.ai/chat/2.png)

_Figure 7 : Instance Windows Client_

#### 3.3.3 Récupération du mot de passe Windows

Le mot de passe administrateur Windows est chiffré et doit être déchiffré avec la clé privée.

**Procédure :**

1.  Sélectionner l'instance "Windows-Client" dans la console EC2
2.  Cliquer sur "Connect" (bouton en haut)
3.  Aller à l'onglet "RDP client"
4.  Cliquer sur "Get password"
5.  Cliquer sur "Browse" et sélectionner le fichier `wazuh-key.pem`
6.  Cliquer sur "Decrypt Password"
7.  Le mot de passe s'affiche → le copier et le noter
8.  Username : **Administrator**

![Récupération du mot de passe Windows via la console AWS](https://claude.ai/chat/PASSWORD.png)

_Figure 8 : Récupération du mot de passe Windows via la console AWS_

### 3.4 Ajustement des Security Groups

> **Important :** Après la création de toutes les instances, il faut restreindre l'accès aux ports 1514 et 1515 du serveur Wazuh.

#### 3.4.1 Modifier Wazuh-Server-SG

1.  EC2 → Security Groups
2.  Sélectionner "Wazuh-Server-SG"
3.  Onglet "Inbound rules" → "Edit inbound rules"
4.  Pour la règle port 1514 :
    -   Supprimer la source "0.0.0.0/0"
    -   Ajouter "Linux-Client-SG"
    -   Ajouter "Windows-Client-SG"
5.  Répéter pour le port 1515
6.  Cliquer sur "Save rules"

> **Justification :** Cette restriction améliore la sécurité en limitant l'accès aux seuls endpoints autorisés.

----------

## 4. Installation du serveur Wazuh

### 4.1 Connexion SSH au serveur

#### 4.1.1 Depuis Windows (CMD / PowerShell)

Sous Windows, la connexion au serveur Wazuh peut être effectuée directement via le terminal `CMD` ou `PowerShell`, grâce au client SSH intégré aux versions récentes de Windows (Windows 10 et 11).

1.  Ouvrir le terminal Windows (`CMD` ou `PowerShell`)
    
2.  Se placer dans le répertoire contenant la clé privée :
    
    ```bash
    cd Downloads
    
    ```
    
3.  Établir la connexion SSH vers le serveur Wazuh :
    
    ```bash
    ssh -i wazuh-key.pem ubuntu@[IP_PUBLIQUE_WAZUH_SERVER]
    
    ```
    
4.  Lors de la première connexion, une alerte de sécurité s'affiche afin de vérifier l'empreinte du serveur. Taper `yes` pour confirmer :
    
    ```bash
    Are you sure you want to continue connecting (yes/no)?
    
    ```
    
5.  Une fois authentifié, l'accès au serveur Wazuh est établi avec succès.
    

### 4.2 Mise à jour du système

> **Raison :** Assurer que tous les paquets système sont à jour avant l'installation.

```bash
# Mise à jour de la liste des paquets
sudo apt update

# Installation des mises à jour
sudo apt -y upgrade

```

> **Durée :** 3-5 minutes selon les mises à jour disponibles.

### 4.3 Téléchargement du script d'installation

Wazuh fournit un script d'installation automatisé "All-in-One" qui installe tous les composants nécessaires.

```bash
# Télécharger le script officiel Wazuh
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh

# Vérifier le téléchargement
ls -lh wazuh-install.sh

```

### 4.4 Installation Wazuh All-in-One

> **Attention :** Cette étape prend 10-15 minutes. Ne pas fermer la fenêtre !

```bash
# Exécuter l'installation All-in-One
sudo bash wazuh-install.sh -a -i

```

#### 4.4.1 Processus d'installation

Le script installe et configure automatiquement :

1.  **Wazuh Indexer :** Base de données Elasticsearch pour le stockage
2.  **Wazuh Manager :** Moteur d'analyse et de corrélation
3.  **Wazuh Dashboard :** Interface web de visualisation

#### 4.4.2 Informations de connexion

À la fin de l'installation, le script affiche :

```
INFO: You can access the web interface https://172.31.29.180
User: admin
Password: ChBidg89A.Qwr8LhZw+2t?vJyhoJPHG

```

> **TRÈS IMPORTANT :**
> 
> -   Noter le mot de passe affiché (unique pour chaque installation)
> -   Le copier dans le fichier notes
> -   Il sera nécessaire pour accéder au dashboard

![Installation du serveur Wazuh en cours](https://claude.ai/chat/6.png)

_Figure 9 : Installation du serveur Wazuh en cours_

### 4.5 Vérification des services

Après l'installation, vérifier que tous les services fonctionnent correctement.

```bash
# Vérifier Wazuh Manager
sudo systemctl status wazuh-manager
# Résultat attendu: Active: active (running) en VERT
# Appuyer sur 'q' pour quitter

# Vérifier Wazuh Indexer
sudo systemctl status wazuh-indexer
# Résultat attendu: Active: active (running)

# Vérifier Wazuh Dashboard
sudo systemctl status wazuh-dashboard
# Résultat attendu: Active: active (running)

```

### 4.6 Accès au Dashboard Wazuh

#### 4.6.1 Connexion à l'interface web

1.  Ouvrir un navigateur web (Chrome, Firefox, Edge, Safari)
2.  Taper l'adresse : `https://[IP_PUBLIQUE_WAZUH_SERVER]`
3.  **Important :** Utiliser `https://` et non `http://`

#### 4.6.2 Accepter le certificat auto-signé

Le navigateur affichera un avertissement de sécurité car le certificat SSL est auto-signé.

**Sur Chrome :**

-   Cliquer sur "Advanced" (Paramètres avancés)
-   Cliquer sur "Proceed to [IP] (unsafe)"

**Sur Firefox :**

-   Cliquer sur "Advanced"
-   Cliquer sur "Accept the Risk and Continue"

> **Justification :** C'est normal avec un certificat auto-signé en environnement de test.

#### 4.6.3 Page de connexion

-   Username : `admin`
-   Password : [Le mot de passe noté lors de l'installation]
-   Cliquer sur "Log in"

#### 4.6.4 Premier accès

Vous arrivez sur le Dashboard Wazuh :

![Dashboard Wazuh après première connexion](https://claude.ai/chat/7.png)

_Figure 10 : Dashboard Wazuh après première connexion_

----------

## 5. Déploiement des agents

### 5.1 Principe de fonctionnement

Les agents Wazuh sont des logiciels légers installés sur chaque endpoint à superviser. Ils collectent les événements de sécurité et les transmettent au serveur central pour analyse.

**Communications :**

-   Agent → Serveur sur port 1514 (envoi des logs)
-   Serveur ← Agent sur port 1515 (enrôlement initial)
-   Chiffrement et authentification par certificats

### 5.2 Agent Linux

#### 5.2.1 Préparation depuis le Dashboard

1.  Dans le Dashboard Wazuh, cliquer sur le menu (☰)
2.  Server management → Endpoints summary
3.  Cliquer sur "Deploy new agent" (bouton bleu en haut à droite)
4.  Configuration de l'agent :
    -   **Operating system :** DEB amd64 (pour Ubuntu)
    -   **Server address :** IP PRIVÉE du Wazuh-Server (ex: 172.31.29.180)
    -   **Agent name :** Linux-Client
    -   **Agent group :** default
5.  Copier le bloc de commandes généré

> **Important :** Toujours utiliser l'IP PRIVÉE du serveur (communication interne AWS).

#### 5.2.2 Connexion SSH au client Linux

**Ouvrir une NOUVELLE connexion SSH** (garder celle du serveur ouverte).

**Windows (CMD / PowerShell) :**

Sous Windows, la connexion au client Linux peut être réalisée à l'aide du client SSH intégré dans `CMD` ou `PowerShell`, sans recourir à PuTTY.

1.  Ouvrir `CMD` ou `PowerShell`
    
2.  Se positionner dans le dossier contenant la clé privée :
    
    ```bash
    cd Downloads
    
    ```
    
3.  Établir la connexion SSH vers le client Linux :
    
    ```bash
    ssh -i wazuh-key.pem ubuntu@[IP_PUBLIQUE_LINUX_CLIENT]
    
    ```
    
4.  Confirmer l'empreinte du serveur lors de la première connexion en tapant `yes`.
    

#### 5.2.3 Installation de l'agent

```bash
# Télécharger le package de l'agent
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb

# Installer avec configuration
sudo WAZUH_MANAGER='[IP_PRIVEE_WAZUH_SERVER]' \
     WAZUH_AGENT_NAME='Linux-Client' \
     dpkg -i ./wazuh-agent_4.7.5-1_amd64.deb

# Recharger systemd
sudo systemctl daemon-reload

# Activer au démarrage
sudo systemctl enable wazuh-agent

# Démarrer l'agent
sudo systemctl start wazuh-agent

```

> **Durée :** 30-60 secondes pour l'installation complète.

#### 5.2.4 Vérification

```bash
# Vérifier le statut de l'agent
sudo systemctl status wazuh-agent
# Résultat attendu: Active: active (running)
# Appuyer sur 'q' pour quitter

```

#### 5.2.5 Validation dans le Dashboard

1.  Retourner au Dashboard Wazuh (navigateur)
2.  Fermer la fenêtre "Deploy new agent" si ouverte
3.  Rafraîchir la page
4.  Vous devriez voir : **1 agent** dans "Active agents"
5.  Cliquer sur "Linux-Client" pour voir les détails :
    -   Status : Active
    -   OS : Ubuntu 22.04
    -   Version : Wazuh 4.7.5
    -   IP address
    -   Last keep alive

![Agent Linux actif et connecté](https://claude.ai/chat/10.png)

_Figure 11 : Agent Linux actif et connecté_

![Apparition de l'agent Linux dans le Dashboard](https://claude.ai/chat/9.png)

_Figure 12 : Apparition de l'agent Linux dans le Dashboard_

### 5.3 Agent Windows

#### 5.3.1 Connexion RDP au client Windows

**Windows :**

1.  Appuyer sur la touche Windows
2.  Taper "rdp" ou "Connexion Bureau à distance"
3.  Ordinateur : [IP_PUBLIQUE_WINDOWS_CLIENT]
4.  Cliquer sur "Connecter"
5.  Accepter l'avertissement de sécurité
6.  Credentials :
    -   Username : `Administrator`
    -   Password : [mot de passe récupéré section 3.3.3]

![Configuration de la connexion Bureau à distance](https://claude.ai/chat/Windowsetape1.png)

_Figure 13 : Configuration de la connexion Bureau à distance_

![Avertissement de sécurité lors de la connexion](https://claude.ai/chat/Windowsetape2.png)

_Figure 14 : Avertissement de sécurité lors de la connexion_

![Bureau Windows Server 2022 accessible](https://claude.ai/chat/Windowsetape3.png)

_Figure 15 : Bureau Windows Server 2022 accessible_

#### 5.3.2 Préparation depuis le Dashboard

1.  Dans le Dashboard Wazuh (sur votre ordinateur local)
2.  Server management → Endpoints summary
3.  "Deploy new agent"
4.  Configuration :
    -   **Operating system :** Windows
    -   **Architecture :** MSI 64-bit
    -   **Server address :** IP PRIVÉE du Wazuh-Server
    -   **Agent name :** Windows-Client
    -   **Agent group :** default
5.  Copier la commande PowerShell générée

Exemple de commande :

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi `
  -OutFile ${env:tmp}\wazuh-agent.msi

msiexec.exe /i ${env:tmp}\wazuh-agent.msi /q `
  WAZUH_MANAGER='172.31.29.180' `
  WAZUH_AGENT_NAME='Windows-Client' `
  WAZUH_REGISTRATION_SERVER='172.31.29.180'

```

#### 5.3.3 Installation de l'agent

1.  Dans la session RDP Windows, ouvrir PowerShell en Administrateur :
    
    -   Cliquer sur Start (bouton Windows)
    -   Taper : `PowerShell`
    -   Clic droit sur "Windows PowerShell"
    -   "Run as administrator"
    -   Accepter l'UAC (User Account Control)
2.  Coller la commande copiée (clic droit dans PowerShell)
    
3.  Appuyer sur Entrée
    
4.  Attendre la fin de l'installation (1-2 minutes)
    
5.  L'installation est silencieuse (pas de fenêtre)
    

![Installation de l'agent Wazuh via PowerShell](https://claude.ai/chat/Windowsetape4.png)

_Figure 16 : Installation de l'agent Wazuh via PowerShell_

#### 5.3.4 Vérification du service

```powershell
# Vérifier le statut du service
Get-Service WazuhSvc

# Résultat attendu:
# Status   : Running
# Name     : WazuhSvc

```

#### 5.3.5 Validation dans le Dashboard

1.  Retourner au Dashboard Wazuh
2.  Attendre 1-2 minutes et rafraîchir (F5)
3.  Vous devriez maintenant voir : **2 agents actifs**
    -   Linux-Client
    -   Windows-Client
4.  Cliquer sur "Windows-Client" pour les détails

![Agent Windows actif dans le Dashboard](https://claude.ai/chat/11.png)

_Figure 17 : Agent Windows actif dans le Dashboard_

----------

## 6. Installation et configuration de Sysmon (Windows)

### 6.1 Présentation de Sysmon

#### 6.1.1 Qu'est-ce que Sysmon ?

**Sysmon (System Monitor)** est un service système et pilote de périphérique de la suite Microsoft Sysinternals qui enrichit considérablement les capacités de logging de Windows.

#### 6.1.2 Pourquoi utiliser Sysmon ?

Windows génère des événements de sécurité basiques, mais Sysmon ajoute :

-   **Création de processus** avec ligne de commande complète et hashes
-   **Connexions réseau** détaillées (IP source/destination, ports)
-   **Création et modification de fichiers**
-   **Chargement de DLL et drivers**
-   **Modifications du registre**
-   **Accès mémoire entre processus**

#### 6.1.3 Intérêt pour la détection

Sysmon est essentiel pour :

-   Détecter les malwares et ransomwares
-   Identifier les exécutions suspectes (PowerShell encodé, scripts malveillants)
-   Tracer les connexions réseau anormales
-   Faire du threat hunting avancé
-   Reconstituer la chaîne d'attaque (kill chain)

### 6.2 Téléchargement de Sysmon

#### 6.2.1 Procédure

Dans la session RDP Windows :

1.  Ouvrir un navigateur (Microsoft Edge ou Internet Explorer)
2.  Si Internet Explorer demande de configurer la sécurité, accepter
3.  Accéder à : https://learn.microsoft.com/sysinternals/downloads/sysmon
4.  Cliquer sur "Download Sysmon" (lien bleu)
5.  Le fichier `Sysmon.zip` se télécharge dans Downloads

### 6.3 Installation de Sysmon

#### 6.3.1 Extraction du fichier

1.  Ouvrir l'Explorateur de fichiers
2.  Naviguer vers `C:\Users\Administrator\Downloads`
3.  Clic droit sur `Sysmon.zip`
4.  Sélectionner "Extract All..."
5.  Cliquer sur "Extract"
6.  Un dossier `Sysmon` est créé

#### 6.3.2 Installation via PowerShell (Administrateur)

> **Méthode recommandée :** Installation en ligne de commande avec acceptation automatique de la licence.

1.  Ouvrir PowerShell en Administrateur
2.  Naviguer vers le dossier Sysmon :

```powershell
cd C:\Users\Administrator\Downloads\Sysmon

# Installer Sysmon avec configuration par défaut
.\Sysmon64.exe -accepteula -i

# Résultat attendu:
# "Sysmon64 started."
# "SysmonDrv started."

```

![Installation de Sysmon via PowerShell](https://claude.ai/chat/Sysmonetape1.png)

_Figure 18 : Installation de Sysmon via PowerShell_

### 6.4 Vérification du service Sysmon

#### 6.4.1 Vérification via PowerShell

```powershell
# Vérifier que le service est actif
Get-Service Sysmon64

# Résultat attendu:
# Status      : Running
# Name        : Sysmon64
# DisplayName : Sysmon64
# StartType   : Automatic

```

![Service Sysmon64 actif et fonctionnel](https://claude.ai/chat/Sysmonetape2.png)

_Figure 19 : Service Sysmon64 actif et fonctionnel_

#### 6.4.2 Vérification des événements dans l'Observateur

1.  Ouvrir l'Observateur d'événements Windows :
    -   Start → Taper "Event Viewer"
    -   Ou : `eventvwr.msc`
2.  Naviguer vers :
    -   Applications and Services Logs
    -   Microsoft
    -   Windows
    -   Sysmon
    -   Operational
3.  Vous devriez voir des événements Sysmon (Event ID 1, 3, 11, etc.)

### 6.5 Configuration de Wazuh pour collecter Sysmon

Pour que Wazuh collecte et analyse les événements Sysmon, il faut modifier la configuration de l'agent.

#### 6.5.1 Modification du fichier ossec.conf

1.  Ouvrir PowerShell en Administrateur
2.  Éditer le fichier de configuration :

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"

```

#### 6.5.2 Ajout de la configuration Sysmon

Dans Notepad :

1.  Chercher la balise fermante `</ossec_config>` (vers la fin du fichier)
2.  Juste AVANT cette balise, ajouter les lignes suivantes :

```xml
<!-- Configuration pour collecter Sysmon -->
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>

```

**Explication :**

-   `<location>` : Canal d'événements Windows à surveiller
-   `<log_format>` : Format "eventchannel" pour les événements Windows

3.  Sauvegarder : File → Save (ou Ctrl+S)
4.  Fermer Notepad

#### 6.5.3 Redémarrage de l'agent Wazuh

Pour appliquer les modifications :

```powershell
# Arrêter le service
NET STOP WazuhSvc

# Démarrer le service
NET START WazuhSvc

# Vérifier qu'il est bien running
Get-Service WazuhSvc

```

### 6.6 Vérification des événements Sysmon dans Wazuh

#### 6.6.1 Génération d'événements de test

Pour vérifier que Sysmon et Wazuh fonctionnent ensemble, créer quelques processus :

```powershell
# Ouvrir le Bloc-notes
notepad.exe
# Attendre 5 secondes, puis fermer

# Ouvrir la Calculatrice
calc.exe
# Fermer après quelques secondes

# Exécuter une commande
cmd.exe /c "whoami && ipconfig"

```

#### 6.6.2 Vérification dans le Dashboard

1.  Retourner au Dashboard Wazuh (navigateur)
2.  Menu → Threat Hunting → Events
3.  Dans la barre de recherche, taper :

```
agent.name:"Windows-Client" AND data.win.system.eventID:1

```

> **Event ID 1 = Process Creation (création de processus)**

#### 6.6.3 Informations visibles

Vous devriez voir des événements contenant :

-   **Image :** Chemin complet de l'exécutable (`C:\Windows\System32\notepad.exe`)
-   **CommandLine :** Ligne de commande complète
-   **User :** Utilisateur ayant lancé le processus
-   **Hashes :** MD5, SHA256 du fichier
-   **ParentImage :** Processus parent (qui a lancé ce processus)
-   **ParentCommandLine :** Ligne de commande du parent

#### 6.6.4 Autres Event IDs Sysmon utiles
| Event ID | Description              |
|---------:|--------------------------|
| 1        | Process Creation         |
| 3        | Network Connection       |
| 5        | Process Terminated       |
| 7        | Image Loaded (DLL)       |
| 8        | CreateRemoteThread       |
| 10       | Process Access           |
| 11       | File Created             |
| 12/13/14| Registry Events          |
| 22       | DNS Query                |

#### 6.6.5 Requête pour les connexions réseau

```
agent.name:"Windows-Client" AND data.win.system.eventID:3

```

Affiche toutes les connexions réseau établies depuis le Windows Client.

----------

## 7. Tests et génération d'événements

### 7.1 Objectifs des tests

Les tests permettent de :

-   Valider que les agents collectent correctement les événements
-   Vérifier que Wazuh détecte les comportements suspects
-   Comprendre les mécanismes de détection (règles, niveaux d'alerte)
-   Pratiquer l'investigation d'incidents
-   Générer des preuves pour le rapport (captures d'écran)

### 7.2 Scénario 1 : Brute Force SSH (Client Linux)

#### 7.2.1 Objectif

Simuler une attaque par force brute SSH pour tester la détection des tentatives d'authentification échouées multiples.

#### 7.2.2 Description de l'attaque

Une attaque brute force SSH consiste à tenter de se connecter avec différents noms d'utilisateur et mots de passe jusqu'à trouver les bons identifiants. C'est une des attaques les plus courantes contre les serveurs Linux exposés sur Internet.

#### 7.2.3 Procédure de test

**Depuis votre ordinateur local :**

```bash
# Windows (CMD ou PowerShell)
ssh fakeuser@[IP_PUBLIQUE_LINUX_CLIENT]
# Taper n'importe quel mot de passe incorrect
# Répéter cette commande 5 à 10 fois

```

**Alternative depuis le serveur Wazuh :**

```bash
# Connexion SSH au Wazuh-Server
# Puis depuis le serveur:
ssh fakeuser@[IP_PRIVEE_LINUX_CLIENT]
# Répéter 5-10 fois avec différents utilisateurs

```

#### 7.2.4 Détection dans Wazuh

1.  Dashboard → Threat Hunting → Events
2.  Filtre agent :
    
    ```
    agent.name:"Linux-Client"
    
    ```
    
3.  Rechercher les échecs d'authentification :
    
    ```
    agent.name:"Linux-Client" AND authentication_failed
    
    ```
    

#### 7.2.5 Événements détectés

Vous devriez voir des alertes avec :

-   **Rule ID :** 5710 (sshd: Attempt to login using a non-existent user)
-   **Level :** 5 (Low severity)
-   **Description :** "sshd: authentication failed"
-   **User :** fakeuser, admin, test, etc.
-   **Source IP :** Votre adresse IP

Si plusieurs tentatives depuis la même IP :

-   **Rule ID :** 5712 (Multiple authentication failures)
-   **Level :** 10 (High severity) - ALERTE CRITIQUE

![Détection de tentatives SSH échouées - Scénario Brute Force](https://claude.ai/chat/bruteforce.png)

_Figure 20 : Détection de tentatives SSH échouées - Scénario Brute Force_

> **Analyse :** Cette capture montre plusieurs tentatives d'authentification SSH échouées consécutives, caractéristiques d'une attaque par brute force. Wazuh a correctement identifié et alerté sur ce comportement suspect avec un niveau de gravité élevé.

### 7.3 Scénario 2 : Élévation de privilèges (Client Linux)

#### 7.3.1 Objectif

Détecter l'utilisation de la commande `sudo` qui permet d'exécuter des commandes avec des privilèges administrateur (root).

#### 7.3.2 Description

L'élévation de privilèges est une technique utilisée par les attaquants après avoir compromis un compte utilisateur normal pour obtenir des droits administrateur. Surveiller les usages de `sudo` permet de détecter des comportements anormaux ou non autorisés.

#### 7.3.3 Procédure de test

```bash
# Connexion SSH au Linux-Client
ssh -i wazuh-key.pem ubuntu@[IP_PUBLIQUE_LINUX_CLIENT]

# Passer en root avec sudo
sudo su

```

#### 7.3.4 Détection dans Wazuh

```
agent.name:"Linux-Client" AND sudo

```

Ou plus spécifiquement :

```
agent.name:"Linux-Client" AND rule.groups:elevation_privilege

```

#### 7.3.5 Événements détectés

-   **Rule ID :** 5402 (Successful sudo to ROOT executed)
-   **Level :** 3
-   **Command :** La commande exécutée avec sudo
-   **User :** ubuntu

![Détection de l'utilisation de sudo et élévation vers root](https://claude.ai/chat/sudoSU.png)

_Figure 21 : Détection de l'utilisation de sudo et élévation vers root_

> **Analyse :** La capture montre l'exécution de `sudo su` par l'utilisateur ubuntu, permettant de passer en root. Ce type d'événement est crucial à surveiller car il représente une escalade de privilèges qui pourrait être légitime (administration) ou malveillante (attaquant).

### 7.4 Scénario 3 : File Integrity Monitoring (Client Linux)

#### 7.4.1 Objectif

Tester la surveillance de l'intégrité des fichiers (FIM - File Integrity Monitoring) en modifiant un fichier système sensible.

#### 7.4.2 Description

Le FIM surveille les modifications, créations et suppressions de fichiers critiques. C'est essentiel pour détecter :

-   Modifications non autorisées de fichiers système
-   Installation de backdoors
-   Altération de fichiers de configuration
-   Compromission de binaires système

#### 7.4.3 Procédure de test

```bash
# Modifier un fichier sensible (ajout d'un commentaire inoffensif)
echo "test" | sudo tee -a /etc/passwd

# Attendre 1-2 minutes pour le scan FIM

```

#### 7.4.4 Détection dans Wazuh

```
agent.name:"Linux-Client" AND syscheck

```

Ou :

```
agent.name:"Linux-Client" AND rule.groups:syscheck

```

#### 7.4.5 Événements détectés

-   **Rule ID :** 550 (Integrity checksum changed)
-   **Level :** 7 (Important)
-   **File :** /etc/passwd
-   **Changes :** Size, modification time, checksum
-   **Before/After :** Hashes MD5/SHA1 avant et après modification

![Détection de modification du fichier /etc/passwd par File Integrity Monitoring](https://claude.ai/chat/FIMerror.png)

_Figure 22 : Détection de modification du fichier /etc/passwd par File Integrity Monitoring_

> **Analyse :** Le FIM a détecté la modification du fichier sensible /etc/passwd. L'alerte indique les changements de checksum, confirmant que l'intégrité du fichier a été compromise. Dans un contexte réel, cela pourrait indiquer l'ajout d'un compte backdoor.

### 7.5 Scénario 4 : Échecs de connexion RDP (Client Windows)

#### 7.5.1 Objectif

Détecter les tentatives de connexion RDP échouées, similaires à l'attaque brute force SSH mais sur Windows.

#### 7.5.2 Description

RDP (Remote Desktop Protocol) est fréquemment ciblé par les attaquants pour compromettre des serveurs Windows. Les tentatives échouées multiples indiquent une potentielle attaque par brute force.

#### 7.5.3 Procédure de test

1.  Se déconnecter de la session RDP actuelle :
    
    -   Dans Windows Server : Start → Power → Disconnect
    -   Ou simplement fermer la fenêtre RDP
2.  Tenter des connexions avec un mauvais mot de passe :
    
    -   Ouvrir à nouveau la connexion RDP
    -   IP : [IP_PUBLIQUE_WINDOWS_CLIENT]
    -   Username : Administrator
    -   Password : `WrongPassword123!` (ou autre mot de passe incorrect)
    -   Répéter 3 à 5 fois
3.  Se reconnecter avec le BON mot de passe
    

#### 7.5.4 Détection dans Wazuh

```
agent.name:"Windows-Client" AND rule.id:60122

```

Ou chercher l'Event ID Windows :

```
agent.name:"Windows-Client" AND data.win.system.eventID:4625

```

#### 7.5.5 Événements détectés

-   **Windows Event ID :** 4625 (An account failed to log on)
-   **Rule ID Wazuh :** 60122 (Windows: User logon failed)
-   **Level :** 5
-   **Logon Type :** 10 (RemoteInteractive - RDP)
-   **Failure Reason :** Bad password
-   **Source IP :** Votre adresse IP

![Détection de tentatives de connexion RDP échouées sur Windows](https://claude.ai/chat/LoginRDP.png)

_Figure 23 : Détection de tentatives de connexion RDP échouées sur Windows_

> **Analyse :** Les événements Windows Security 4625 capturés par Wazuh montrent plusieurs tentatives de connexion RDP échouées. Le champ "Logon Type: 10" confirme qu'il s'agit de RDP. Les tentatives répétées depuis la même IP sont caractéristiques d'une attaque automatisée.

### 7.6 Récapitulatif des tests


| Scénario           | Système | Technique             | Rule Level |
|--------------------|---------|-----------------------|------------|
| Brute Force SSH    | Linux   | Authentication        | 5–10       |
| Sudo Elevation     | Linux   | Privilege Escalation  | 3          |
| FIM                | Linux   | File Modification     | 7          |
| RDP Failed         | Windows | Authentication        | 5          |


----------

## 8. Analyse comparative : SIEM vs EDR

### 8.1 Définitions

#### 8.1.1 SIEM (Security Information and Event Management)

**Définition :** Système de gestion centralisée des informations et événements de sécurité.

**Fonctions principales :**

-   **Collecte :** Agrégation des logs de multiples sources (serveurs, applications, équipements réseau, firewalls, etc.)
-   **Normalisation :** Uniformisation des formats de logs
-   **Corrélation :** Mise en relation d'événements provenant de différentes sources
-   **Analyse :** Détection d'anomalies et de patterns suspects
-   **Alerting :** Génération d'alertes en temps réel
-   **Conformité :** Respect des exigences réglementaires (GDPR, PCI-DSS, etc.)
-   **Reporting :** Génération de rapports pour audit et investigation

**Vision :** Vue d'ensemble (bird's eye view) de l'ensemble du système d'information.

#### 8.1.2 EDR (Endpoint Detection and Response)

**Définition :** Solution de détection et réponse spécialisée dans la surveillance approfondie des endpoints (postes de travail, serveurs).

**Fonctions principales :**

-   **Monitoring continu :** Surveillance en temps réel des activités des endpoints
-   **Visibilité granulaire :** Détails sur les processus, fichiers, registre, connexions réseau
-   **Détection comportementale :** Identification de comportements anormaux
-   **Threat hunting :** Recherche proactive de menaces
-   **Investigation :** Analyse forensique des incidents
-   **Response :** Isolation, blocage, remediation
-   **IOC tracking :** Suivi des indicateurs de compromission

**Vision :** Vue microscopique détaillée de chaque endpoint.

### 8.2 Différences clés


| Critère        | SIEM                                      | EDR                                   |
|----------------|-------------------------------------------|----------------------------------------|
| **Périmètre**  | Tout le SI (réseau, applications, serveurs) | Endpoints uniquement                   |
| **Profondeur** | Événements de haut niveau                 | Détails granulaires (processus, fichiers) |
| **Sources**    | Multiples (logs variés)                   | Endpoints spécifiquement               |
| **Force**      | Corrélation multi-sources                 | Visibilité endpoint approfondie        |
| **Détection**  | Règles et corrélations                    | Comportemental + signatures            |
| **Réponse**    | Alerting principalement                  | Isolation, blocage, remédiation         |
| **Use Case**   | Conformité, audit, vue globale            | Détection malware, APT, forensics       |


### 8.3 Exemples concrets dans ce projet

#### 8.3.1 Capacités SIEM de Wazuh

**Exemple 1 : Corrélation SSH Brute Force**

-   Collecte des logs SSH de Linux-Client
-   Détection de tentatives individuelles (Rule 5710, Level 5)
-   Corrélation des tentatives multiples (Rule 5712, Level 10)
-   Alerte sur pattern d'attaque brute force

**Exemple 2 : Monitoring multi-endpoints**

-   Vue centralisée des événements Linux ET Windows
-   Dashboard unique pour tous les agents
-   Corrélation possible entre événements Linux et Windows

**Exemple 3 : Audit et conformité**

-   Traçabilité de toutes les connexions
-   Logs des modifications de fichiers sensibles (FIM)
-   Historique des élévations de privilèges
-   Rapports pour audit de sécurité

#### 8.3.2 Capacités EDR de Wazuh

**Exemple 1 : Sysmon sur Windows**

-   Détails sur chaque processus créé (Event ID 1)
-   Ligne de commande complète visible
-   Hashes MD5/SHA256 pour validation
-   Chaîne parent-enfant pour reconstruction d'attaque

**Exemple 2 : File Integrity Monitoring**

-   Surveillance en temps réel de /etc/passwd
-   Détection des modifications non autorisées
-   Calcul de checksums avant/après
-   Alerte immédiate sur compromission

**Exemple 3 : Détection comportementale**

-   Création de compte + ajout groupe admin = suspect
-   Multiples échecs RDP = brute force
-   Processus inhabituel depuis \Temp\ = potentiel malware

### 8.4 Complémentarité SIEM + EDR

#### 8.4.1 Pourquoi combiner les deux ?

1.  **Vision globale + détails :**
    
    -   SIEM : "Il y a une activité suspecte sur le réseau"
    -   EDR : "Voici exactement quel processus, quelle commande, quel fichier"
2.  **Détection multi-couches :**
    
    -   SIEM : Détecte l'anomalie par corrélation
    -   EDR : Détecte le comportement malveillant au niveau endpoint
3.  **Investigation complète :**
    
    -   SIEM : Quel est le scope de l'incident ?
    -   EDR : Comment l'attaque s'est-elle déroulée sur cet endpoint ?

#### 8.4.2 Exemple de scénario combiné

**Scénario :** Attaque ransomware

1.  **EDR détecte :**
    
    -   Processus suspect créé depuis email (Sysmon Event ID 1)
    -   Chiffrement massif de fichiers (FIM)
    -   Connexions réseau vers C2 (Command & Control)
2.  **SIEM corrèle :**
    
    -   Email malveillant reçu (logs Exchange)
    -   Téléchargement depuis URL suspecte (logs proxy)
    -   Spread latéral vers autres machines (logs réseau)
    -   Élévation de privilèges sur serveur de fichiers
3.  **Résultat :**
    
    -   Vue complète de la kill chain
    -   Identification du patient zéro (EDR)
    -   Cartographie de la propagation (SIEM)
    -   Réponse coordonnée sur tous les systèmes

#### 8.4.3 Wazuh : Le meilleur des deux mondes

Wazuh est unique car il combine SIEM et EDR dans une seule plateforme :

-   **Architecture SIEM :** Serveur central, agents distribués, corrélation
-   **Capacités EDR :** Monitoring endpoint, FIM, Sysmon, réponse active
-   **Open Source :** Gratuit, personnalisable, communauté active
-   **Intégrations :** Threat intelligence, VirusTotal, MITRE ATT&CK

----------

## 9. Conclusion

Ce projet a permis de mettre en place une infrastructure de sécurité basée sur Wazuh, déployée dans un environnement cloud AWS. À travers l'installation du serveur Wazuh et l'intégration de plusieurs agents, il a été possible de centraliser les journaux, de surveiller les activités systèmes et de détecter différents types d'incidents de sécurité.

La combinaison des fonctionnalités SIEM et EDR a offert une meilleure visibilité sur les événements de sécurité, aussi bien au niveau du réseau que des machines clientes. Les tests réalisés ont permis de valider la capacité de la plateforme à détecter des comportements suspects tels que les attaques par force brute, les élévations de privilèges et les modifications non autorisées des fichiers.

Au-delà de l'aspect technique, ce projet a renforcé la compréhension des concepts fondamentaux de la cybersécurité, notamment la surveillance continue, l'analyse des logs et l'investigation des incidents. Il a également permis de développer des compétences pratiques en administration système, en cloud computing et en sécurité opérationnelle.

Enfin, ce travail constitue une base solide pour des améliorations futures, telles que l'ajout de règles de détection personnalisées, l'automatisation de la réponse aux incidents et le déploiement à plus grande échelle. Il représente une expérience formatrice et pertinente pour une orientation vers les métiers de la cybersécurité.

----------

## 10. Annexes

### Glossaire

-   **APT** - Advanced Persistent Threat - Menace avancée et persistante
-   **C2** - Command and Control - Serveur de contrôle d'un malware
-   **CVE** - Common Vulnerabilities and Exposures - Base de vulnérabilités
-   **EDR** - Endpoint Detection and Response
-   **FIM** - File Integrity Monitoring - Surveillance d'intégrité de fichiers
-   **IAM** - Identity and Access Management
-   **IOC** - Indicator of Compromise - Indicateur de compromission
-   **MFA** - Multi-Factor Authentication
-   **MITRE ATT&CK** - Framework de tactiques et techniques d'attaquants
-   **PAM** - Privileged Access Management
-   **RDP** - Remote Desktop Protocol
-   **SIEM** - Security Information and Event Management
-   **SOAR** - Security Orchestration, Automation and Response
-   **SOC** - Security Operations Center
-   **SSH** - Secure Shell
-   **Sysmon** - System Monitor (Microsoft Sysinternals)
-   **TTPs** - Tactics, Techniques, and Procedures

----------

**Fin du document**
