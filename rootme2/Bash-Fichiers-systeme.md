# Bash - Fichiers système (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32360
Utilisateur: root
Mot de passe: da2db9810e9beedf0218ddaf5b85230a
```

### Commande de connexion SSH

```bash
ssh root@sockets.challenges.root-me.pro -p 32360
```

---

## Étape 1 : Utilisateur avec zsh comme shell

### Objectif
Trouver la description de l'utilisateur dont le bash par défaut est zsh.

### Commande utilisée

```bash
# Rechercher tous les utilisateurs ayant zsh comme shell
grep zsh /etc/passwd
```

### Sortie complète

```
root@deployment:~# grep zsh /etc/passwd
matt17:x:9991:9991:you_found_me_:/home/matt17:/bin/usr/zsh
```

### Explication

Le fichier `/etc/passwd` contient les informations de tous les utilisateurs du système.

**Format de chaque ligne :**
```
username:password:UID:GID:description:home_directory:shell
```

**Décomposition de la ligne trouvée :**
```
matt17:x:9991:9991:you_found_me_:/home/matt17:/bin/usr/zsh
│      │ │    │    │             │             └── Shell par défaut (zsh)
│      │ │    │    │             └── Répertoire home
│      │ │    │    └── Description/commentaire (GECOS)
│      │ │    └── GID (Group ID)
│      │ └── UID (User ID)
│      └── Mot de passe (x = stocké dans /etc/shadow)
└── Nom d'utilisateur
```

| Champ | Valeur |
|-------|--------|
| Username | matt17 |
| Password | x (dans /etc/shadow) |
| UID | 9991 |
| GID | 9991 |
| **Description** | **you_found_me_** |
| Home | /home/matt17 |
| Shell | /bin/usr/zsh |

### Réponse
```
you_found_me_
```

---

## Étape 2 : Validité du mot de passe

### Objectif
Trouver le nombre maximal de jours pendant lesquels le mot de passe de "matt17" reste valide.

### Commande utilisée

```bash
# Rechercher les informations de mot de passe de matt17
# Note: nécessite les droits root pour lire /etc/shadow
grep matt17 /etc/shadow
```

### Sortie complète

```
root@deployment:~# grep matt17 /etc/shadow
matt17:$y$j9T$/W.X1/FvwtUEgAvWhpLH5/$pijRlgkxMP9dJiPTVOuNURLqoalNlKswjVb3Z4Z5zuA:20399:13:37:6:::
```

### Explication

Le fichier `/etc/shadow` contient les mots de passe hashés et les politiques de mot de passe.

**Format de chaque ligne :**
```
username:password_hash:lastchange:min:max:warn:inactive:expire:reserved
```

**Décomposition de la ligne trouvée :**
```
matt17:$y$j9T$...:20399:13:37:6:::
│      │           │     │  │  │ │ │ └── Réservé
│      │           │     │  │  │ │ └── Date d'expiration du compte
│      │           │     │  │  │ └── Jours d'inactivité avant désactivation
│      │           │     │  │  └── Jours d'avertissement avant expiration
│      │           │     │  └── Jours MAXIMUM de validité du mot de passe
│      │           │     └── Jours minimum avant changement possible
│      │           └── Dernier changement (jours depuis 1er Jan 1970)
│      └── Hash du mot de passe (algorithme yescrypt)
└── Nom d'utilisateur
```

| Champ | Valeur | Description |
|-------|--------|-------------|
| username | matt17 | Nom d'utilisateur |
| password | $y$j9T$... | Hash yescrypt du mot de passe |
| lastchange | 20399 | Dernier changement (jours depuis epoch) |
| min | 13 | Minimum 13 jours avant de pouvoir changer |
| **max** | **37** | **Maximum 37 jours de validité** |
| warn | 6 | Avertissement 6 jours avant expiration |
| inactive | (vide) | Pas de période d'inactivité |
| expire | (vide) | Pas de date d'expiration du compte |

### Réponse
```
37
```

---

## Étape 3 : ID du groupe

### Objectif
Trouver l'ID du groupe "linux-expert".

### Commande utilisée

```bash
# Rechercher le groupe linux-expert
grep linux-expert /etc/group
```

### Sortie complète

```
root@deployment:~# grep linux-expert /etc/group
linux-expert:x:443:matt17
```

### Explication

Le fichier `/etc/group` contient les informations de tous les groupes du système.

**Format de chaque ligne :**
```
groupname:password:GID:members
```

**Décomposition de la ligne trouvée :**
```
linux-expert:x:443:matt17
│            │ │   └── Liste des membres (séparés par des virgules)
│            │ └── GID (Group ID)
│            └── Mot de passe du groupe (x = non utilisé)
└── Nom du groupe
```

| Champ | Valeur |
|-------|--------|
| Nom du groupe | linux-expert |
| Password | x (non utilisé) |
| **GID** | **443** |
| Membres | matt17 |

### Réponse
```
443
```

---

## Étape 4 : Entrée DNS hardcodée

### Objectif
Trouver le FQDN d'une entrée DNS hardcodée sur le système.

### Commande utilisée

```bash
# Afficher le contenu du fichier hosts
cat /etc/hosts
```

### Sortie complète

```
root@deployment:~# cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
fe00::0         ip6-localnet
fe00::0         ip6-mcastprefix
fe00::1         ip6-allnodes
fe00::2         ip6-allrouters
172.18.123.228  deployment-019c89d1-eb02-7338-941b-96102b061ba7-cc956776c-pkgw9
fc00:2000:2000:a372:a8:dd08:be2e:bb1a   deployment-019c89d1-eb02-7338-941b-96102b061ba7-cc956776c-pkgw9
192.168.43.1    you.found-me.local
```

### Explication

Le fichier `/etc/hosts` permet de définir des résolutions DNS locales, sans passer par un serveur DNS.

**Format de chaque ligne :**
```
IP_ADDRESS    HOSTNAME [ALIAS...]
```

**Entrées du fichier :**

| IP | Hostname | Type |
|----|----------|------|
| 127.0.0.1 | localhost | Standard (loopback IPv4) |
| ::1 | localhost | Standard (loopback IPv6) |
| fe00::0 | ip6-localnet | Standard (IPv6) |
| 172.18.123.228 | deployment-... | Kubernetes (auto-généré) |
| **192.168.43.1** | **you.found-me.local** | **Entrée personnalisée (le flag!)** |

### Réponse
```
you.found-me.local
```

---

## Étape 5 : Secret dans sudoers

### Objectif
Trouver le secret caché dans le fichier qui gère les droits sudo.

### Commande utilisée

```bash
# Afficher le contenu du fichier sudoers
# Note: nécessite les droits root
cat /etc/sudoers
```

### Sortie complète

```
root@deployment:~# cat /etc/sudoers
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# This fixes CVE-2005-4890 and possibly breaks some versions of kdesu
Defaults        use_pty

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "@include" directives:
@includedir /etc/sudoers.d

