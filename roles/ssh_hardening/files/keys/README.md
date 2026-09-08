# Clés publiques de l'équipe

Ce dossier contient **uniquement des clés publiques** (`.pub`).
Jamais de clé privée dans un dépôt Git.

## Clés en place

| Fichier | Compte Linux créé | Type | Empreinte SHA256 |
|---------|-------------------|------|------------------|
| `yan.pub` | `yan` | RSA 2048 | `p6LiwCFtVOhl9DeOXUn8sSKbTvZysPjFh17MJy7FJaI` |
| `hippo.pub` | `hippo` | RSA 2048 | `7T6y/tOZTFj8eclJJ665o09ML3B/nZd3xULoHDUcS3g` |
| `eliaz.pub` | `eliaz` | RSA 2048 | `cGijZyMyhGo4aZilEGi/Q8CgueHfmzt2eQMzCncTmVs` |

La correspondance fichier ↔ compte est définie par la liste `admins` dans
`inventory/group_vars/all.yml`.

Vérifier une empreinte :

```bash
ssh-keygen -lf yan.pub
```

## Note sur les clés PuTTY (Windows)

PuTTYgen enregistre par défaut un format qui n'est **pas** celui attendu ici :

```
---- BEGIN SSH2 PUBLIC KEY ----
Comment: "rsa-key-20260908"
AAAAB3NzaC1yc2EAAAADAQAB...
---- END SSH2 PUBLIC KEY ----
```

OpenSSH attend une **seule ligne** commençant par le type de clé :

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB... commentaire
```

Dans PuTTYgen, la bonne version est celle du grand champ en haut de la
fenêtre, intitulé *« Public key for pasting into OpenSSH authorized_keys
file »* — pas le fichier produit par le bouton *Save public key*.

Conversion en ligne de commande si besoin :

```bash
ssh-keygen -i -f cle_putty.pub > cle_openssh.pub
```

## Ajouter un administrateur

1. Il génère sa paire de clés :

```bash
ssh-keygen -t ed25519 -a 100 -C "prenom@baie-cyber" -f ~/.ssh/id_ed25519_baie
```

2. Il envoie **le `.pub` uniquement**, que l'on dépose ici sous
   `prenom.pub`.

3. On ajoute son entrée dans la liste `admins` de
   `inventory/group_vars/all.yml` :

```yaml
  - name: "prenom"
    comment: "Prenom Nom - Administrateur"
    key_file: "prenom.pub"
    shell: "/bin/bash"
    sudo: true
```

4. On propage sur tout le parc :

```bash
ansible-playbook ssh-config.yml
```

## Retirer un administrateur

Supprimer son entrée de la liste `admins`, puis relancer `ssh-config.yml`.
Les clés étant déployées en mode exclusif, son `authorized_keys` est réécrit
et son accès disparaît de toutes les machines en une commande.

## Pourquoi ed25519 plutôt que RSA

Les clés RSA 2048 en place fonctionnent : le `sshd_config` du projet accepte
`rsa-sha2-512`. Pour les prochaines clés, préférez cependant ed25519 :

- clé nettement plus courte à sécurité équivalente ou supérieure ;
- signature et vérification plus rapides ;
- pas de choix de taille à faire, donc pas de risque de générer trop petit ;
- l'option `-a 100` durcit le chiffrement de la clé privée sur le disque,
  ce qui ralentit une attaque par force brute si le fichier est volé.

Dans tous les cas, mettez une **passphrase**. Une clé privée sans passphrase
est un mot de passe écrit sur un post-it : quiconque copie le fichier obtient
l'accès.
