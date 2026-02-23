# Bash - Fichiers communs (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32200
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32200
```

---

## Étape 1 : Secret dans le binaire "flag"

### Objectif
Trouver le premier secret contenu dans le binaire nommé `flag`.

### Commandes utilisées

```bash
# Trouver le binaire flag sur le système
find / -name flag 2>/dev/null
```

### Sortie

```
user@deployment:~$ find / -name flag 2>/dev/null
/usr/bin/flag
```

### Extraction du secret

```bash
# Extraire les chaînes lisibles du binaire
strings /usr/bin/flag
```

### Sortie

```
user@deployment:~$ strings /usr/bin/flag
#!/usr/bin/python3
# C0ngr4ts ! Th1s 1s p4rt 1 : 0c6434516f460834
print("The execution is not sufficient. Find me to get the answer to this question (:")
```

### Explication

La commande `strings` extrait toutes les chaînes de caractères lisibles d'un fichier binaire. Ici, le "binaire" est en fait un script Python, et le secret était caché dans un commentaire.

**Note :** Exécuter le binaire (`/usr/bin/flag`) ne donne pas le secret - il faut analyser son contenu.

### Réponse
```
0c6434516f460834
```

---

## Étape 2 : Fichier secret de l'utilisateur "caesar"

### Objectif
Trouver les identifiants de l'utilisateur "caesar" cachés dans un dossier système optionnel et donner le contenu de son fichier secret.

### Commandes utilisées

```bash
# Chercher dans le dossier optionnel /opt
find /opt -type f 2>/dev/null
```

### Sortie

```
user@deployment:~$ find /opt -type f 2>/dev/null
/opt/caesar.txt
```

### Lecture des identifiants

```bash
cat /opt/caesar.txt
```

### Sortie

```
user@deployment:~$ cat /opt/caesar.txt
Change for user "caesar" with : `su - caesar` using password `937d7ee26c1284e0adc9b66e83ef56b0`
```

### Connexion en tant que caesar

```bash
su - caesar
Password: 937d7ee26c1284e0adc9b66e83ef56b0
```

### Recherche du fichier secret

```bash
caesar@deployment:~$ ls -la
total 36
drwxrwx--- 2 caesar caesar 4096 Aug  7  2025 .
drwxr-xr-x 1 root   root   4096 Aug  7  2025 ..
-rwxrwx--- 1 caesar caesar  253 Aug  7  2025 .bash_history
-rwxrwx--- 1 caesar caesar  220 Apr 18  2025 .bash_logout
-rwxrwx--- 1 caesar caesar 3526 Apr 18  2025 .bashrc
-rwxrwx--- 1 caesar caesar  807 Apr 18  2025 .profile
-rwxrwx--- 1 caesar caesar  569 Aug  7  2025 .zsh_history
-rwxrwx--- 1 caesar caesar   39 Aug  7  2025 message.txt

caesar@deployment:~$ cat message.txt
ahahah you will never find my secret !
```

### Analyse de l'historique

Le fichier `message.txt` est un leurre. On analyse l'historique zsh :

```bash
caesar@deployment:~$ cat .zsh_history
: 1731941525:0;cd ~/
: 1731941898:0;ls -la
: 1731941911:0;mkdir -p super/secret/path
: 1731941920:0;PASSWD=$(cat /dev/urandom | tr -dc '[:graph:]' | head -c 16)
: 1731942275:0;echo "$PASSWD" > super/secret/path/flag-part-2.txt
: 1731943968:0;cd /tmp
: 1731944899:0;TMPDIR=$(mktemp -d -p /var/tmp/)
: 1731944903:0;cd $TMPDIR
: 1732001778:0;mv ~/super/secret/path/flag-part-2.txt .sec3.txt
: 1732003117:0;nano ~/message.txt
: 1732008912:0;rm -rf ~/super
: 1732019114:0;ls -l
: 1732019182:0;whoami
: 1732019305:0;ping -c 2 9.9.9.9
: 1732019505:0;echo nothing to see here
```

### Explication de l'historique

L'historique révèle que :
1. Un fichier `flag-part-2.txt` a été créé dans `~/super/secret/path/`
2. Il a été déplacé vers `/var/tmp/` et renommé `.sec3.txt`
3. Le dossier `~/super` a été supprimé

### Récupération du secret

```bash
caesar@deployment:~$ find /var/tmp -name .sec3.txt 2>/dev/null
/var/tmp/tmp.H1sy2PlVg8/.sec3.txt

caesar@deployment:~$ cat /var/tmp/tmp.H1sy2PlVg8/.sec3.txt
OQn_Z1lIIyAWEVTG
```

### Fichiers et dossiers clés

| Chemin | Description |
|--------|-------------|
| `/opt/` | Dossier système optionnel pour logiciels tiers |
| `/opt/caesar.txt` | Fichier contenant les identifiants |
| `.zsh_history` | Historique des commandes zsh |
| `/var/tmp/` | Dossier temporaire persistant |

### Réponse
```
OQn_Z1lIIyAWEVTG
```

---

## Étape 3 : Secret sur le disque dur externe

### Objectif
Trouver le troisième secret caché dans le disque dur externe monté sur la machine.

### Commandes utilisées

```bash
# Voir les systèmes de fichiers montés
mount | grep -v cgroup

