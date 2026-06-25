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

Test depuis la VM elle-même (connexion à `localhost`) :

```bash
sftp -i ~/.ssh/id_testsftp testsftp@localhost
```

```
sftp> pwd
Remote working directory: /
sftp> ls -la
drwxr-xr-x    ? root     root         4096 Jun 24 13:22 .
drwxr-xr-x    ? root     root         4096 Jun 24 13:22 ..
-rw-r--r--    ? 1001     1002          220 Jan  7  2023 .bash_logout
-rw-r--r--    ? 1001     1002          343 Jul 18  2023 .bashrc
-rw-r--r--    ? 1001     1002          807 Jan  7  2023 .profile
drwx------    ? 1001     1002         4096 Jun 24 13:23 .ssh
sftp> cd ..
sftp> ls
```

`pwd` affiche `/` — `testsftp` perçoit `/home/testsftp` comme sa racine. `cd ..` ne fait remonter à rien de visible : le chroot empêche toute sortie. Les fichiers de l'utilisateur s'affichent avec des UID/GID numériques (`1001`/`1002`) plutôt que des noms, preuve supplémentaire que `/etc/passwd` et `/etc/group` (réels, sur la VM) ne sont pas accessibles depuis l'intérieur du chroot.

Test du blocage shell :
```bash
ssh -i ~/.ssh/id_testsftp testsftp@localhost
```
```
This service allows sftp connections only.
Connection to localhost closed.
```

✅ Confirmé : `testsftp` est confiné à son dossier et ne peut obtenir aucun shell — seul le SFTP est autorisé.

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

### 4.4 — Restriction de la connexion root et audit complémentaire

#### `PermitRootLogin`

Vérification de la valeur par défaut (commentée, donc valeur OpenSSH appliquée implicitement) :

```bash
cat /etc/ssh/sshd_config | grep "PermitRoot"
```
```
#PermitRootLogin prohibit-password
```

`prohibit-password` interdit déjà une connexion root par mot de passe (redondant avec `PasswordAuthentication no`, déjà global), mais autoriserait en théorie une connexion root par clé si une clé était un jour ajoutée à `/root/.ssh/authorized_keys`. Pour un hardening explicite plutôt qu'implicite, la directive est forcée :

```bash
sudo nano /etc/ssh/sshd_config
```
```
PermitRootLogin no
```

```bash
sudo sshd -t
sudo systemctl restart ssh
```

> **Pourquoi interdire totalement la connexion root, même par clé ?**  
> [À compléter]

Vérification sur une seconde session que l'accès `ubuntu` + `sudo` reste fonctionnel après le changement.

#### Audit des services actifs au démarrage

```bash
sudo systemctl list-unit-files --type=service | grep "enabled"
```

Point notable : `unattended-upgrades.service` était déjà `enabled` — les mises à jour de sécurité sont appliquées automatiquement sans action manuelle.

> **Que faut-il vérifier dans une liste de services actifs au démarrage, dans une optique de hardening ?**  
> [À compléter]

#### Audit des ports ouverts

```bash
ss -tlnp
```
```
LISTEN   0.0.0.0:22
LISTEN   127.0.0.54:53
LISTEN   127.0.0.53%lo:53
LISTEN   [::]:22
```

Un seul port réellement exposé au réseau externe : **22** (SSH, déjà protégé par clé + UFW + fail2ban). Le port 53 (DNS, `systemd-resolved`) est bindé sur `127.x.x.x` — accessible uniquement en local, sans risque externe.

> **Pourquoi est-il important de distinguer un port lié à `0.0.0.0`/`[::]` d'un port lié à `127.x.x.x` ?**  
> [À compléter]

---

### 4.5 — Audit de sécurité avec Lynis

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

### 4.6 — Fail2ban

#### Pourquoi fail2ban en complément d'UFW

UFW (`limit 22/tcp`) ralentit le brute force (max 6 tentatives / 30s) mais ne bannit jamais l'IP — elle peut continuer à réessayer indéfiniment. Fail2ban surveille les logs d'authentification (`/var/log/auth.log`), détecte les échecs répétés depuis une même IP, et **bannit** automatiquement cette IP via une règle firewall pendant une durée définie. Les deux outils sont complémentaires : UFW limite le débit, fail2ban punit la répétition.

#### Installation et configuration

```bash
sudo apt install fail2ban
```

> **Pourquoi copier `jail.conf` en `jail.local` plutôt que d'éditer `jail.conf` directement ?**  
> `jail.conf` est le fichier de configuration par défaut fourni par le paquet — il est **écrasé à chaque mise à jour** de fail2ban. `jail.local` est un fichier séparé, qui n'est jamais touché par les mises à jour, et dont les valeurs **prennent la priorité** sur celles de `jail.conf`. Éditer directement `jail.conf` ferait perdre toute la configuration personnalisée au prochain `apt upgrade`.

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Section `[sshd]` configurée ainsi :
```
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 1h
findtime = 10m
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

> **Notes sur cette configuration :**
> - `port = ssh` : alias résolu via `/etc/services`, équivalent fonctionnel à `22`, plus lisible.
> - `logpath`/`backend` : laissés tels quels — ce sont des **variables** que fail2ban résout automatiquement selon l'OS détecté (ici, le bon chemin de log SSH d'Ubuntu). Les modifier en dur casserait la portabilité du fichier sur un autre système.

```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

