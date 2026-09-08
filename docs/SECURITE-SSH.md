# Le durcissement SSH, expliqué

Ce document reprend chaque mesure appliquée par le rôle `ssh_hardening`, ce
qu'elle empêche concrètement, et ce qu'il faut savoir en dire à l'oral.

---

## 1. Authentification : uniquement par clé

```
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
```

Les quatre directives sont nécessaires. C'est le piège classique : couper
`PasswordAuthentication` seul laisse `KbdInteractiveAuthentication` ouvert sur
certaines configurations PAM, et le mot de passe reste utilisable.

**Ce que ça empêche :** toute attaque par force brute ou par dictionnaire.
Sans mot de passe valide possible, un attaquant devrait voler une clé privée
protégée par passphrase.

Les comptes sont d'ailleurs créés avec `password: "!"`, c'est-à-dire un champ
de mot de passe verrouillé : même en cas d'erreur de configuration future, il
n'existe aucun mot de passe à deviner.

---

## 2. Root ne se connecte pas

```
PermitRootLogin no
```

**Ce que ça apporte :** la traçabilité. Chacun se connecte avec son compte
nominatif puis passe root via sudo, qui journalise. Sur une connexion root
directe, impossible de savoir lequel des trois a agi.

C'est aussi la cible n°1 des scans automatisés : la quasi-totalité des
tentatives observées dans les journaux visent `root`.

---

## 3. Accès restreint à un groupe

```
AllowGroups adminsys
```

**Ce que ça empêche :** qu'un compte de service créé plus tard — par une
application web, un agent de supervision — puisse ouvrir une session SSH. Même
avec une clé valide, un compte hors du groupe `adminsys` est refusé.

C'est une liste blanche : ce qui n'est pas explicitement autorisé est refusé.

---

## 4. Clés d'hôte régénérées à chaque déploiement

Un clone hérite des clés d'hôte du template. Sans régénération, toutes vos VM
présentent la même empreinte.

**Ce que ça empêche :** une attaque de l'homme du milieu. Le contrôle
d'empreinte SSH repose sur l'unicité de la clé d'hôte : si dix machines
partagent la même, un attaquant qui compromet l'une d'elles peut se faire
passer pour les neuf autres, et le client SSH n'affichera aucun avertissement.

Le rôle affiche l'empreinte ED25519 de chaque machine après régénération —
notez-les dans votre documentation projet, c'est ce qui permet de vérifier une
première connexion.

---

## 5. Algorithmes cryptographiques restreints

```
KexAlgorithms curve25519-sha256,...
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,...
MACs hmac-sha2-512-etm@openssh.com,...
```

OpenSSH accepte par défaut des algorithmes anciens pour rester compatible avec
de vieux clients. On restreint ici aux recommandations ANSSI et Mozilla
« modern ».

**Ce que ça empêche :** une attaque par repli, où l'attaquant force la
négociation vers l'algorithme le plus faible accepté par les deux parties.

Le suffixe `-etm` (*encrypt-then-MAC*) est important : il calcule
l'authentification sur le message chiffré plutôt que sur le clair, ce qui
ferme une classe d'attaques par oracle de padding.

---

## 6. Limitation des tentatives

```
MaxAuthTries 3
LoginGraceTime 30
MaxStartups 10:30:60
```

- `MaxAuthTries 3` — trois essais par connexion, puis coupure.
- `LoginGraceTime 30` — une session non authentifiée est fermée après 30 s.
  Par défaut c'est 120 s, ce qui laisse largement le temps de saturer le
  serveur avec des connexions ouvertes.
- `MaxStartups 10:30:60` — au-delà de 10 connexions en cours
  d'authentification, 30 % sont rejetées au hasard ; à 60, tout est rejeté.
  C'est la protection contre l'épuisement de ressources.

---

## 7. Fonctionnalités désactivées

```
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
GatewayPorts no
```

**Ce que ça empêche :** le rebond. `AllowTcpForwarding` permet d'utiliser une
machine comme relais vers d'autres réseaux — exactement ce qu'un attaquant
cherche après avoir compromis un serveur en DMZ.