# Congrats ! Here is the secret : a61f4df39c169677
```

### Explication

Le fichier `/etc/sudoers` définit qui peut exécuter quelles commandes avec `sudo`.

**Éléments clés du fichier :**

| Ligne | Signification |
|-------|---------------|
| `Defaults env_reset` | Réinitialise l'environnement pour sudo |
| `Defaults secure_path="..."` | Définit le PATH sécurisé pour sudo |
| `root ALL=(ALL:ALL) ALL` | root peut tout faire |
| `%sudo ALL=(ALL:ALL) ALL` | Les membres du groupe sudo peuvent tout faire |
| `@includedir /etc/sudoers.d` | Inclut les fichiers du dossier sudoers.d |

**Le secret était caché dans un commentaire à la fin du fichier.**

### Réponse
```
a61f4df39c169677
```

---

## Récapitulatif des fichiers système Linux

| Fichier | Description | Accès |
|---------|-------------|-------|
| `/etc/passwd` | Informations utilisateurs (username, UID, GID, description, home, shell) | Lecture publique |
| `/etc/shadow` | Mots de passe hashés et politique de validité | Root uniquement |
| `/etc/group` | Informations des groupes (nom, GID, membres) | Lecture publique |
| `/etc/hosts` | Résolutions DNS locales hardcodées | Lecture publique |
| `/etc/sudoers` | Configuration des droits sudo | Root uniquement |
| `/etc/sudoers.d/` | Fichiers de configuration sudo additionnels | Root uniquement |

## Commandes utiles

```bash
# Rechercher un utilisateur par son shell
grep /bin/bash /etc/passwd

# Rechercher un utilisateur spécifique
grep username /etc/passwd

# Voir les infos de mot de passe (root requis)
grep username /etc/shadow

# Voir tous les groupes d'un utilisateur
groups username

# Rechercher un groupe
grep groupname /etc/group

# Voir les entrées DNS locales
cat /etc/hosts

# Vérifier la syntaxe de sudoers
visudo -c

# Éditer sudoers de manière sécurisée
visudo
```

## Automatisation avec expect (depuis une machine distante)

```bash
# Exemple de connexion SSH automatisée avec expect
expect -c '
spawn ssh -o StrictHostKeyChecking=no -p 32360 root@sockets.challenges.root-me.pro
expect "password:"
send "da2db9810e9beedf0218ddaf5b85230a\r"
expect "# "
send "grep zsh /etc/passwd\r"
expect "# "
send "exit\r"
expect eof
'
```
