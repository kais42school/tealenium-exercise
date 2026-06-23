# Configuration d'un serveur Ubuntu
**Auteur :** Kais  
**Contexte :** Exercice technique — Tealenium  
**Objectif :** Documenter l'ensemble des opérations d'installation et de configuration réalisées sur une VM Ubuntu, afin de permettre la reproduction à l'identique sur un autre environnement.

---

## Table des matières
1. [Présentation de l'environnement](#1-présentation-de-lenvironnement)
2. [Mise à jour de l'OS vers Ubuntu 24.04 LTS](#2-mise-à-jour-de-los-vers-ubuntu-2404-lts)
3. [Installation d'une solution FTP sécurisée](#3-installation-dune-solution-ftp-sécurisée)
4. [Sécurisation du serveur](#4-sécurisation-du-serveur)
5. [Vérification finale](#5-vérification-finale)

---

## 1. Présentation de l'environnement

| Paramètre | Valeur |
|---|---|
| Adresse IP | *non versionnée — fournie par le client, voir accès local* |
| Utilisateur | *non versionné — voir accès local* |
| OS initial | Ubuntu 23.04 (Lunar Lobster) |
| OS cible | Ubuntu 24.04 LTS (Noble Numbat) |
| Accès | SSH (clé ED25519) |

> **Note de sécurité :** les identifiants de connexion (IP, utilisateur, mot de passe) ne doivent jamais apparaître dans un dépôt Git, même privé. S'ils ont besoin d'être documentés pour reproduire l'environnement, ils doivent vivre dans un fichier `.env` ou équivalent, explicitement listé dans `.gitignore`.

### Vérification de la version initiale

```bash
lsb_release -a
```

Résultat observé :
```
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 23.04
Release:        23.04
Codename:       lunar
```

---

## 2. Mise à jour de l'OS vers Ubuntu 24.04 LTS

### Contexte et problématique

Ubuntu 23.04 (Lunar) et Ubuntu 23.10 (Mantic) sont deux versions intermédiaires (*non-LTS*) dont le support est arrivé à expiration (*End of Life*). Leurs dépôts ne sont plus disponibles sur les miroirs officiels (`archive.ubuntu.com`), ce qui rend toute opération `apt` impossible sans intervention préalable.

La migration vers Ubuntu 24.04 LTS a donc nécessité **deux montées de version successives** :
```
Ubuntu 23.04 (Lunar) ──► Ubuntu 23.10 (Mantic) ──► Ubuntu 24.04 LTS (Noble)
```
À chaque étape, les dépôts ont dû être redirigés vers les archives historiques d'Ubuntu (`old-releases.ubuntu.com`).

---

### Étape 1 — Redirection des dépôts vers les archives (23.04 EOL)

```bash
# Sauvegarde du fichier sources.list original
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup

# Remplacement des dépôts actifs par les archives historiques
sudo sed -i 's/archive.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list

# Vérification que les dépôts sont désormais accessibles
sudo apt update
```

> **Pourquoi cette étape ?**  
> [À compléter — expliquer ce que tu as compris sur les dépôts EOL]

---

### Étape 2 — Première migration : 23.04 → 23.10 via le script officiel

```bash
# Création d'un répertoire de travail
mkdir ~/upgrader
cd ~/upgrader

# Téléchargement du script de migration depuis les archives Ubuntu
wget http://old-releases.ubuntu.com/ubuntu/dists/mantic-updates/main/dist-upgrader-all/current/mantic.tar.gz

# Extraction de l'archive
tar -xaf mantic.tar.gz

# Lancement de la migration vers 23.10
sudo ./mantic
```

> **Pourquoi utiliser ce script plutôt que `do-release-upgrade` directement ?**  
> [À compléter — expliquer ton choix]

---

### Étape 3 — Mise à jour complète de tous les paquets

```bash
sudo apt update && sudo apt dist-upgrade -y
```

> **Pourquoi `dist-upgrade` et non `upgrade` ?**  
> [À compléter — expliquer la différence]

---

### Étape 4 — Deuxième migration : 23.10 → 24.04 LTS

```bash
sudo do-release-upgrade
```

---

### Vérification post-migration

```bash
lsb_release -a
```

Résultat obtenu :
```
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04 LTS
Release:        24.04
Codename:       noble
```

---

## 3. Installation d'une solution FTP sécurisée

> *Section à compléter*

---

## 4. Sécurisation du serveur

### Contexte et problématique

Dès la prise en main de la VM, les logs SSH révèlent des centaines de tentatives de connexion automatisées depuis des adresses IP étrangères. Il s'agit d'attaques par **brute force** : des robots testent en continu des combinaisons de mots de passe sur le port 22 exposé publiquement.

```bash
sudo systemctl status ssh
```

> **Que montre cette commande et qu'est-ce que ça t'a appris sur l'état de la VM ?**  
> [À compléter]

---

### 4.1 — Authentification par clé SSH

#### Principe

> **Pourquoi remplacer l'authentification par mot de passe par une clé SSH ?**  
> [À compléter — expliquer la différence entre les deux méthodes et pourquoi c'est plus sûr]

> **Pourquoi avoir choisi l'algorithme ED25519 plutôt que RSA ?**  
> [À compléter]

> **Pourquoi génère-t-on la clé sur sa machine locale et non sur la VM ?**  
> [À compléter]

#### Génération de la paire de clés (sur la machine locale)

```bash
ssh-keygen -t ed25519 -C "tealenium_vm_keygen" -f ~/.ssh/id_tealenium
```

> **Pourquoi utiliser l'option `-f` avec un nom spécifique ?**  
> [À compléter — expliquer le risque d'écraser une clé existante]

#### Ajout de la clé publique sur la VM

```bash
echo "ssh-ed25519 <clé_publique> tealenium_vm_keygen" >> ~/.ssh/authorized_keys
```

> **Pourquoi utiliser `>>` et non `>` ?**  
> [À compléter]

#### Vérification

```bash
cat ~/.ssh/authorized_keys
```

---

### 4.2 — Désactivation de l'authentification par mot de passe

Une fois la connexion par clé validée, l'authentification par mot de passe est désactivée pour rendre les attaques brute force inopérantes.

```bash
# Vérification de la configuration actuelle
cat /etc/ssh/sshd_config | grep "Authentication"
```

Paramètre ciblé et modifié :
```bash
sudo nano /etc/ssh/sshd_config
```

```
PasswordAuthentication no
```

```bash
# Redémarrage du service SSH pour appliquer les changements
sudo systemctl restart ssh
```

> **Pourquoi est-il impératif de tester la connexion par clé AVANT de désactiver les mots de passe ?**  
> [À compléter]

---

### 4.3 — Pare-feu UFW

#### État initial

```bash
sudo ufw status
```

Résultat observé :
```
Status: inactive
```

> **Pourquoi UFW était-il inactif et quel risque cela représentait-il ?**  
> [À compléter]

#### Configuration et activation

```bash
# Autorisation du port SSH avec limitation du brute force (max 6 tentatives / 30s)
sudo ufw limit 22/tcp

# Activation du pare-feu
sudo ufw enable
```

> **Pourquoi appliquer `limit` plutôt que `allow` sur le port 22 ?**  
> [À compléter]

> **Pourquoi est-il impératif d'autoriser le port 22 AVANT d'activer UFW ?**  
> [À compléter]

#### Vérification

```bash
sudo ufw status
```

Résultat obtenu :
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT       Anywhere
22/tcp (v6)                LIMIT       Anywhere (v6)
```

---

## 5. Vérification finale

> *Section à compléter*

---

*Document rédigé dans le cadre d'un exercice technique d'alternance.*
