# Clés publiques de l'équipe

Déposez ici **uniquement les clés publiques** (fichiers `.pub`).
Jamais de clé privée dans un dépôt Git.

## Fichiers attendus

| Fichier          | Propriétaire | Compte créé sur les VM |
|------------------|--------------|------------------------|
| `yan.pub`        | Yan          | `yan`                  |
| `collegue1.pub`  | Collègue 1   | `collegue1`            |
| `collegue2.pub`  | Collègue 2   | `collegue2`            |

La correspondance fichier ↔ compte est définie dans la liste `admins`
de `inventory/group_vars/all.yml`. Pour ajouter un quatrième
administrateur : déposer sa clé ici, puis ajouter une entrée dans cette
liste.

## Générer une clé (chaque membre, sur son poste)

```bash
ssh-keygen -t ed25519 -a 100 -C "prenom@baie-cyber" -f ~/.ssh/id_ed25519_baie
```

- `-t ed25519` : algorithme moderne, court et rapide. Préférez-le à RSA.
- `-a 100` : 100 tours de dérivation pour la passphrase, ralentit une
  attaque par force brute sur la clé privée si elle est volée.
- Mettez une **passphrase**. Une clé privée sans passphrase, c'est un
  mot de passe écrit sur un post-it.

Chacun envoie ensuite **le fichier `.pub` uniquement** :

```bash
cat ~/.ssh/id_ed25519_baie.pub
```

## Vérifier une clé avant de l'intégrer

```bash
ssh-keygen -lf yan.pub
```

Doit afficher quelque chose comme :

```
256 SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx prenom@baie-cyber (ED25519)
```

Si la commande renvoie une erreur, le fichier est corrompu (souvent un
retour à la ligne ajouté par un copier-coller).
