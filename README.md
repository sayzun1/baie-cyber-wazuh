# SIEM Wazuh automatisé — Debian 13 + Docker + Ansible

Déploiement automatisé d'un SIEM **Wazuh 4.14.7** en conteneurs, sur une VM
**Debian 13 (Trixie)** créée à la volée sur un hyperviseur **Proxmox VE**.

Une commande, et vous obtenez une machine durcie, avec Docker et la stack
Wazuh opérationnelle :

```bash
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24"
```

Projet réalisé dans le cadre d'une licence Administrateur Système, Réseau et
Sécurité — volet supervision d'une baie réseau cyber.

> **Dépôt lié.** Ce dépôt est autonome : il embarque tout ce qu'il faut pour
> tourner seul. Le projet complet de la baie (déploiement générique des VM,
> maquette multi-zones, audit du parc) vit dans
> [baie-cyber-ansible](https://github.com/sayzun1/baie-cyber-ansible).
> Les rôles `common`, `ssh_hardening`, `vm_provision` et `vm_preseed` sont
> partagés entre les deux : une correction sur l'un doit être reportée sur
> l'autre.

---

## Sommaire

- [Ce que fait le projet](#ce-que-fait-le-projet)
- [Architecture](#architecture)
- [Installation](#installation)
- [Utilisation](#utilisation)
- [Sécurité](#sécurité)
- [Documentation](#documentation)

---

## Ce que fait le projet

| Étape | Contenu |
|-------|---------|
| **Création de la VM** | Clone d'un template Debian 13 cloud-init sur Proxmox — nom, IP, bridge, VLAN et ressources en paramètres |
| **Socle système** | Paquets de base, fuseau horaire, NTP, agent QEMU, mises à jour de sécurité automatiques |
| **Accès SSH** | Comptes nominatifs avec clés, sudo journalisé, sshd durci, clés d'hôte régénérées, fail2ban |
| **Docker** | Docker CE et le plugin compose depuis le dépôt officiel, rotation des logs, réglages noyau requis par l'indexer |
| **Wazuh** | Stack single-node 4.14.7, certificats générés, mots de passe par défaut remplacés, démarrage automatique au boot |
| **Pare-feu** | Collecte ouverte aux zones surveillées, administration réservée au VLAN d'admin |
| **Sauvegarde** | Archive quotidienne de la configuration, des règles et des décodeurs, avec rotation |

Durée d'un déploiement complet : 20 à 30 minutes, dont environ 2 Go d'images
Docker à télécharger.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌────────────────────┐                                         │
│   │ Debian « Ansible » │  ── API HTTPS ──▶  ┌─────────────────┐   │
│   │                    │                    │  PROXMOX VE     │   │
│   │  - playbooks       │                    │                 │   │
│   │  - vault chiffré   │  ◀── SSH (clés) ──│  template       │   │
│   └────────────────────┘                    │  debian13       │   │
│                                             └────────┬────────┘   │
│                                                      │ clone      │
│                                                      ▼            │
│                              ┌─────────────────────────────────┐  │
│                              │  srv-wazuh01  (Debian 13)       │  │
│                              │                                 │  │
│                              │   Docker                        │  │
│                              │   ├── wazuh.indexer   (données) │  │
│                              │   ├── wazuh.manager   (analyse) │  │
│                              │   └── wazuh.dashboard (web)     │  │
│                              └─────────────────────────────────┘  │
│                                      ▲    ▲    ▲                  │
│                    agents ───────────┘    │    └──── syslog       │
│                    (LAN, DMZ, admin)      │         (équipements) │
│                                    administrateurs                │
│                                    (VLAN admin only)              │
└──────────────────────────────────────────────────────────────────┘
```

Les trois conteneurs se répartissent le travail : l'**indexer** stocke et
indexe les événements, le **manager** les analyse et déclenche les alertes,
le **dashboard** les présente.

---

## Installation

### 1. Machine Ansible

```bash
sudo apt update && sudo apt install -y ansible git
git clone https://github.com/sayzun1/baie-cyber-wazuh.git
cd baie-cyber-wazuh
ansible-galaxy collection install -r requirements.yml
```

### 2. Accès API Proxmox

Sur l'hyperviseur, en root :

```bash
pveum user add ansible@pve
pveum role add AnsibleProv -privs "VM.Allocate VM.Clone VM.Config.CDROM \
  VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType \
  VM.Config.Memory VM.Config.Network VM.Config.Options VM.Monitor \
  VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit \
  Sys.Audit SDN.Use"
pveum aclmod / -user ansible@pve -role AnsibleProv
pveum user token add ansible@pve automation --privsep 0
```

Le secret n'est affiché **qu'une seule fois** — copiez-le immédiatement.

### 3. Template Debian 13

```bash
cd /var/lib/vz/template/iso/
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2

apt install -y libguestfs-tools
virt-customize -a debian-13-genericcloud-amd64.qcow2 \
  --install qemu-guest-agent,python3,sudo,chrony,vim,curl,ca-certificates \
  --run-command 'systemctl enable qemu-guest-agent' \
  --truncate /etc/machine-id

VMID=9001
qm create $VMID --name debian13-cloudinit --memory 2048 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr0 --ostype l26 --scsihw virtio-scsi-single \
  --agent enabled=1 --serial0 socket --vga serial0
qm importdisk $VMID debian-13-genericcloud-amd64.qcow2 local-lvm
qm set $VMID --scsi0 local-lvm:vm-$VMID-disk-0,discard=on,ssd=1
qm set $VMID --boot order=scsi0
qm set $VMID --ide2 local-lvm:cloudinit
qm set $VMID --ciuser sysadm --ipconfig0 ip=dhcp
qm template $VMID
```

Procédure détaillée : [`docs/PREPARATION-TEMPLATE.md`](docs/PREPARATION-TEMPLATE.md).

### 4. Clés SSH de l'équipe

Les trois clés de l'équipe sont **déjà intégrées** au projet :

| Fichier | Compte Linux | Type | Empreinte SHA256 |
|---------|--------------|------|------------------|
| `yan.pub` | `yan` | RSA 2048 | `p6LiwCFtVOhl9DeOXUn8sSKbTvZysPjFh17MJy7FJaI` |
| `hippo.pub` | `hippo` | RSA 2048 | `7T6y/tOZTFj8eclJJ665o09ML3B/nZd3xULoHDUcS3g` |
| `eliaz.pub` | `eliaz` | RSA 2048 | `cGijZyMyhGo4aZilEGi/Q8CgueHfmzt2eQMzCncTmVs` |

Pour ajouter un administrateur : il génère sa paire,

```bash
ssh-keygen -t ed25519 -a 100 -C "prenom@baie-cyber" -f ~/.ssh/id_ed25519_baie
```

son fichier `.pub` va dans `roles/ssh_hardening/files/keys/`, et une entrée
est ajoutée à la liste `admins` de `inventory/group_vars/all.yml`, qui fait
la correspondance compte ↔ fichier. Détails dans
[`roles/ssh_hardening/files/keys/README.md`](roles/ssh_hardening/files/keys/README.md).

### 5. Secrets

```bash
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
vim inventory/group_vars/vault.yml
ansible-vault encrypt inventory/group_vars/vault.yml
echo "MotDePasseVault" > .vault_pass && chmod 600 .vault_pass
```

Le vault doit contenir le token Proxmox et **les trois mots de passe
Wazuh** :

```yaml
vault_wazuh_admin_password:        "..."
vault_wazuh_kibanaserver_password: "..."
vault_wazuh_api_password:          "..."
```

Générer des mots de passe conformes :

```bash
for i in 1 2 3; do tr -dc 'A-Za-z0-9!@#%^*_+=' </dev/urandom | head -c 24; echo; done
```

### 6. Variables

Ajustez `inventory/group_vars/all.yml` : IP de l'hyperviseur, nom du noeud,
stockage, plan d'adressage, et surtout `ssh_allowed_networks` — les réseaux
depuis lesquels vous administrez.

---

## Utilisation

```bash
# Déploiement standard : 4 vCPU / 8 Go / 100 Go sur vmbr3
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24"

# Machine plus confortable
ansible-playbook deploy-wazuh.yml \
  -e "vm_name=srv-siem01 vm_ip=10.0.0.30/24" \
  -e "vm_cores=6 vm_memory=12288 vm_disk=200"

# Sur une VM Debian 13 déjà existante
ansible-playbook deploy-wazuh.yml --limit srv-wazuh01 -e "creer_vm=false"

# Docker seulement, sans déployer Wazuh
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24" --skip-tags wazuh

# Simulation, aucune modification
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24" --check
```

### Paramètres

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| `vm_ip` | — | **obligatoire** — IP en CIDR |
| `vm_name` | `srv-wazuh01` | Nom de la machine |
| `vm_bridge` | `vmbr3` | Bridge Proxmox |
| `vm_vlan` | *(vide)* | Tag VLAN |
| `vm_cores` | `4` | vCPU (minimum Wazuh : 4) |
| `vm_memory` | `8192` | RAM en Mo (minimum : 8192) |
| `vm_disk` | `100` | Disque en Go (minimum : 50) |
| `wazuh_version` | `4.14.7` | Version de la stack |
| `creer_vm` | `true` | `false` pour cibler une VM existante |

### Après le déploiement

Interface web sur `https://<ip>`, identifiant `admin`, mot de passe celui du
vault. Le certificat est auto-signé — l'avertissement du navigateur est
attendu.

```bash
systemctl status wazuh-stack       # état de la stack
systemctl restart wazuh-stack      # redémarrage complet
cd /opt/wazuh/wazuh-docker/single-node && docker compose logs -f
```

Enrôler un agent sur une machine à surveiller :

```bash
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb
sudo WAZUH_MANAGER='10.0.0.30' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
sudo systemctl enable --now wazuh-agent
```

---

## Sécurité

Trois points structurent la sécurité de ce déploiement.

**Les mots de passe par défaut sont bloquants.** Wazuh est livré avec
`admin`/`SecretPassword` et `wazuh-wui`/`MyS3cr37P450r.*-`, deux couples
publiés dans sa propre documentation. Sur la machine qui centralise les
journaux de toute l'infrastructure, les laisser reviendrait à offrir la vue
complète du SI à quiconque atteint le port 443. Le playbook refuse donc de
démarrer si les trois mots de passe ne sont pas dans le vault — et la
vérification a lieu *avant* la création de la VM, pour ne pas provisionner
100 Go et échouer vingt minutes plus tard.

**Le pare-feu distingue collecte et administration.** Un SIEM doit être
joignable depuis partout, sinon il ne collecte rien ; il ne doit être
pilotable que depuis un seul endroit.

| Port | Service | Ouvert depuis |
|------|---------|---------------|
| 22 | SSH | VLAN administration |
| 443 | Dashboard web | VLAN administration |
| 55000 | API Wazuh | VLAN administration |
| 1514 | Journaux des agents | LAN + DMZ + admin |
| 1515 | Enrôlement | LAN + DMZ + admin, débit limité |
| 514/udp | Syslog équipements | LAN + DMZ + admin |
| 9200 | Indexer | **personne** — réseau Docker interne |

**Les clés d'hôte SSH sont régénérées après le clonage.** Un clone hérite des
clés du template : sans régénération, toutes les VM du parc présentent la
même empreinte, et le contrôle d'empreinte SSH ne protège plus de rien.

Le détail de chaque directive est dans
[`docs/SECURITE-SSH.md`](docs/SECURITE-SSH.md).

---

## Arborescence

```
.
├── deploy-wazuh.yml             ★ Le playbook du SIEM
├── deploy-vm.yml                Créer une VM Linux générique
├── ssh-config.yml               Réappliquer la conf SSH
├── audit-ssh.yml                Auditer la conformité SSH
├── destroy-vm.yml               Supprimer une VM
│
├── ansible.cfg
├── requirements.yml             Collections Ansible
│
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml              ★ Variables du projet
│       └── vault.yml.example    Modèle des secrets
│
├── templates/
│   └── nftables-wazuh.conf.j2   Pare-feu du SIEM
│
├── roles/
│   ├── vm_provision/            Clone du template cloud-init
│   ├── vm_preseed/              Installation Debian automatisée
│   ├── common/                  Socle système
│   ├── ssh_hardening/           Comptes, clés, durcissement
│   ├── docker/                  ★ Docker CE + réglages noyau
│   └── wazuh_docker/            ★ Stack Wazuh single-node
│
└── docs/
    ├── WAZUH.md                 ★ Le SIEM de A à Z
    ├── ARCHITECTURE.md
    ├── PREPARATION-TEMPLATE.md
    ├── SECURITE-SSH.md
    └── UTILISATION.md
```

---

## Documentation

- [`docs/WAZUH.md`](docs/WAZUH.md) — template Debian 13, mots de passe,
  enrôlement des agents, pare-feu, dimensionnement, dépannage
- [`docs/SECURITE-SSH.md`](docs/SECURITE-SSH.md) — chaque directive de
  durcissement expliquée
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — choix techniques
- [`docs/PREPARATION-TEMPLATE.md`](docs/PREPARATION-TEMPLATE.md) — le template
- [`docs/UTILISATION.md`](docs/UTILISATION.md) — exploitation et dépannage

---

## Ne jamais committer

Le `.gitignore` les exclut, mais vérifiez avant chaque push :

- `.vault_pass`
- `inventory/group_vars/vault.yml` non chiffré
- toute clé **privée**

```bash
git status
head -1 inventory/group_vars/vault.yml    # doit dire $ANSIBLE_VAULT
```

---

## Sources

- [Wazuh — Deployment on Docker](https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html)
- [Docker Engine — Install on Debian](https://docs.docker.com/engine/install/debian/)
- [Debian Official Cloud Images](https://cloud.debian.org/images/cloud/)
- [Proxmox VE — Documentation](https://pve.proxmox.com/pve-docs/)
