# Bash - Grep n Find (Root-Me Challenge)

## Informations de connexion

```
SSH: ssh user@sockets.challenges.root-me.pro -p 32538
Password: password
```

---

## Partie 1 : Trouver les fichiers "find-me"

### Objectif
Trouver tous les fichiers dont le nom commence par "find-me" et les lister par ordre croissant du suffixe numérique.

### Commande utilisée

```bash
find / -name "find-me*" 2>/dev/null | sort -V
```

**Explication :**
- `find /` : Recherche depuis la racine du système
- `-name "find-me*"` : Fichiers dont le nom commence par "find-me"
- `2>/dev/null` : Redirige les erreurs (permissions refusées) vers /dev/null
- `sort -V` : Tri par version (ordre numérique naturel)

### Réponses

| Ordre | Chemin complet |
|-------|----------------|
| 1 | `/home/user/3adc020f9d1d0b0351d119266a1931e5/cc392bd4f1a2dd3d01e46048f86cff33/find-me-1.md` |
| 2 | `/find-me-2.txt` |
| 3 | `/etc/find-me-3.png` |
| 4 | `/var/log/find-me-4.mp3` |

---

## Partie 2 : Fichier contenant "hideonbush"

### Objectif
Trouver le chemin complet du fichier caché dans un des dossiers de `/home/user` dont le contenu contient la chaîne "hideonbush" (insensible à la casse).

### Commande utilisée

```bash
grep -rli hideonbush /home/user 2>/dev/null
```

**Explication :**
- `grep` : Recherche de motifs dans les fichiers
- `-r` : Recherche récursive dans les sous-dossiers
- `-l` : Affiche uniquement les noms de fichiers (pas le contenu)
- `-i` : Insensible à la casse (majuscules/minuscules)
- `2>/dev/null` : Supprime les messages d'erreur

### Réponse

```
/home/user/5bc577398bc11fc30285a0e0aeb32320/bf5816554f2d581b4cf8a21cdb7fe2a5/secret.txt
```

La chaîne trouvée dans le fichier : `HiDeOnBuSh`

---

## Partie 3 : Recherche par expression régulière

### Objectif
Trouver un fichier contenant une chaîne respectant les règles suivantes :
- Commence par 4 chiffres
- Suivi de 12 à 15 lettres (majuscules/minuscules), **sans** les lettres f, l, p
- Suivi de la lettre E
- Se termine par 4 chiffres

### Commande utilisée

```bash
grep -rEo '[0-9]{4}[abcdeghijkmnoqrstuvwxyzABCDEGHIJKMNOQRSTUVWXYZ]{12,15}E[0-9]{4}' /home/user 2>/dev/null
```

**Explication :**
- `-E` : Utilise les expressions régulières étendues (ERE)
- `-o` : Affiche uniquement la partie correspondante
- `[0-9]{4}` : Exactement 4 chiffres
- `[abcdeghijkmnoqrstuvwxyzABCDEGHIJKMNOQRSTUVWXYZ]{12,15}` : 12 à 15 lettres excluant f, l, p
- `E` : La lettre E littérale
- `[0-9]{4}` : Exactement 4 chiffres

### Réponse

**Fichier :**
```
/home/user/3adc020f9d1d0b0351d119266a1931e5/secret.txt
```

**Chaîne trouvée :** `1337cOngratsthAtsmE1337`

**Décomposition :**
| Partie | Valeur | Validation |
|--------|--------|------------|
| 4 chiffres | `1337` | ✓ |
| 12-15 lettres (sans f,l,p) | `cOngratsthAtsm` (14 lettres) | ✓ |
| Lettre E | `E` | ✓ |
| 4 chiffres | `1337` | ✓ |

---

## Récapitulatif des commandes utiles

| Commande | Description |
|----------|-------------|
| `find / -name "pattern"` | Rechercher des fichiers par nom |
| `grep -r "motif" /chemin` | Recherche récursive de texte |
| `grep -i` | Recherche insensible à la casse |
| `grep -l` | Affiche uniquement les noms de fichiers |
| `grep -E` | Expressions régulières étendues |
| `grep -o` | Affiche uniquement les correspondances |
| `sort -V` | Tri par version (numérique naturel) |
| `2>/dev/null` | Supprime les messages d'erreur |
