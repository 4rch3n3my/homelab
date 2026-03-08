# Projet 00 — Proxmox VE 8.4

Installation et configuration d'un hyperviseur bare metal Proxmox VE 8.4 servant de base à l'ensemble du homelab.

---

## Objectifs

- Installer Proxmox VE sur un serveur physique dédié
- Configurer le réseau en IP fixe sur le LAN
- Préparer le stockage (SSD pour les VMs, HDD pour les ISOs et backups)
- Supprimer les dépôts entreprise et activer les dépôts communautaires (sans abonnement)

---

## Matériel

| Composant | Détail |
|-----------|--------|
| Rôle | Hyperviseur bare metal |
| Connexion réseau | RJ45 (Ethernet filaire) |
| SSD | 232 Go → ZFS → stockage VMs actives |
| HDD | 465 Go → ext4 → ISOs et backups |
| IP assignée | 192.168.1.50/24 |

---

## Installation

### 1. Téléchargement de l'ISO

ISO utilisée : `proxmox-ve_8.4-1.iso`

Téléchargement sur [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)

Création de la clé USB bootable :

```bash
# Depuis Linux
dd if=proxmox-ve_8.4-1.iso of=/dev/sdX bs=4M status=progress
```

### 2. Installation guidée

Paramètres renseignés pendant l'installeur :

| Paramètre | Valeur |
|-----------|--------|
| Filesystem | ZFS (RAID0) sur SSD |
| Hostname | `pve.homelab.local` |
| IP | `192.168.1.50` |
| Netmask | `255.255.255.0` |
| Gateway | `192.168.1.1` |
| DNS | `192.168.1.1` |

> ZFS a été choisi pour ses capacités de snapshots natifs, utile pour les sauvegardes de VMs avant modifications risquées.

### 3. Premier accès

Interface web accessible après reboot :

```
URL  : https://192.168.1.50:8006
User : root
Pass : (mot de passe défini à l'installation)
```

---

## Post-installation

### Désactivation des dépôts Enterprise

Les dépôts enterprise nécessitent un abonnement payant. Sans abonnement, `apt update` remonte une erreur 401.

```bash
# Désactiver le dépôt PVE Enterprise
echo "# désactivé - pas d'abonnement" > /etc/apt/sources.list.d/pve-enterprise.list

# Désactiver le dépôt Ceph Enterprise
echo "# désactivé - pas d'abonnement" > /etc/apt/sources.list.d/ceph.list
```

### Activation des dépôts communautaires

```bash
# Dépôt PVE community (no-subscription)
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list

# Dépôt Ceph community
echo "deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription" \
  >> /etc/apt/sources.list.d/pve-community.list

# Mise à jour du système
apt update && apt upgrade -y
```

---

## Configuration du stockage

### SSD — Stockage ZFS (VMs actives)

Configuré automatiquement par l'installeur Proxmox.

| Paramètre | Valeur |
|-----------|--------|
| Nom | `local` |
| Type | ZFS |
| Chemin | `/dev/sdb` (SSD) |
| Contenu | Disk images, Containers |

### HDD — Stockage ext4 (ISOs et backups)

Le HDD n'est pas géré par l'installeur. Configuration manuelle :

```bash
# Formatage en ext4
mkfs.ext4 /dev/sda

# Création du point de montage
mkdir -p /mnt/hdd

# Montage immédiat
mount /dev/sda /mnt/hdd

# Récupération de l'UUID pour le montage persistant
blkid /dev/sda
# UUID="5214696b-52dc-4d9e-a641-f0aeead9da10"

# Ajout dans /etc/fstab pour montage au boot
echo "UUID=5214696b-52dc-4d9e-a641-f0aeead9da10 /mnt/hdd ext4 defaults 0 2" \
  >> /etc/fstab

# Vérification
mount -a && df -h /mnt/hdd
```

### Ajout du stockage HDD dans l'interface Proxmox

Datacenter → Storage → Add → Directory

| Paramètre | Valeur |
|-----------|--------|
| ID | `hdd-storage` |
| Directory | `/mnt/hdd` |
| Content | ISO image, VZDump backup, Disk image, Snippets |

---

## Architecture finale

```
Proxmox VE 8.4 — 192.168.1.50
│
├── Stockage SSD (ZFS) — local
│   └── VMs actives (disk images)
│
├── Stockage HDD (ext4) — hdd-storage
│   ├── ISOs
│   └── Backups VZDump
│
├── wazuh-server (VM Debian 12) — 192.168.1.25
│   └── Wazuh SIEM 4.11.2 (projet 02)
│
└── [futures VMs]
    ├── Windows Server 2022 (AD/GPO) — projet 03
    └── ...
```

---

## Prochaines étapes

- [x] Déployer wazuh-server (projet 02)
- [ ] Déployer Windows Server 2022 pour Active Directory (projet 03)
- [ ] Configurer les snapshots ZFS automatiques
- [ ] Mettre en place des backups VZDump planifiés vers hdd-storage

---

## Références

- [Documentation Proxmox VE](https://pve.proxmox.com/wiki/Main_Page)
- [Proxmox — No-Subscription Repository](https://pve.proxmox.com/wiki/Package_Repositories#sysadmin_no_subscription_repo)
- [ZFS sur Proxmox](https://pve.proxmox.com/wiki/ZFS_on_Linux)
