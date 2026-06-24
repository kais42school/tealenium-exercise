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

### 3.1 — Vérification de l'existant

Avant toute installation, vérification de la présence d'un service FTP/SFTP sur le système :

```bash
dpkg -l | grep "ftp"
```

Résultat observé :
```
ii  ftp                       20230507-2build3   all    dummy transitional package for tnftp
ii  openssh-sftp-server       1:9.6p1-3ubuntu13.16  amd64  secure shell (SSH) sftp server module
ii  tnftp                     20230507-2build3   amd64  enhanced ftp client
```

> **Que montre ce résultat sur la différence entre `ftp`/`tnftp` et `openssh-sftp-server` ?**  
> [À compléter]

Vérification que le service SSH tourne et que le sous-système SFTP est actif dans sa configuration :

```bash
sudo systemctl status ssh
cat /etc/ssh/sshd_config | grep "ftp"
```

Résultat : le service `ssh` est `active (running)`, et la ligne suivante est déjà présente et non commentée dans `sshd_config` :
```
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

> **Pourquoi n'y a-t-il pas eu besoin d'installer un serveur FTP supplémentaire (ex. vsftpd) ?**  
> [À compléter]

> **Quel est le problème de sécurité du protocole FTP classique, et en quoi SFTP le résout-il ?**  
> [À compléter]

> **Note pour la reproductibilité sur une autre VM :** sur cet environnement, `openssh-sftp-server` était déjà installé par défaut. Ce n'est pas garanti sur toute installation Ubuntu. Si le paquet est absent, l'installer avec :
> ```bash
> sudo apt install openssh-sftp-server
> ```
> et vérifier ensuite que la ligne `Subsystem sftp ...` est bien présente (et non commentée) dans `/etc/ssh/sshd_config`.

---

### 3.2 — Restriction des utilisateurs SFTP par chroot

#### Principe

Par défaut, un utilisateur SFTP n'est pas confiné à son dossier personnel : son dossier "home" est seulement le point où il arrive à la connexion, pas une limite — il peut naviguer ailleurs dans la limite des permissions Unix. Le chroot SSH (`ChrootDirectory`) impose une restriction structurelle : l'utilisateur ne voit et n'accède qu'au dossier désigné, qui devient sa racine apparente, quelles que soient les permissions Unix par ailleurs.

> **Pourquoi est-il important de restreindre les utilisateurs SFTP à leur propre dossier, plutôt que de se fier au seul dossier home par défaut ?**  
> [À compléter]

#### Recherche de la syntaxe

Recherche de la directive permettant de cibler un utilisateur/groupe spécifique dans la configuration SSH :

```bash
man sshd_config | grep "Match"
```

> **Pourquoi consulter le `man` plutôt que de chercher la syntaxe sur le web ?**  
> [À compléter]

Vérification préalable des groupes du compte administrateur, pour décider s'il doit être inclus dans la restriction :

```bash
groups ubuntu
id ubuntu
cat /etc/group
```

> **Pourquoi le compte `ubuntu` ne doit-il pas être inclus dans le groupe de restriction SFTP ?**  
> [À compléter]

#### Création d'un groupe dédié

```bash
sudo groupadd sftponly
```

> **Pourquoi cibler un groupe plutôt qu'un utilisateur individuel (`Match User`) dans la configuration SSH ?**  
> [À compléter]

#### Configuration du bloc de restriction

Ajout en fin de fichier `/etc/ssh/sshd_config` (via `sudo nano /etc/ssh/sshd_config`) :

```
Match Group sftponly
    ChrootDirectory /home/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

> **Que fait chacune de ces quatre directives ?**  
> [À compléter]

Vérification de la syntaxe avant tout redémarrage du service :

```bash
sudo sshd -t
```

Application du changement :

```bash
sudo systemctl restart ssh
sudo systemctl daemon-reload
sudo systemctl restart ssh
sudo systemctl status ssh
```

> **Pourquoi tester la syntaxe avec `sshd -t` avant de redémarrer le service ?**  
> [À compléter]

Vérification sur une seconde session SSH ouverte en parallèle que l'accès normal (compte admin) n'a pas été cassé par le changement.

#### Compte de test

Création d'un compte jetable, sans accès shell, pour valider la configuration sans risquer un compte de production :

```bash
sudo useradd -m -G sftponly -s /usr/sbin/nologin testsftp
```