`AllowAgentForwarding` mérite une mention particulière : le transfert d'agent
expose votre agent SSH local sur la machine distante. Si celle-ci est
compromise, root peut utiliser votre agent pour se connecter à toutes les
machines auxquelles **vous** avez accès. C'est un vecteur de propagation
redoutable et souvent sous-estimé.

Si vous avez besoin d'un rebond ponctuel, utilisez `ProxyJump` côté client :
la fonctionnalité reste disponible sans exposition côté serveur.

---

## 8. Journalisation détaillée

```
LogLevel VERBOSE
```

En niveau VERBOSE, sshd journalise l'empreinte de la clé utilisée à chaque
connexion :

```
Accepted publickey for yan from 10.0.0.5 port 51234 ssh2: ED25519 SHA256:abc...
```

**Ce que ça apporte :** savoir *quelle clé* a servi, pas seulement quel
compte. En cas d'incident, c'est ce qui permet de déterminer si une clé
précise a été compromise.

C'est aussi ce que fail2ban analyse pour détecter les tentatives.

---

## 9. Fail2ban

Deux prisons sont configurées :

| Prison | Déclenchement | Durée |
|--------|---------------|-------|
| `sshd` | 4 échecs en 10 min | 1 heure |
| `sshd-agressif` | 2 échecs en 5 min | 24 heures |

Les réseaux d'administration sont dans `ignoreip` : une erreur de manipulation
depuis votre poste ne peut pas vous bannir de votre propre infrastructure.
C'est une précaution qui évite bien des mauvaises surprises en TP.

---

## 10. Filtrage réseau

Le pare-feu nftables de chaque VM n'accepte le port 22 que depuis les réseaux
d'administration :

```
ip saddr @admin_networks tcp dport 22 ct state new limit rate 10/minute accept
tcp dport 22 log prefix "NFT-SSH-DROP " drop
```

**Ce que ça apporte :** la défense en profondeur. Le durcissement de sshd
protège la porte ; cette règle décide qui peut seulement s'en approcher. Une
machine compromise en DMZ ne peut pas scanner ni attaquer le SSH des autres
zones — le paquet est jeté avant que sshd ne le voie.

La limitation de débit ajoute une protection contre le balayage : dix
nouvelles connexions par minute suffisent largement à trois administrateurs,
et cassent l'efficacité d'un scan.

---

## 11. Sudo journalisé

```
%adminsys ALL=(ALL:ALL) NOPASSWD:ALL
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output
```

`NOPASSWD` peut surprendre. La logique : les comptes n'ont volontairement pas
de mot de passe, l'authentification est entièrement portée par la clé SSH avec
sa passphrase. Demander un mot de passe inexistant bloquerait simplement
l'administration.

En contrepartie, `log_input, log_output` enregistre l'intégralité des sessions
sudo — commandes tapées et sortie affichée. La traçabilité compense la
suppression du second facteur.

---

## Vérifier que tout est bien appliqué

```bash
ansible-playbook audit-ssh.yml
```

Ou à la main sur une machine :

```bash
sudo sshd -T | grep -E 'passwordauth|permitrootlogin|allowgroups|loglevel'
sudo fail2ban-client status sshd
sudo nft list ruleset
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

---

## Questions probables en soutenance

**« Pourquoi pas de mot de passe du tout ? »**
Parce qu'un mot de passe est devinable et rejouable, une clé ED25519 non. La
passphrase de la clé assure le facteur « ce que je sais », la clé privée le
facteur « ce que je possède ».

**« Que se passe-t-il si un collègue quitte le projet ? »**
On retire son entrée de la liste `admins`, on relance `ssh-config.yml`. Comme
les clés sont déployées en mode exclusif, son `authorized_keys` est réécrit et
son accès disparaît de toutes les machines en une commande.

**« Comment savez-vous que la configuration est réellement appliquée ? »**
`sshd -T` affiche la configuration effective du démon, pas le contenu du
fichier. C'est ce que lit le playbook d'audit, et c'est la différence entre
« c'est écrit dans le fichier » et « c'est actif ».

**« Et si le playbook casse le SSH ? »**
Les directives `validate` empêchent d'écrire un fichier invalide : Ansible
teste la configuration avant de la mettre en place et échoue proprement si
elle est incorrecte. Le fichier d'origine est de plus sauvegardé en
`.orig` et `backup: true` conserve les versions successives.
