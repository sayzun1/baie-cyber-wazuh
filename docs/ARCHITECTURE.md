# Architecture du déploiement

Document de conception du projet : ce qui compose la chaîne, comment elle
s'exécute, et pourquoi ces choix.

---

## 1. Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────────────┐
│                          BAIE RÉSEAU CYBER                           │
│                                                                      │
│   ┌────────────────────────┐                                         │
│   │  Debian « Ansible »    │   Poste de pilotage                     │
│   │                        │   - dépôt Git du projet                 │
│   │  - ansible-core        │   - clés privées des 3 admins           │
│   │  - collections galaxy  │   - vault des secrets                   │
│   └───────────┬────────────┘                                         │
│               │                                                      │
│               │ ① API HTTPS (token)                                  │
│               ▼                                                      │
│   ┌────────────────────────────────────────────────────────┐         │
│   │              HYPERVISEUR PROXMOX VE                     │         │
│   │                                                         │         │
│   │   [ template debian12-cloudinit ]  ──clone──┐           │         │
│   │                                             ▼           │         │
│   │   vmbr1 ──── srv-dns01    srv-web01   (zone LAN)        │         │
│   │   vmbr2 ──── srv-proxy01              (zone DMZ)        │         │
│   │   vmbr3 ──── srv-siem01   srv-backup01 (zone ADMIN)     │         │
│   └────────────────────────────┬────────────────────────────┘         │
│               ▲                │                                      │
│               │ ② SSH (clés)   │                                      │
│               └────────────────┘                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Deux canaux, deux rôles bien séparés :

- **① API Proxmox** — Ansible crée, configure et démarre les VM. Il ne se
  connecte pas en SSH à l'hyperviseur : les modules `proxmox_*` parlent
  directement à l'API REST. D'où le `ansible_connection: local` sur le groupe
  `proxmox` dans l'inventaire.
- **② SSH vers les VM** — une fois la machine démarrée, Ansible s'y connecte
  par clé pour appliquer le socle système et le durcissement.

---

## 2. Découpage en zones

| Zone | Bridge | VLAN | Réseau | Contenu |
|------|--------|------|--------|---------|
| LAN | `vmbr1` | 10 | 192.168.10.0/24 | Services internes (DNS, web interne) |
| DMZ | `vmbr2` | 20 | 172.16.0.0/24 | Services exposés (reverse proxy) |
| ADMIN | `vmbr3` | 30 | 10.0.0.0/24 | Supervision, SIEM, sauvegarde |

Le paramètre `vm_bridge` du playbook est ce qui décide de la zone d'une
machine. C'est volontairement un paramètre de ligne de commande : le
rattachement réseau est la décision la plus structurante d'un déploiement en
environnement cyber, elle doit être explicite à chaque création.

**Point à savoir défendre en soutenance :** l'accès SSH n'est ouvert que
depuis les réseaux listés dans `ssh_allowed_networks`, en pratique le VLAN
d'administration. Une machine compromise en DMZ ne peut donc pas rebondir en
SSH sur les autres VM : le pare-feu nftables de chaque machine refuse la
connexion avant même que sshd ne réponde.

---

## 3. Déroulement d'un déploiement

```
ansible-playbook deploy-vm.yml -e "vm_name=... vm_ip=..."
   │
   ├─ PHASE 1 — sur l'hyperviseur (connexion locale, API)
   │    ├─ Contrôles préalables
   │    │    ├─ vm_name conforme à la RFC 1123
   │    │    ├─ vm_ip au format CIDR valide
   │    │    ├─ vm_bridge au format vmbrN
   │    │    └─ nom pas déjà utilisé sur le noeud
   │    ├─ Attribution du VMID (premier libre entre 100 et 999)
   │    ├─ Lecture des clés publiques de l'équipe
   │    ├─ Clonage du template
   │    ├─ Application CPU / RAM / disque / réseau
   │    ├─ Injection cloud-init : hostname, IP, passerelle, DNS, clés
   │    ├─ Démarrage
   │    └─ Attente que le port 22 réponde
   │
   └─ PHASE 2 — sur la VM (connexion SSH par clé)
        ├─ rôle common
        │    ├─ hostname, /etc/hosts, fuseau horaire
        │    ├─ paquets de base, mises à jour
        │    ├─ NTP (chrony), agent QEMU
        │    └─ unattended-upgrades
        └─ rôle ssh_hardening
             ├─ groupe adminsys + 3 comptes nominatifs
             ├─ clés publiques (mode exclusif)
             ├─ sudo avec journalisation
             ├─ régénération des clés d'hôte
             ├─ sshd_config durci (validé avant application)
             ├─ fail2ban
             ├─ nftables : SSH limité aux réseaux d'admin
             └─ contrôles de conformité
```

