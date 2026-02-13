# TP — Administration d'un système Linux (Debian)

**Matière :** Administration Système Linux  

---

## Partie I — Administration des utilisateurs

### 1. Connexion en tant que root

```bash
su -
```

La connexion s'effectue sans problème. Le prompt passe en `root@debian:~#`.

---

### 2. Liste des comptes utilisateurs et des groupes existants

**Comptes utilisateurs :**

```bash
getent passwd
```

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
sshd:x:991:65534:sshd user:/run/sshd:/usr/sbin/nologin
postfix:x:101:103:Postfix MTA:/var/spool/postfix:/usr/sbin/nologin
systemd-timesync:x:990:990:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:989:989:System Message Bus:/nonexistent:/usr/sbin/nologin
```

On constate que le système ne contient aucun compte utilisateur "humain" pour l'instant (tous les comptes listés sont des comptes système avec un shell `/usr/sbin/nologin` ou `/bin/false`).

**Groupes :**

```bash
getent group
```

```
root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:
tty:x:5:
disk:x:6:
lp:x:7:
mail:x:8:
news:x:9:
uucp:x:10:
man:x:12:
proxy:x:13:
kmem:x:15:
dialout:x:20:
fax:x:21:
voice:x:22:
cdrom:x:24:
floppy:x:25:
tape:x:26:
sudo:x:27:
audio:x:29:
dip:x:30:
www-data:x:33:
backup:x:34:
operator:x:37:
list:x:38:
irc:x:39:
src:x:40:
shadow:x:42:
utmp:x:43:
video:x:44:
sasl:x:45:
plugdev:x:46:
staff:x:50:
games:x:60:
users:x:100:
nogroup:x:65534:
systemd-journal:x:999:
systemd-network:x:998:
crontab:x:997:
input:x:996:
sgx:x:995:
clock:x:994:
kvm:x:993:
render:x:992:
_ssh:x:101:
netdev:x:102:
postfix:x:103:
postdrop:x:104:
systemd-timesync:x:990:
messagebus:x:989:
```

---

### 3. UID et GID du compte root

```bash
id root
```

```
uid=0(root) gid=0(root) groups=0(root)
```

**Résultat :** l'UID de root est **0** et son GID est **0**. C'est la convention standard sur tous les systèmes Unix/Linux : le super-utilisateur possède toujours l'identifiant 0.

---

### 4. Valeurs minimales UID/GID par défaut et modification

**Valeurs par défaut :**

```bash
grep -E '^(UID_MIN|GID_MIN)' /etc/login.defs
```

```
UID_MIN                  1000
GID_MIN                  1000
```

Par défaut, Debian attribue les UID et GID à partir de **1000** pour les comptes utilisateurs.

**Modification pour UID_MIN = 2000 et GID_MIN = 1500 :**

Édition du fichier `/etc/login.defs` (utilisé par `useradd`) :

```bash
sed -i 's/^UID_MIN.*/UID_MIN  2000/' /etc/login.defs
sed -i 's/^GID_MIN.*/GID_MIN  1500/' /etc/login.defs
```

Édition du fichier `/etc/adduser.conf` (utilisé par `adduser`, spécifique à Debian) :

```bash
sed -i 's/^FIRST_UID=.*/FIRST_UID=2000/' /etc/adduser.conf
sed -i 's/^FIRST_GID=.*/FIRST_GID=1500/' /etc/adduser.conf
```

Les deux fichiers ont été modifiés afin que, quelle que soit la commande utilisée (`useradd` ou `adduser`), les prochains comptes soient créés à partir d'UID 2000 et de GID 1500.

---

### 5. Création des groupes grp1, grp2 et grp3

```bash
groupadd grp1
groupadd grp2
groupadd -g 1523 grp3
```

**Vérification :**

```bash
getent group grp1 grp2 grp3
```

```
grp1:x:1500:
grp2:x:1501:
grp3:x:1523:
```

On observe que grp1 a reçu le GID **1500** (nouveau minimum configuré), grp2 le GID **1501** (incrémenté automatiquement), et grp3 le GID **1523** (forcé avec l'option `-g`).

---

### 6. Création des utilisateurs util1, util2 et util3

**util1** — groupe principal grp1, pseudonyme "tux1" renseigné dans le champ GECOS :

```bash
useradd -m -g grp1 -c "tux1" util1
```

**util2** — groupe principal grp2, puis ajout aux groupes secondaires grp1 et grp3 :

```bash
useradd -m -g grp2 util2
usermod -aG grp1,grp3 util2
```

**util3** — groupe principal grp3 :

```bash
useradd -m -g grp3 util3
```

**Définition des mots de passe** (nécessaire pour la connexion console plus tard) :

```bash
passwd util1
passwd util2
passwd util3
```

---

### 7. UID et GID des comptes et groupes créés

```bash
id util1 util2 util3
```

```
uid=2000(util1) gid=1500(grp1) groups=1500(grp1)
uid=2001(util2) gid=1501(grp2) groups=1501(grp2),1500(grp1),1523(grp3)
uid=2002(util3) gid=1523(grp3) groups=1523(grp3)
```

**Récapitulatif :**

| Compte / Groupe | UID  | GID (principal) | Groupes secondaires |
|-----------------|------|-----------------|---------------------|
| util1           | 2000 | 1500 (grp1)     | —                   |
| util2           | 2001 | 1501 (grp2)     | grp1, grp3          |
| util3           | 2002 | 1523 (grp3)     | —                   |

| Groupe | GID  |
|--------|------|
| grp1   | 1500 |
| grp2   | 1501 |
| grp3   | 1523 |

Les UID démarrent bien à 2000 et les GID à 1500, conformément à la configuration effectuée à l'étape 4.

---

### 8. Suppression du groupe grp3 — est-ce possible ?

```bash
groupdel grp3
```

```
groupdel: cannot remove the primary group of user 'util3'
```

**Réponse :** Non, la suppression est **impossible**. Le système refuse car grp3 est le **groupe principal** de l'utilisateur util3. Sous Linux, un groupe ne peut pas être supprimé tant qu'il est référencé comme groupe principal d'au moins un compte utilisateur. Il faut d'abord supprimer ou modifier le compte concerné.

---

### 9. Suppression de util3 (sans son répertoire) puis suppression de grp3

Suppression du compte util3 **sans** l'option `-r` (le répertoire `/home/util3` est conservé) :

```bash
userdel util3
```

Puis suppression du groupe grp3 (désormais possible car plus aucun utilisateur ne l'a comme groupe principal) :

```bash
groupdel grp3
```

La commande s'exécute sans erreur cette fois.

---

### 10. Propriétaire actuel du répertoire /home/util3

```bash
ls -ld /home/util3
```

```
drwx------ 2 2002 1523 4096 Feb 13 10:18 /home/util3
```

**Constat :** le répertoire affiche les identifiants numériques **2002** et **1523** au lieu de noms. C'est parce que le compte util3 (UID 2002) et le groupe grp3 (GID 1523) ont été supprimés : il n'existe plus de correspondance dans `/etc/passwd` ni dans `/etc/group`. Les fichiers deviennent des **fichiers orphelins** — ils appartiennent à un UID/GID qui ne correspond plus à aucun compte existant.

---

### 11. Retrouver et supprimer tous les fichiers orphelins

**Recherche** des fichiers appartenant à l'ancien UID 2002 ou à l'ancien GID 1523 :

```bash
find / -xdev \( -uid 2002 -o -gid 1523 \) -print 2>/dev/null
```

```
/home/util3
/home/util3/.profile
/home/util3/.bashrc
/home/util3/.bash_logout
```

**Suppression** de ces fichiers orphelins :

```bash
find / -xdev \( -uid 2002 -o -gid 1523 \) -delete 2>/dev/null
```

L'option `-xdev` limite la recherche au système de fichiers courant (évite de parcourir les montages réseau ou les pseudo-systèmes de fichiers comme `/proc` et `/sys`).

---

### 12. Connexion en tant que util1 sur une console

```bash
su - util1
```

Pour que la connexion fonctionne correctement, il faut que :

- un **mot de passe** ait été défini pour util1 (`passwd util1`),
- le **shell** attribué soit valide (par défaut `/bin/bash` ou `/bin/sh`),
- le compte ne soit pas **verrouillé** (vérifiable avec `passwd -S util1`).

Si ces conditions sont remplies, la connexion se déroule normalement et l'utilisateur arrive dans son répertoire `/home/util1` avec les droits associés au groupe grp1.

---

## Partie II — Cron

### 1. Connexion en tant que root

```bash
su -
```

---

### 2. Vérification du démon cron

```bash
systemctl status cron
```

```
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-02-13 10:09:17 CET; 11min ago
     Main PID: 77 (cron)
