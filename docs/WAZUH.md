# SIEM Wazuh — déploiement et exploitation

Wazuh 4.14.7 en conteneurs sur une VM Debian 13 (Trixie), déployé par
`deploy-wazuh.yml`.

---

## 1. Ce que fait le playbook

```
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24"
   │
   ├─ Contrôles préalables
   │    ├─ mots de passe Wazuh présents dans le vault
   │    └─ ressources demandées ≥ minimum Wazuh (4 vCPU / 8 Go / 50 Go)
   │
   ├─ PHASE 1 — création de la VM Debian 13 sur Proxmox
   │
   └─ PHASE 2 — configuration
        ├─ rôle common          socle système
        ├─ rôle ssh_hardening   3 comptes admin, sshd durci, fail2ban
        ├─ rôle docker          Docker CE + compose, réglages noyau
        ├─ rôle wazuh_docker    stack Wazuh single-node
        └─ pare-feu SIEM        collecte ouverte, administration restreinte
```

Durée : 20 à 30 minutes, dont ~2 Go d'images à télécharger.

---

## 2. Avant de lancer

### 2.1 Le template Debian 13

Même procédure que pour Debian 12
([`PREPARATION-TEMPLATE.md`](PREPARATION-TEMPLATE.md)), avec l'image Trixie
et un VMID différent :

```bash
cd /var/lib/vz/template/iso/
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2

wget https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum -c SHA512SUMS --ignore-missing

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
qm set $VMID --ciuser sysadm
qm set $VMID --ipconfig0 ip=dhcp
qm template $VMID
```

Le nom `debian13-cloudinit` doit correspondre à `vm_template_name` dans
`deploy-wazuh.yml`.

### 2.2 Les mots de passe

Trois mots de passe à ajouter dans `inventory/group_vars/vault.yml` :

```bash
# Générer trois mots de passe solides
for i in 1 2 3; do tr -dc 'A-Za-z0-9!@#%^*_+=' </dev/urandom | head -c 24; echo; done

ansible-vault edit inventory/group_vars/vault.yml
```

```yaml
vault_wazuh_admin_password:        "..."
vault_wazuh_kibanaserver_password: "..."
vault_wazuh_api_password:          "..."
```

> Contraintes de l'indexer : 8 caractères minimum, au moins une majuscule,
> une minuscule, un chiffre et un caractère spécial.

Le playbook **refuse de démarrer** si ces variables manquent. C'est
volontaire : Wazuh est livré avec `admin`/`SecretPassword` et
`wazuh-wui`/`MyS3cr37P450r.*-`, deux couples publiés dans sa propre
documentation. Sur la machine qui centralise les journaux de toute
l'infrastructure, les laisser en place reviendrait à donner la vue complète
du SI à quiconque atteint le port 443.

---

## 3. Lancer le déploiement

```bash
# Valeurs par défaut : srv-wazuh01, 4 vCPU / 8 Go / 100 Go, vmbr3
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24"

# Machine plus confortable
ansible-playbook deploy-wazuh.yml \
  -e "vm_name=srv-siem01 vm_ip=10.0.0.30/24" \
  -e "vm_cores=6 vm_memory=12288 vm_disk=200"

# Sur une VM Debian 13 qui existe déjà
ansible-playbook deploy-wazuh.yml --limit srv-wazuh01 -e "creer_vm=false"

# Docker seulement, sans déployer Wazuh
ansible-playbook deploy-wazuh.yml -e "vm_ip=10.0.0.30/24" --skip-tags wazuh
```

À la fin, le playbook affiche l'URL et le rappel des identifiants.

---

## 4. Premier accès

Ouvre **https://10.0.0.30** depuis une machine du VLAN d'administration.

Le certificat est auto-signé : le navigateur affiche un avertissement, c'est
attendu. Identifiant `admin`, mot de passe celui du vault.

> Si la page ne répond pas tout de suite, ce n'est pas forcément une panne :
> l'indexer met plusieurs minutes à ouvrir ses index au premier démarrage.
> `docker compose logs -f wazuh.indexer` te dit où il en est.