> **Que fait `systemctl` ?**  
> `systemctl` est l'outil de gestion des unités systemd : services, sockets, timers... La plupart des daemons (processus tournant en arrière-plan) d'Ubuntu moderne sont gérés ainsi — `ssh`, `ufw`, `fail2ban`, `cron`, `rsyslog`, etc. C'est ce qui permet de démarrer/arrêter/redémarrer un service et de consulter son état.

Vérification détaillée des logs au démarrage :
```bash
sudo journalctl -u fail2ban -n 30 --no-pager
```

Un warning est apparu : `WARNING 'allowipv6' not defined in 'Definition'. Using default one: 'auto'`. Purement informatif — `allowipv6` est un paramètre optionnel non défini explicitement, fail2ban applique sa valeur par défaut sans que cela pose de problème.

#### Vérification du fonctionnement

```bash
sudo fail2ban-client status sshd
```

Résultat obtenu :
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     4
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   91.92.40.176
```

✅ La jail `sshd` est active : 4 tentatives échouées détectées et 1 IP automatiquement bannie après dépassement du seuil (`maxretry = 3`) — validé par le brute force réel déjà observé sur la VM, sans simulation nécessaire.

---

### 4.7 — Bannière légale

#### Principe

Une bannière n'empêche techniquement rien, mais avertit explicitement tout visiteur qu'il n'a pas le droit d'accéder au système sans autorisation — utile comme preuve dans une éventuelle démarche légale en cas d'intrusion.

- `/etc/issue` → affiché pour une connexion **locale** (console directe)
- `/etc/issue.net` → affiché pour une connexion **SSH distante** — le plus pertinent ici, vu que la VM est administrée à distance

```bash
sudo nano /etc/issue
sudo nano /etc/issue.net
```

Activation de l'affichage côté SSH (`sshd` lit ce fichier avant authentification) :
```
Banner /etc/issue.net
```

```bash
sudo sshd -t
sudo systemctl restart ssh
```

> **Différence entre `ssh` et `sshd` :**  
> `ssh` (sans **d**) est la commande **client** — celle utilisée localement pour se connecter à une machine distante. `sshd` (*daemon*) est le **service serveur** qui tourne en permanence sur la VM, écoute le port 22, applique la configuration de `sshd_config` (chroot, bannière, restrictions...) et accepte ou refuse les connexions entrantes.

**Troisième scan, après fail2ban et bannière — Hardening index : 69/100**

> **Pourquoi le gain est-il plus faible qu'après les correctifs précédents ?**  
> [À compléter]

---

### 4.8 — Scanner de malware (rkhunter)

#### Installation et préparation

```bash
sudo apt install rkhunter
sudo rkhunter --update
sudo rkhunter --propupd
```

> **À quoi sert `--propupd`, et pourquoi le lancer juste après l'installation ?**  
> Cette commande crée une base de référence des propriétés des fichiers système actuels (tailles, hash, permissions) — elle établit "l'état sain" de la VM à cet instant. Les scans suivants comparent par rapport à cette référence, plutôt que de générer de faux positifs sur des fichiers légitimes.

(Note : `rkhunter --update` a renvoyé un avertissement mineur de configuration — `Invalid WEB_CMD configuration option` — sans impact sur le fonctionnement de l'outil ni sur les résultats des scans.)

#### Premier scan

```bash
sudo rkhunter --check
```

Résultat : `Suspect files: 0`, `Possible rootkits: 0` — aucune infection détectée. Un seul `[ Warning ]` relevé, sur la section "hidden files and directories" :

```bash
sudo grep -i "hidden" /var/log/rkhunter.log
```
```
Warning: Hidden file found: /etc/.resolv.conf.systemd-resolved.bak: ASCII text
Warning: Hidden file found: /etc/.updated: ASCII text
```

Vérification de la nature de ces fichiers — deux faux positifs connus et documentés, sans rapport avec un quelconque malware :
- `/etc/.updated` : créé par `systemd-update-done.service`, contient uniquement un horodatage de dernière mise à jour
- `/etc/.resolv.conf.systemd-resolved.bak` : sauvegarde automatique créée par `systemd-resolved`

#### Whitelisting et second scan

```bash
sudo nano /etc/rkhunter.conf
```
Ajout :
```
ALLOWHIDDENFILE=/etc/.updated
ALLOWHIDDENFILE=/etc/.resolv.conf.systemd-resolved.bak
```

> **Pourquoi whitelister explicitement plutôt que d'ignorer le warning ?**  
> [À compléter]

```bash
sudo rkhunter --check --sk
```

Résultat : `No warnings were found while checking the system.` ✅

#### Quatrième scan Lynis — Hardening index : 69/100 (inchangé)

```bash
sudo lynis audit system
```

Le composant `Malware scanner` passe de `[X]` à `[V]`, confirmant la résolution de `HRDN-7230` — mais le score global reste à 69/100, sans gain visible.

> **Pourquoi le score n'augmente-t-il pas malgré la correction ?**  
> Le "Hardening index" de Lynis est un score pondéré, pas une moyenne simple par suggestion corrigée. `HRDN-7230` a un poids marginal dans le calcul global, contrairement aux suggestions volontairement laissées hors scope (partitionnement disque dédié, mot de passe GRUB, auditd, process accounting, serveur de logs externe...), qui pèsent davantage. Un score de 69/100 ici reflète des choix de scope assumés pour un environnement d'exercice, pas des failles non traitées.

---

## 5. Vérification finale

> *Section à compléter*

---

*Document rédigé dans le cadre d'un exercice technique d'alternance.*
