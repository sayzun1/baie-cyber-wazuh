# Procédures d'exploitation

Aide-mémoire pour l'usage courant et les pannes classiques.

---

## Créer une VM

```bash
# Cas courant
ansible-playbook deploy-vm.yml -e "vm_name=srv-web02 vm_ip=192.168.10.21/24"

# Dans une autre zone
ansible-playbook deploy-vm.yml \
  -e "vm_name=srv-proxy02 vm_ip=172.16.0.11/24 vm_gateway=172.16.0.254 vm_bridge=vmbr2"

# Machine costaude (SIEM, IDS)
ansible-playbook deploy-vm.yml \
  -e "vm_name=srv-ids01 vm_ip=10.0.0.50/24 vm_bridge=vmbr3" \
  -e "vm_cores=4 vm_memory=8192 vm_disk=100"

# VMID imposé (utile pour garder une numérotation par zone)
ansible-playbook deploy-vm.yml -e "vm_name=srv-dns02 vm_ip=192.168.10.12/24 vm_id=112"
```

### Options utiles

| Option | Effet |
|--------|-------|
| `--check` | Simulation, aucune modification |
| `--diff` | Affiche les différences ligne à ligne |
| `-v` / `-vvv` | Verbosité croissante pour le débogage |
| `--tags ssh` | N'exécute que la partie SSH |
| `--skip-tags provision` | Saute la création, ne fait que configurer |
| `--limit srv-web01` | Restreint à une machine |
| `--start-at-task "..."` | Reprend à une tâche donnée après un échec |

---

## Ajouter un quatrième administrateur

1. Il génère sa clé :

```bash
ssh-keygen -t ed25519 -a 100 -C "prenom@baie-cyber" -f ~/.ssh/id_ed25519_baie
```

2. Déposer son `.pub` dans `roles/ssh_hardening/files/keys/prenom.pub`

3. Ajouter l'entrée dans `inventory/group_vars/all.yml` :

```yaml
admins:
  # ... les trois existants ...
  - name: "prenom"
    comment: "Prenom Nom - Administrateur"
    key_file: "prenom.pub"
    shell: "/bin/bash"
    sudo: true
```

4. Propager :

```bash
ansible-playbook ssh-config.yml
```

---

## Retirer un administrateur

Supprimez son entrée de la liste `admins`, puis relancez `ssh-config.yml`.
Le mode exclusif réécrit les `authorized_keys`, son accès disparaît partout.

Le compte système reste présent (pour conserver ses fichiers et l'historique
des journaux). Pour le supprimer complètement :

```bash
ansible vms -m user -a "name=ancien state=absent remove=yes" -b
```

---

## Changer une clé compromise

Traitez cela comme un incident, dans cet ordre :

```bash
# 1. Révoquer immédiatement : retirer l'entrée de admins, puis
ansible-playbook ssh-config.yml

# 2. Vérifier que la clé n'est plus nulle part
ansible vms -m shell -a "grep -c 'AAAA<debut-de-la-cle>' /home/*/.ssh/authorized_keys" -b

# 3. Chercher les connexions faites avec cette clé
ansible vms -m shell -a "journalctl -u ssh --since '30 days ago' | grep 'SHA256:<empreinte>'" -b

# 4. Réintégrer la nouvelle clé
```

---

## Vérifier l'état du parc

```bash
# Toutes les machines répondent-elles ?
ansible vms -m ping

# Conformité SSH
ansible-playbook audit-ssh.yml

# Bannissements en cours
ansible vms -m shell -a "fail2ban-client status sshd" -b

# Tentatives d'accès refusées par le pare-feu
ansible vms -m shell -a "journalctl -k --since today | grep NFT-SSH-DROP | tail -20" -b

# Machines en attente de redémarrage
ansible vms -m shell -a "test -f /var/run/reboot-required && echo OUI || echo non" -b
```

---

## Résolution des pannes

### « Failed to connect to the host via ssh »

Vérifiez dans l'ordre :

```bash
ping <ip>                                  # la VM est-elle joignable ?
nc -zv <ip> 22                             # le port répond-il ?
ssh -vvv sysadm@<ip>                       # trace détaillée du client
```

Causes fréquentes :

- La VM n'a pas fini de démarrer — attendez et relancez.
- Mauvais bridge : la VM est sur un réseau qui n'est pas le vôtre. Vérifiez
  avec `qm config <vmid> | grep net0` sur l'hyperviseur.
- Votre poste n'est pas dans `ssh_allowed_networks` : le pare-feu de la VM
  jette vos paquets. Ajoutez votre réseau et relancez `ssh-config.yml` depuis
  une machine autorisée.

### « Host key verification failed »

Attendu si vous aviez déjà connu une machine portant cette IP : les clés
d'hôte ont été régénérées.

```bash
ssh-keygen -R <ip>
```

### La VM démarre mais n'a pas d'adresse IP

cloud-init n'a pas appliqué la configuration. Depuis la console Proxmox :

```bash
cloud-init status --long
journalctl -u cloud-init -b
```

Cause la plus fréquente : le template a été créé sans faire
`cloud-init clean`. cloud-init croit avoir déjà tourné et ignore la
configuration. Reprenez la section « Maintenance du template » de
`PREPARATION-TEMPLATE.md`.

### « Authentication failure » sur l'API Proxmox

```bash
# Tester le token à la main
curl -k -H "Authorization: PVEAPIToken=ansible@pve!automation=<secret>" \
  https://<ip-proxmox>:8006/api2/json/nodes
```

Si cela échoue : le token est mal recopié, ou l'ACL n'a pas été appliquée.
Vérifiez avec `pveum acl list` sur l'hyperviseur.

### « The template ... is not found »

Le nom dans `vm_template_name` ne correspond pas à celui du template sur
Proxmox. Listez-les :

```bash
qm list | grep -i template
```

### Un playbook échoue au milieu

Corrigez la cause puis reprenez là où ça s'est arrêté :

```bash
ansible-playbook deploy-vm.yml -e "..." --start-at-task "SSH | Installation de fail2ban"
```

### Je me suis exclu de la machine

Passez par la console noVNC de Proxmox, connectez-vous en root avec le mot de
passe de secours du vault, puis :

```bash
cp /etc/ssh/sshd_config.orig /etc/ssh/sshd_config
systemctl restart ssh
```

C'est précisément le rôle de la sauvegarde `.orig` créée par le rôle.

---

## Commandes ad-hoc pratiques

```bash
# Inventaire lisible
ansible-inventory --graph

# Voir toutes les variables d'une machine
ansible-inventory --host srv-web01

# Faits système d'une machine
ansible srv-web01 -m setup | less

# Redémarrer une zone entière
ansible dmz -m reboot -b

# Mise à jour de sécurité sur tout le parc, machine par machine
ansible-playbook ssh-config.yml -e "lot=1"

# Chercher un fichier partout
ansible vms -m find -a "paths=/etc patterns=*.conf age=-1d" -b
```

---

## Bonnes pratiques de travail à trois

- **Toujours `--check --diff` avant d'appliquer** sur des machines existantes.
- **Un seul pilote à la fois.** Deux playbooks concurrents sur la même machine
  produisent des résultats imprévisibles.
- **Commitez après chaque changement de variable**, avec un message qui dit
  pourquoi. Le dépôt est la mémoire du projet et la trace de qui a décidé quoi.
- **Ne modifiez jamais une VM à la main.** Toute correction manuelle sera
  écrasée au prochain passage du playbook — et pire, elle sera invisible pour
  vos collègues. Si quelque chose manque, ça se corrige dans le rôle.