```

Le service cron est **actif et en cours d'exécution**. On s'assure qu'il est activé au démarrage :

```bash
systemctl start cron
systemctl enable cron
```

> **Note Debian :** le service s'appelle `cron` et non `crond` (contrairement à CentOS/RHEL).

---

### 3. Consultation de /etc/cron.allow et /etc/cron.deny

```bash
ls -l /etc/cron.allow /etc/cron.deny
```

```
ls: cannot access '/etc/cron.allow': No such file or directory
ls: cannot access '/etc/cron.deny': No such file or directory
```

**Constat :** les deux fichiers sont **absents**. Sur Debian, en l'absence de ces fichiers, **tous les utilisateurs** sont autorisés à utiliser `crontab` par défaut. Le comportement est le suivant :

- Si `cron.allow` existe → seuls les utilisateurs listés dedans peuvent utiliser crontab.
- Si `cron.allow` n'existe pas mais `cron.deny` existe → tous sauf ceux listés dans `cron.deny`.
- Si aucun des deux n'existe → tout le monde est autorisé.

---

### 4. Connexion en tant que "user"

L'énoncé demande de se connecter en tant que `user`. Ce compte n'existant pas sur le système, il faut le créer au préalable :

```bash
useradd -m user
passwd user
```

Puis, dans un autre terminal :

```bash
su - user
```

---

### 5. Affichage de la crontab de user

```bash
crontab -l
```

```
no crontab for user
```

La crontab est vide, aucune tâche n'est planifiée.

---

### 6. Programmation d'une tâche toutes les minutes

```bash
crontab -e
```

Ajout de la ligne suivante :

```
* * * * * /bin/pwd > /tmp/pwd.out
```

Le format cron est : `minute heure jour_du_mois mois jour_de_la_semaine commande`. Ici, les cinq étoiles signifient "toutes les minutes, toutes les heures, tous les jours".

---

### 7. Vérification après une minute

```bash
cat /tmp/pwd.out
```

```
/home/user
```

Le fichier contient le résultat de la commande `pwd`, exécutée dans le contexte du répertoire personnel de l'utilisateur `user`. Cela confirme que la tâche cron fonctionne correctement.

---

### 8. Arrêt du démon cron (en root)

```bash
su -
systemctl stop cron
```

---

### 9. Suppression du fichier /tmp/pwd.out

```bash
rm -f /tmp/pwd.out
```

---

### 10. Ajout de user dans /etc/cron.deny

```bash
echo "user" >> /etc/cron.deny
```

Ce fichier est créé automatiquement s'il n'existe pas. L'utilisateur `user` est désormais interdit d'accès à la commande `crontab`.

---

### 11. Redémarrage du démon cron

```bash
systemctl start cron
```

---

### 12. Tentative d'édition de la crontab sous user

```bash
su - user
crontab -e
```

```
You (user) are not allowed to use this program (crontab)
See crontab(1) for more information
```

**Résultat :** l'édition est **refusée**. L'utilisateur `user` figure dans `/etc/cron.deny` et ne peut donc plus créer ni modifier sa crontab.

---

### 13. Vérification de /tmp/pwd.out après une minute — déduction

```bash
cat /tmp/pwd.out
```

```
/home/user
```

**Déduction importante :** même si l'utilisateur `user` est désormais dans `/etc/cron.deny` et ne peut plus modifier sa crontab, la **tâche déjà programmée continue de s'exécuter**. Le fichier `cron.deny` empêche uniquement l'accès à la commande `crontab` (création/modification/suppression), mais n'annule pas les tâches déjà en place. Seul un administrateur (root) peut supprimer la crontab de l'utilisateur bloqué.

---

### 14. Suppression de la crontab de user et redémarrage (en root)

```bash
su -
crontab -r -u user
systemctl restart cron
```

La crontab de `user` est vidée par l'administrateur root. Le service cron est redémarré pour s'assurer de la prise en compte.

---

### 15. Tâche système : nettoyage de /tmp à 9h30 et 13h30

Création d'un fichier dans `/etc/cron.d/` (crontab système) :

```bash
nano /etc/cron.d/clean-tmp
```

Contenu du fichier :

```
# Nettoyage des fichiers temporaires modifiés il y a plus d'un jour
# Exécution à 9h30 et 13h30 tous les jours
30 9,13 * * * root find /tmp -type f -mtime +1 -delete
```

Sécurisation des permissions :

```bash
chmod 644 /etc/cron.d/clean-tmp
```

**Explication de la commande :**

- `30 9,13 * * *` → à la minute 30, aux heures 9 et 13, tous les jours
- `root` → exécuté en tant que root (obligatoire dans les crontabs système)
- `find /tmp -type f -mtime +1 -delete` → recherche et supprime les fichiers réguliers dans `/tmp` dont la date de modification est supérieure à 1 jour (24h)

---

### 16. Prise en compte des modifications

```bash
systemctl restart cron
```

Le démon cron surveille automatiquement les modifications dans `/etc/cron.d/`, mais un redémarrage explicite garantit la prise en compte immédiate des changements.

---

## Partie III — Les logs

### 1. Aller dans le répertoire /var/log

```bash
cd /var/log
pwd
```

```
/var/log
```

Le répertoire `/var/log` est le répertoire centralisé contenant l'ensemble des fichiers journaux (logs) du système Linux. On y retrouve les logs du noyau, des services, de l'authentification, du courrier, etc.

---

### 2. Lister les fichiers avec affichage détaillé, du plus ancien au plus récent

```bash
ls -ltr
```

```
total 240
-rw-rw----  1 root utmp                 0 Oct  1 16:43 btmp
lrwxrwxrwx  1 root root                39 Oct  1 16:43 README -> ../../usr/share/doc/systemd/README.logs
drwx------  2 root root              4096 Oct  1 16:43 private
drwxr-sr-x+ 2 root systemd-journal   4096 Oct  1 16:43 journal
drwxr-xr-x  3 root root              4096 Oct  1 16:43 runit
-rw-r--r--  1 root root                 0 Oct  1 16:44 syslog
drwxr-xr-x  2 root root              4096 Feb 13 10:10 apt
-rw-r--r--  1 root root             13023 Feb 13 10:10 alternatives.log
-rw-r--r--  1 root root            194527 Feb 13 10:10 dpkg.log
-rw-rw-r--  1 root utmp               384 Feb 13 10:12 wtmp
-rw-rw-r--  1 root utmp               292 Feb 13 10:12 lastlog
-rw-r--r--  1 root root              8192 Feb 13 10:20 wtmp.db
```

**Explication des options :**

- `-l` → affichage détaillé (permissions, propriétaire, taille, date de modification)
- `-t` → tri par date de modification (le plus récent en premier)
- `-r` → inverse l'ordre du tri (donc le plus ancien en premier, le plus récent en dernier)

La combinaison `-ltr` permet de visualiser rapidement quels fichiers de logs ont été modifiés récemment (ils apparaissent en bas de la liste).

**Observations :** les fichiers les plus anciens datent du 1er octobre (installation du système), tandis que les plus récents datent du 13 février (jour du TP). On remarque également que le fichier `syslog` est vide (taille 0) et que le fichier `messages` est absent.

---

### 3. Consulter le fichier des messages

```bash
cat /var/log/messages
```

```
cat: /var/log/messages: No such file or directory
```

Le fichier `/var/log/messages` n'existe pas. On tente alors `/var/log/syslog` :

```bash
cat /var/log/syslog
```

Le fichier existe mais est **vide** (0 octets). Cela est dû au fait que **rsyslog n'est pas installé** sur cette installation Debian minimale. Sans démon syslog, aucun fichier journal texte n'est généré.

**Installation de rsyslog :**

```bash
apt install rsyslog
```

```
Installing:
  rsyslog