# Ou lister le contenu de /mnt
ls -la /mnt/
```

### Sortie

```
root@deployment:~# mount | grep -v cgroup
none on / type virtiofs (rw,relatime)
...
kataShared on /root/.useless.txt type virtiofs (ro,relatime)
...

root@deployment:~# ls -la /mnt/
total 12
drwxr-xr-x 1 root root   4096 Aug  7  2025 .
drwxr-xr-x 1 root root   4096 Feb 23 09:35 ..
drwxrwx--- 2 root caesar 4096 Aug  7  2025 sda1
```

### Accès au disque externe

Le dossier `/mnt/sda1` est accessible par le groupe `caesar` :

```bash
# Se connecter en tant que caesar
su - caesar
Password: 937d7ee26c1284e0adc9b66e83ef56b0

caesar@deployment:~$ ls -la /mnt/sda1/
total 24
drwxrwx--- 2 root caesar 4096 Aug  7  2025 .
drwxr-xr-x 1 root root   4096 Aug  7  2025 ..
-rwxrwx--- 1 root caesar  184 Aug  7  2025 p4rt-3.md
-rwxrwx--- 1 root caesar   14 Aug  7  2025 payement.pdf
-rwxrwx--- 1 root caesar   14 Aug  7  2025 shared-file.docx
-rwxrwx--- 1 root caesar   14 Aug  7  2025 ticket.png
```

### Lecture du fichier secret

```bash
caesar@deployment:~$ cat /mnt/sda1/p4rt-3.md
# Hey

## Congrats !

Th1s is the F14g nb 3 : 56190ccd15f774ef

## How to become root ?

$ su -
Password: 79cc78fc2d59b5f8ed7a014fdf3c6076

## Made by

Root-Me PRO - 2025
```

### Explication

- `/mnt/` est le point de montage standard pour les périphériques externes
- `sda1` = première partition du premier disque (convention Linux)
- Le fichier contenait aussi le mot de passe root pour les étapes suivantes

### Points de montage courants

| Chemin | Description |
|--------|-------------|
| `/mnt/` | Point de montage temporaire pour périphériques |
| `/media/` | Point de montage automatique (clés USB, etc.) |
| `sda`, `sdb`... | Noms des disques (a=premier, b=second...) |
| `sda1`, `sda2`... | Partitions du disque |

### Réponse
```
56190ccd15f774ef
```

**Bonus - Mot de passe root :** `79cc78fc2d59b5f8ed7a014fdf3c6076`

---

## Étape 4 : Secret dans les logs nginx

### Objectif
Trouver le quatrième secret caché dans les fichiers journaux du software nginx.

### Commandes utilisées

```bash
# Se connecter en root
su -
Password: 79cc78fc2d59b5f8ed7a014fdf3c6076

