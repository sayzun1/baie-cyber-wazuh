# Préparation du template Debian cloud-init

Opération **unique**, à faire une seule fois sur l'hyperviseur. Comptez
10 minutes. Une fois le template prêt, chaque VM se crée en une quarantaine
de secondes.

Toutes les commandes se lancent en root sur le shell du noeud Proxmox.

---

## Pourquoi un template plutôt qu'une installation à chaque fois

Installer Debian prend 5 à 15 minutes par machine et implique un installeur
qui doit être piloté. Cloner un disque déjà installé prend quelques secondes,
et cloud-init se charge d'y injecter au premier démarrage tout ce qui rend la
machine unique : nom d'hôte, adresse IP, clés SSH.

C'est la méthode standard en production. Le mode preseed reste disponible dans
ce projet pour les cas où aucun template n'existe.

---

## 1. Télécharger l'image cloud officielle

Debian publie des images disque déjà installées et prêtes pour cloud-init.

```bash
cd /var/lib/vz/template/iso/
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

Vérifiez l'intégrité — sur un projet de sécurité, ne sautez pas cette étape :

```bash
wget https://cloud.debian.org/images/cloud/bookworm/latest/SHA512SUMS
sha512sum -c SHA512SUMS --ignore-missing
```

---

## 2. Enrichir l'image avant de la déployer

`libguestfs-tools` permet de modifier l'image sans la démarrer. On y ajoute
l'agent QEMU (sinon Proxmox ne connaîtra jamais l'IP de la VM) et Python
(sans lui, Ansible ne peut rien faire).

```bash
apt update && apt install -y libguestfs-tools

virt-customize -a debian-12-genericcloud-amd64.qcow2 \
  --install qemu-guest-agent,python3,sudo,chrony,vim,curl \
  --run-command 'systemctl enable qemu-guest-agent' \
  --run-command 'echo "PasswordAuthentication no" > /etc/ssh/sshd_config.d/99-nopass.conf' \
  --truncate /etc/machine-id
```

`--truncate /etc/machine-id` est important : sans cela, toutes les VM clonées
partagent le même identifiant machine, ce qui casse notamment l'attribution
d'adresses par DHCP et la corrélation des journaux.

---

## 3. Créer la VM qui deviendra le template

```bash
VMID=9000

qm create $VMID \
  --name debian12-cloudinit \
  --memory 2048 \
  --cores 2 \
  --cpu host \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26 \
  --scsihw virtio-scsi-single \
  --agent enabled=1 \
  --serial0 socket \
  --vga serial0
```

Le port série est nécessaire pour que la console Proxmox fonctionne avec les
images cloud, qui n'ont pas de console graphique.

---

## 4. Importer le disque

```bash
qm importdisk $VMID debian-12-genericcloud-amd64.qcow2 local-lvm
qm set $VMID --scsi0 local-lvm:vm-$VMID-disk-0,discard=on,ssd=1
qm set $VMID --boot order=scsi0
```

`discard=on` permet au stockage de récupérer l'espace des fichiers supprimés
dans la VM. Sur une baie de maquette avec peu de disque, c'est appréciable.

---

## 5. Ajouter le lecteur cloud-init

C'est ce petit disque virtuel qui transportera hostname, IP et clés SSH à
chaque clonage.

```bash
qm set $VMID --ide2 local-lvm:cloudinit
qm set $VMID --ciuser sysadm
qm set $VMID --ipconfig0 ip=dhcp        # écrasé à chaque clone par Ansible
```

---

## 6. Convertir en template

```bash
qm template $VMID
```

La VM devient un modèle en lecture seule. Elle apparaît en gris dans
l'interface et ne peut plus démarrer — c'est normal.

---

## 7. Contrôler que tout est prêt

```bash
qm config 9000
```

Vous devez voir au minimum :

```
agent: enabled=1
ide2: local-lvm:vm-9000-cloudinit,media=cdrom
name: debian12-cloudinit
scsi0: local-lvm:vm-9000-disk-0,discard=on,ssd=1
scsihw: virtio-scsi-single
template: 1
```

Le nom `debian12-cloudinit` doit correspondre exactement à la variable
`vm_template_name` de `inventory/group_vars/all.yml`.

---

## 8. Premier essai

Depuis la machine Ansible :

```bash
ansible-playbook deploy-vm.yml -e "vm_name=srv-test01 vm_ip=192.168.1.99/24"
```

Puis :

```bash
ssh yan@192.168.1.99
```

Si la connexion aboutit sans mot de passe, la chaîne complète fonctionne.
Supprimez ensuite la VM de test :

```bash
ansible-playbook destroy-vm.yml -e "vm_name=srv-test01"
```

---

## Maintenance du template

Un template vieillit : ses paquets accumulent du retard, et chaque VM créée
part avec ce retard. Rafraîchissez-le tous les deux ou trois mois.

```bash
qm clone 9000 9001 --name template-maj
qm start 9001
# se connecter, faire apt update && apt full-upgrade, puis
# cloud-init clean --logs && truncate -s0 /etc/machine-id && poweroff
qm destroy 9000
qm set 9001 --name debian12-cloudinit
qm template 9001
```

`cloud-init clean` est indispensable avant de refaire un template : sans lui,
cloud-init considère qu'il a déjà tourné et ignorera la configuration des
futurs clones.