---

## 4. Choix techniques et justifications

### Pourquoi un token d'API plutôt que root@pam

Un token est révocable individuellement, associable à un rôle aux privilèges
strictement nécessaires, et ne donne pas accès au shell de l'hyperviseur.
S'il fuite, on le révoque en une commande sans changer le mot de passe root.

### Pourquoi les clés d'hôte sont régénérées

Un clone hérite des clés d'hôte du template. Toutes les VM auraient donc la
même empreinte SSH, ce qui rend le contrôle d'empreinte inopérant : un
attaquant pourrait usurper n'importe laquelle des machines sans que le client
SSH ne signale quoi que ce soit. Le rôle `ssh_hardening` supprime les clés
héritées et en génère de nouvelles au premier passage, avec un fichier
marqueur pour ne le faire qu'une fois.

C'est le détail que la plupart des projets étudiants oublient, et un très bon
point à mentionner en soutenance.

### Pourquoi `validate:` sur le sshd_config

```yaml
validate: "/usr/sbin/sshd -t -f %s"
```

Ansible écrit d'abord un fichier temporaire, lance `sshd -t` dessus, et
n'écrase le fichier en place que si le test passe. Sans cette ligne, une
simple faute de frappe dans un template rend la VM définitivement inaccessible
en SSH — il faut alors passer par la console Proxmox pour réparer.

Le même mécanisme protège les fichiers sudoers (`visudo -cf`) et nftables
(`nft -c -f`).

### Pourquoi `exclusive: true` sur les clés

```yaml
ansible.posix.authorized_key:
  exclusive: true
```

Le fichier `authorized_keys` est réécrit intégralement à chaque passage. Toute
clé ajoutée manuellement ou par un tiers disparaît. C'est ce qui garantit que
la liste `admins` du dépôt Git est bien la seule source de vérité des accès —
autrement dit qu'on peut répondre avec certitude à la question « qui a accès à
cette machine ? ».

### Pourquoi l'idempotence compte

Chaque playbook peut être relancé indéfiniment sans effet de bord : le second
passage ne signale aucun changement. C'est ce qui permet de traiter ces
fichiers comme la description de l'état voulu du parc, et de les rejouer pour
corriger toute dérive de configuration.

Démonstration parlante en soutenance : lancer `ssh-config.yml` deux fois de
suite et montrer le `changed=0` du second passage.

---

## 5. Sécurisation des secrets

| Élément | Où il vit | Protection |
|---------|-----------|------------|
| Token API Proxmox | `inventory/group_vars/vault.yml` | Chiffré par ansible-vault |
| Hash des mots de passe | idem | idem |
| Clés **publiques** | `roles/ssh_hardening/files/keys/` | En clair, sans risque |
| Clés **privées** | Poste de chaque admin, jamais dans le dépôt | Passphrase |
| Mot de passe du vault | `.vault_pass`, exclu par `.gitignore` | Permissions 600 |

Aucun mot de passe en clair n'est nécessaire au fonctionnement : les VM
n'acceptent que l'authentification par clé, et sudo est en NOPASSWD puisque
l'authentification est déjà assurée par la clé SSH.

---

## 6. Limites connues

- Un seul noeud Proxmox est géré. Le passage en cluster demanderait de
  répartir les VM entre noeuds (variable `pve_node` par machine).
- Pas de gestion de snapshots ni de sauvegarde des VM ; c'est le rôle de
  Proxmox Backup Server, hors périmètre de ce projet.
- Le mode preseed nécessite soit une saisie au menu de boot, soit une ISO
  remasterisée. Proxmox n'expose pas d'API pour injecter une ligne de commande
  noyau au démarrage de l'installeur.
- IPv6 non traité : le plan d'adressage du projet est en IPv4 seul.