> **Pourquoi tester sur un compte dédié plutôt que directement sur un compte existant ?**  
> [À compléter]

Génération d'une paire de clés dédiée à ce compte de test :

```bash
ssh-keygen -t ed25519 -C "testsftp_keygen" -f ~/.ssh/id_testsftp
sudo mkdir -p /home/testsftp/.ssh
cat ~/.ssh/id_testsftp.pub
sudo nano /home/testsftp/.ssh/authorized_keys
```

(Collage du contenu de la clé publique affichée par `cat ~/.ssh/id_testsftp.pub` dans `authorized_keys`.)

```bash
sudo chown -R testsftp:testsftp /home/testsftp/.ssh
sudo chmod 700 /home/testsftp/.ssh
sudo chmod 600 /home/testsftp/.ssh/authorized_keys
```

#### Contrainte de permissions sur le dossier chrooté

```bash
ls -ld /home/testsftp
```

Résultat observé : le dossier appartenait à `testsftp:testsftp` — incompatible avec les exigences d'OpenSSH, qui refuse le chroot si le dossier ciblé (et ses parents) n'appartient pas à `root` ou est modifiable par d'autres utilisateurs que `root`.

Correction :

```bash
sudo chown root:root /home/testsftp
sudo chmod 755 /home/testsftp
```

Vérification :

```bash
groups testsftp
ls -ld /home/testsftp
```

> **Pourquoi cette contrainte de propriété/permissions existe-t-elle (le dossier de chroot doit appartenir à `root`) ?**  
> [À compléter]

**Note :** le sous-dossier `.ssh` à l'intérieur de `/home/testsftp` reste, lui, possédé par `testsftp` — la contrainte de propriété `root` ne s'applique qu'au dossier qui sert de racine du chroot, pas à son contenu.

#### Vérification de la connexion

[À compléter — résultat de la connexion `sftp -i ~/.ssh/id_testsftp testsftp@<ip>` : confirmer que l'utilisateur est confiné à son dossier (`cd /` ne doit montrer que son propre contenu) et qu'une tentative de connexion shell classique échoue bien]

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

### 4.4 — Audit de sécurité avec Lynis

Installation de l'outil d'audit (lecture seule, ne modifie aucune config) :

```bash
sudo apt install lynis
sudo lynis audit system
```

**Premier scan — Hardening index : 57/100**

Filtrage des avertissements et suggestions dans le rapport :
```bash
sudo grep "warning\[\]" /var/log/lynis-report.dat
sudo grep "suggestion\[\]" /var/log/lynis-report.dat
```

Sur la cinquantaine de suggestions retournées, la plupart concernent un contexte serveur de production/entreprise (partitionnement dédié, mot de passe GRUB, rotation de mots de passe — sans objet puisque l'authentification par mot de passe est désactivée, accounting/auditd, serveur de logs externe...) et ont été jugées hors scope pour cet exercice.

Suggestions retenues et traitées :

| Suggestion Lynis | Action |
|---|---|
| `PKGS-7392` — paquets vulnérables | `sudo apt update && sudo apt upgrade -y` |
| `PKGS-7346` — paquets obsolètes | `sudo apt autoremove --purge` |
| `SSH-7408` — durcissement SSH | Ajout dans `sshd_config` : `AllowTcpForwarding no`, `X11Forwarding no`, `AllowAgentForwarding no`, `MaxAuthTries 3`, `ClientAliveCountMax 2`, `TCPKeepAlive no`, `LogLevel VERBOSE` |
| `DEB-0880` — fail2ban absent | [À compléter — installation et configuration en cours] |
| `BANN-7126`/`BANN-7130` — bannière légale | [À compléter] |
| `HRDN-7230` — scanner de malware absent | [À compléter] |

> **Pourquoi `Port 22` n'a-t-il pas été changé malgré la suggestion Lynis ?**  
> [À compléter]

> **Pourquoi la plupart des suggestions liées aux mots de passe (`AUTH-92xx`) sont sans objet ici ?**  
> [À compléter]

**Second scan, après application des correctifs ci-dessus — Hardening index : 67/100**

```bash
sudo lynis audit system
```

> **Pourquoi le score augmente-t-il, et qu'est-ce qui explique qu'il ne soit pas à 100 ?**  
> [À compléter]

---

## 5. Vérification finale

> *Section à compléter*

---

*Document rédigé dans le cadre d'un exercice technique d'alternance.*