---

## 5. Enrôler un agent

Sur chaque machine à surveiller — y compris les autres VM de la baie.

### Linux (Debian / Ubuntu)

```bash
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb
sudo WAZUH_MANAGER='10.0.0.30' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

### Windows

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER='10.0.0.30'
net start WazuhSvc
```

### Vérifier côté serveur

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

L'agent doit apparaître en `Active`. S'il reste en `Never connected`,
regarde d'abord le pare-feu : le port 1515 doit être joignable depuis sa
zone.

### Déployer l'agent avec Ansible

Plus élégant que de le faire à la main sur chaque machine, et c'est un bon
point en soutenance :

```bash
ansible vms -b -m shell -a "
  curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb &&
  WAZUH_MANAGER='10.0.0.30' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb &&
  systemctl enable --now wazuh-agent
"
```

---

## 6. Exploitation courante

```bash
# État de la stack
systemctl status wazuh-stack
cd /opt/wazuh/wazuh-docker/single-node && docker compose ps

# Journaux
docker compose logs -f                    # tout
docker compose logs -f wazuh.indexer      # un service

# Redémarrer proprement
systemctl restart wazuh-stack

# Arrêter / relancer
systemctl stop wazuh-stack
systemctl start wazuh-stack

# Consommation des conteneurs
docker stats --no-stream

# Sauvegarde manuelle de la configuration
/usr/local/bin/wazuh-backup.sh
ls -lh /opt/wazuh/backups/
```

---

## 7. Le pare-feu du SIEM

C'est la partie la plus intéressante à expliquer à l'oral, parce qu'elle
répond à une contrainte contradictoire.

Un SIEM doit être joignable **depuis partout** — sinon il ne collecte rien —
tout en restant **pilotable depuis un seul endroit**. Les deux coexistent sur
la même machine :

| Port | Service | Ouvert depuis |
|------|---------|---------------|
| 22 | SSH | VLAN administration uniquement |
| 443 | Dashboard web | VLAN administration uniquement |
| 55000 | API Wazuh | VLAN administration uniquement |
| 1514 | Journaux des agents | LAN + DMZ + admin |
| 1515 | Enrôlement des agents | LAN + DMZ + admin, débit limité |
| 514/udp | Syslog équipements | LAN + DMZ + admin |
| 9200 | Indexer | **Personne** — réseau Docker interne seulement |

Le port 9200 mérite une mention. Il donne un accès complet et non filtré aux
données indexées, c'est-à-dire aux journaux de toute l'infrastructure. Une
règle explicite le refuse et le journalise, même si aucune règle ne
l'autorisait : une règle de refus explicite documente l'intention, là où une
simple absence pourrait passer pour un oubli.

Les réseaux d'agents se règlent dans `deploy-wazuh.yml`, variable
`wazuh_agent_networks`.

---

## 8. Dimensionnement

Les valeurs par défaut (4 vCPU / 8 Go / 100 Go) tiennent une maquette
d'une vingtaine d'agents. Au-delà, l'indexer est le premier à souffrir.

| Agents | vCPU | RAM | Disque | Heap indexer |
|--------|------|-----|--------|--------------|
| < 20 | 4 | 8 Go | 100 Go | 4 Go |
| 20 – 50 | 6 | 12 Go | 200 Go | 6 Go |
| 50 – 100 | 8 | 16 Go | 500 Go | 8 Go |

Le heap se règle avec `wazuh_indexer_heap`. La règle OpenSearch : la moitié
de la RAM, sans jamais dépasser 31 Go — au-delà, la JVM perd la compression
des pointeurs et consomme davantage pour rien.

Le disque est ce qui part le plus vite. Compte grossièrement 500 Mo à 1 Go
par agent et par mois selon ce que tu collectes. Un indexer sans espace passe
ses index en lecture seule, s'arrête de collecter, et le message d'erreur
n'est pas explicite — surveille `df -h`.