# Lister les logs nginx
ls -la /var/log/nginx/
```

### Sortie

```
root@deployment:~# ls -la /var/log/nginx/
total 20
drw-r----- 2 www-data adm  4096 Aug  7  2025 .
drwxr-xr-x 1 root     root 4096 Aug  7  2025 ..
-rw-r--r-- 1 www-data adm  2601 Aug  7  2025 access.log
-rw-r--r-- 1 www-data adm  1156 Aug  7  2025 error.log
```

### Lecture des logs

```bash
root@deployment:~# cat /var/log/nginx/access.log
139.162.186.99 - - [30/Jul/2025:13:06:26 +0200] "GET / HTTP/1.0" 301 162 "-" "-"
139.162.186.99 - - [30/Jul/2025:13:06:26 +0200] "GET / HTTP/1.0" 301 162 "-" "-"
139.162.186.99 - - [30/Jul/2025:13:06:27 +0200] "GET / HTTP/1.1" 301 162 "-" "-"
184.105.247.194 - - [30/Jul/2025:13:15:25 +0200] "GET / HTTP/1.1" 301 162 "-" "Mozilla/5.0..."
184.105.247.194 - - [30/Jul/2025:13:17:19 +0200] "GET /webui/ HTTP/1.1" 301 162 "-" "Mozilla/5.0..."
184.105.247.194 - - [30/Jul/2025:13:18:35 +0200] "GET /Config.xml HTTP/1.1" 301 162 "-" "Mozilla/5.0..."
161.35.175.241 - - [30/Jul/2025:14:00:40 +0200] "GET /flaag-p4rt-4?value=db7eb81d2e736afb HTTP/1.1" 301 162 "-" "Mozilla/5.0;"
161.35.175.241 - - [30/Jul/2025:14:00:40 +0200] "GET /.env HTTP/1.1" 301 162 "-" "Mozilla/5.0;"
161.35.175.241 - - [30/Jul/2025:14:00:40 +0200] "GET /.git/config HTTP/1.1" 301 162 "-" "Mozilla/5.0;"
```

### Explication

Le secret est caché dans une requête HTTP GET :
```
GET /flaag-p4rt-4?value=db7eb81d2e736afb
```

### Fichiers de logs courants

| Fichier | Description |
|---------|-------------|
| `/var/log/nginx/access.log` | Logs des requêtes HTTP reçues |
| `/var/log/nginx/error.log` | Logs des erreurs nginx |
| `/var/log/syslog` | Logs système généraux |
| `/var/log/auth.log` | Logs d'authentification |

### Format des logs nginx (Combined Log Format)

```
IP - - [date] "METHOD /path HTTP/version" status size "referer" "user-agent"
```

### Réponse
```
db7eb81d2e736afb
```

---

## Étape 5 : Variable d'environnement d'un processus

### Objectif
Trouver un processus lancé avec une variable d'environnement intéressante et donner sa valeur.

### Commandes utilisées

```bash
# Lister les processus
ps aux
```

### Sortie

```
root@deployment:~# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   3928   264 ?        S    09:35   0:00 /bin/bash /en
root           2  0.0  0.6  25660 13488 ?        S    09:35   0:00 python3 -m ht
root           4  0.0  0.0  15444  1600 ?        S    09:35   0:00 sshd: /usr/sb
root         219 15.0  0.3  17612  6760 ?        Ss   09:40   0:00 sshd: user [p
user         225  0.0  0.2  17872  4916 ?        S    09:40   0:00 sshd: user@pt
user         226  4.3  0.0   4192   552 pts/0    Ss   09:40   0:00 -bash
root         229 31.5  0.0   5884  1724 pts/0    S    09:40   0:00 su -
root         230  0.0  0.0   4192  1816 pts/0    S    09:40   0:00 -bash
root         233  0.0  0.2   8108  4300 pts/0    R+   09:40   0:00 ps aux
```

### Lecture des variables d'environnement du processus

Le processus `python3 -m http.server` (PID 2) semble intéressant :

```bash
# Lire les variables d'environnement du processus PID 2
cat /proc/2/environ | tr "\0" "\n"
```

### Sortie

```
root@deployment:~# cat /proc/2/environ | tr "\0" "\n"
FL4G_P4RT_5=1e5c19052798f8d4
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=deployment-019c89d9-f623-719b-a5af-f6fb47451ee2-5598897fd-wwfm8
PWD=/
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
SHLVL=0
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/python3
```

### Explication

- `/proc/` est un système de fichiers virtuel contenant des informations sur les processus
- `/proc/[PID]/environ` contient les variables d'environnement d'un processus
- Les variables sont séparées par des caractères NULL (`\0`), d'où le `tr "\0" "\n"`

### Structure de /proc

| Chemin | Description |
|--------|-------------|
| `/proc/[PID]/` | Dossier du processus avec ce PID |
| `/proc/[PID]/environ` | Variables d'environnement |
| `/proc/[PID]/cmdline` | Ligne de commande complète |
| `/proc/[PID]/cwd` | Répertoire de travail actuel |
| `/proc/[PID]/exe` | Lien vers l'exécutable |
| `/proc/[PID]/fd/` | Descripteurs de fichiers ouverts |

### Réponse
```
1e5c19052798f8d4
```

---

## Récapitulatif des secrets

| Étape | Localisation | Secret |
|-------|--------------|--------|
| 1 | `/usr/bin/flag` (strings) | `0c6434516f460834` |
| 2 | `/var/tmp/tmp.*/​.sec3.txt` | `OQn_Z1lIIyAWEVTG` |
| 3 | `/mnt/sda1/p4rt-3.md` | `56190ccd15f774ef` |
| 4 | `/var/log/nginx/access.log` | `db7eb81d2e736afb` |
| 5 | `/proc/2/environ` | `1e5c19052798f8d4` |

## Identifiants découverts

| Utilisateur | Mot de passe | Source |
|-------------|--------------|--------|
| caesar | `937d7ee26c1284e0adc9b66e83ef56b0` | `/opt/caesar.txt` |
| root | `79cc78fc2d59b5f8ed7a014fdf3c6076` | `/mnt/sda1/p4rt-3.md` |

## Commandes utiles

```bash
# Recherche de fichiers
find / -name "pattern" 2>/dev/null

# Extraire les chaînes d'un binaire
strings /path/to/binary

# Voir les systèmes de fichiers montés
mount
df -h

# Logs système
ls -la /var/log/
cat /var/log/nginx/access.log

# Processus et leurs environnements
ps aux
cat /proc/[PID]/environ | tr "\0" "\n"

# Changer d'utilisateur
su - username
su -  # (pour root)
```

## Automatisation avec expect

```bash
# Connexion SSH automatisée
expect -c '
spawn ssh -o StrictHostKeyChecking=no -p 32200 user@sockets.challenges.root-me.pro
expect "password:"
send "password\r"
expect "$ "
send "strings /usr/bin/flag\r"
expect "$ "
send "exit\r"
expect eof
'
```