Installing dependencies:
  libestr0  libfastjson4  liblognorm5

Summary:
  Upgrading: 0, Installing: 4, Removing: 0, Not Upgrading: 0
  Download size: 863 kB
...
Setting up rsyslog (8.2504.0-1) ...
Created symlink '/etc/systemd/system/syslog.service' → '/usr/lib/systemd/system/rsyslog.service'.
```

**Tentative de démarrage via systemd :**

```bash
systemctl start rsyslog
```

```
Job for rsyslog.service failed because the control process exited with error code.
```

```bash
systemctl status rsyslog
```

```
× rsyslog.service - System Logging Service
     Active: failed (Result: exit-code)
    Process: 3733 ExecStart=/usr/sbin/rsyslogd -n -iNONE (code=exited, status=226/NAMESPACE)
```

Le démarrage via systemd échoue avec le code **226/NAMESPACE**. Cette erreur est liée aux restrictions de sécurité systemd (`ProtectSystem`, `PrivateDevices`, etc.) qui ne sont pas compatibles avec l'environnement **conteneur LXC** dans lequel tourne cette Debian (sur un hôte Proxmox VE).

De même, `journalctl` ne retourne aucune entrée car `systemd-journald` ne fonctionne pas non plus dans ce conteneur :

```bash
journalctl
```

```
No journal files were found.
-- No entries --
```

**Solution — lancement manuel de rsyslogd :**

```bash
/usr/sbin/rsyslogd
```

Cette fois rsyslog se lance correctement (sans les restrictions de namespace de systemd). Le fichier `/var/log/syslog` se remplit immédiatement :

```bash
ls -la /var/log/syslog /var/log/messages
```

```
ls: cannot access '/var/log/messages': No such file or directory
-rw-r--r-- 1 root root 202154 Feb 13 13:39 /var/log/syslog
```

**Consultation du fichier syslog :**

```bash
less /var/log/syslog
```

Extrait (premières lignes — logs du noyau) :

```
2026-02-13T13:35:42 debian kernel: Linux version 6.17.4-2-pve (build@proxmox)...
2026-02-13T13:35:42 debian kernel: DMI: HP HP ProDesk 600 G6 Desktop Mini PC/8715
2026-02-13T13:35:42 debian kernel: smpboot: CPU0: Intel(R) Core(TM) i5-10500T CPU @ 2.30GHz
2026-02-13T13:35:42 debian kernel: Memory: 16022344K/16547040K available
2026-02-13T13:35:42 debian kernel: e1000e 0000:00:1f.6 nic0: NIC Link is Up 100 Mbps Full Duplex
2026-02-13T13:35:42 debian kernel: EXT4-fs (dm-1): mounted filesystem ... r/w with ordered data mode
...
```

Le fichier contient l'intégralité des messages du noyau depuis le démarrage du système, les événements réseau, les montages de systèmes de fichiers, ainsi que de nombreux messages d'audit AppArmor liés aux conteneurs LXC.

> **Note :** `/var/log/messages` n'est pas créé par défaut sur Debian. La configuration par défaut de rsyslog (`/etc/rsyslog.conf`) redirige les messages principaux vers `/var/log/syslog`. Pour activer `/var/log/messages`, il faut décommenter la règle correspondante dans `/etc/rsyslog.conf`.

---

### 4. Visualiser les dernières lignes avec mise à jour en temps réel

Maintenant que rsyslog fonctionne (lancé manuellement), on peut utiliser `tail -f` sur `/var/log/syslog` :

```bash
tail -f /var/log/syslog
```

```
2026-02-13T13:39:35 debian kernel: audit: type=1400 apparmor="DENIED" operation="userns_create"...
2026-02-13T13:39:35 debian kernel: audit: type=1400 apparmor="DENIED" operation="mount"...
...
```

Pour quitter l'affichage en temps réel : **Ctrl + C**.

> **Note :** `tail -f /var/log/messages` n'est pas possible sur cette Debian car `/var/log/messages` n'est pas créé par défaut (voir note section 3). On utilise `/var/log/syslog` à la place.

**Explication des commandes :**

- `tail -f <fichier>` → affiche les dernières lignes d'un fichier et suit en continu les nouvelles lignes ajoutées. L'option `-f` signifie "follow".
- `tail -n 50 -f /var/log/syslog` → affiche les 50 dernières lignes puis suit en temps réel.

Cette commande est essentielle en administration système pour **surveiller en direct** l'activité du système : démarrage de services, tentatives de connexion, erreurs, etc.

---

## Récapitulatif des commandes clés

| Action | Commande |
|--------|----------|
| Lister les utilisateurs | `getent passwd` |
| Lister les groupes | `getent group` |
| Voir UID/GID d'un user | `id <user>` |
| Créer un groupe avec GID spécifique | `groupadd -g <GID> <groupe>` |
| Créer un utilisateur avec groupe principal | `useradd -m -g <groupe> <user>` |
| Ajouter un user à des groupes secondaires | `usermod -aG <grp1>,<grp2> <user>` |
| Supprimer un utilisateur (garder home) | `userdel <user>` |
| Supprimer un groupe | `groupdel <groupe>` |
| Trouver les fichiers orphelins | `find / -xdev \( -uid <UID> -o -gid <GID> \) -print` |
| Vérifier le statut de cron | `systemctl status cron` |
| Éditer la crontab d'un user | `crontab -e` (en tant que user) |
| Supprimer la crontab d'un user (root) | `crontab -r -u <user>` |
| Créer une tâche cron système | Fichier dans `/etc/cron.d/` |
| Lister les logs du plus ancien au plus récent | `ls -ltr /var/log` |
| Consulter un fichier de log | `less /var/log/syslog` ou `journalctl` |
| Suivre un log en temps réel | `tail -f /var/log/syslog` ou `journalctl -f` |