---

## 9. Dépannage

### Le dashboard ne répond pas

```bash
cd /opt/wazuh/wazuh-docker/single-node
docker compose ps
docker compose logs wazuh.dashboard | tail -50
```

Le plus souvent, l'indexer n'est pas encore prêt. Le premier démarrage
demande 5 à 10 minutes.

### `max virtual memory areas vm.max_map_count [65530] is too low`

Le réglage noyau n'a pas été appliqué :

```bash
sysctl -n vm.max_map_count     # doit afficher 262144
sysctl -w vm.max_map_count=262144
systemctl restart wazuh-stack
```

Le rôle `docker` le pose dans `/etc/sysctl.d/99-wazuh-docker.conf`. S'il
manque, relance `--tags docker`.

### L'authentification échoue alors que le mot de passe est bon

Deux causes classiques :

1. **Le mot de passe contient un `$`.** Docker Compose l'interprète comme une
   variable. Le rôle double automatiquement les `$` en `$$`, mais si tu as
   modifié le compose à la main, vérifie ce point en premier.
2. **Le hash et le mot de passe ne concordent pas** entre
   `internal_users.yml` et `docker-compose.yml`. Les deux doivent être
   changés ensemble — c'est justement ce que le rôle automatise.

### Un agent reste en « Never connected »

```bash
# Depuis l'agent : le port répond-il ?
nc -zv 10.0.0.30 1515

# Depuis le SIEM : le pare-feu jette-t-il quelque chose ?
journalctl -k --since '10 min ago' | grep NFT
```

Si le réseau de l'agent n'est pas dans `wazuh_agent_networks`, ajoute-le et
relance `--tags firewall`.

### La stack ne redémarre pas après un reboot

```bash
systemctl status wazuh-stack
systemctl enable wazuh-stack
```

### Repartir de zéro

```bash
cd /opt/wazuh/wazuh-docker/single-node
docker compose down -v          # -v supprime aussi les données !
```

Puis relance `ansible-playbook deploy-wazuh.yml --limit srv-wazuh01 -e "creer_vm=false"`.

---

## 10. Questions probables en soutenance

**« Pourquoi Docker plutôt qu'une installation classique ? »**
Trois composants (indexer, serveur, dashboard) avec des dépendances Java et
Node précises. En conteneurs, les versions sont figées et cohérentes entre
elles, le déploiement est reproductible à l'identique, et la mise à jour se
résume à changer un tag d'image. En installation classique, une mise à jour
de la distribution peut casser une dépendance sans prévenir.

**« Pourquoi épingler la version 4.14.7 ? »**
Parce qu'un `latest` qui bouge tout seul, c'est une maquette qui ne se
reproduit pas à l'identique — typiquement le jour de la soutenance. La
version épinglée garantit que ce qui a été testé est ce qui tourne.

**« Que se passe-t-il si le SIEM tombe ? »**
Les agents mettent leurs événements en file d'attente localement et les
renvoient à la reconnexion, dans la limite de leur buffer. Une panne courte
ne perd rien ; une panne longue, si. C'est un argument pour surveiller le
SIEM lui-même — un SIEM qui ne collecte plus ne le signale à personne.

**« Le port 9200 est bloqué même de l'intérieur, pourquoi le déclarer ? »**
Parce que la politique par défaut est déjà `drop` : la règle n'ajoute rien
techniquement. Elle ajoute une intention explicite et un journal. Si
quelqu'un tente de l'atteindre, on le voit dans les logs — et le prochain
administrateur comprend que ce port est fermé volontairement, pas par oubli.

---

## Sources

- [Wazuh — Deployment on Docker](https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html)
- [Wazuh — Changing the default password](https://documentation.wazuh.com/current/deployment-options/docker/changing-default-password.html)
- [Docker Engine — Install on Debian](https://docs.docker.com/engine/install/debian/)
- [Debian Official Cloud Images](https://cloud.debian.org/images/cloud/)
