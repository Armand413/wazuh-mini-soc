# Inventaire du laboratoire CyberDefender

**Date de relevé :** 2 novembre 2026
**Source :** captures des commandes `cat /etc/os-release`, `free -h`, `df -h` et `ip -br addr` sur chaque machine.

## 1. Hôte

| Élément | Valeur |
|---|---|
| Processeur | Intel Core i7-1185G7 (4 cœurs / 8 threads) |
| RAM | 16 Go (environ 15,4 Go utilisables) |
| Disque | 954 Go au total, environ 409 Go libres |
| Système | Windows 64 bits |
| Hyperviseur | **VirtualBox** (déduit : adresses MAC en `08:00:27`, NAT en `10.0.X.15`, dossier partagé `sf_...`) |

## 2. Machines virtuelles

| Hostname | Système | RAM totale (disponible) | Disque `/` (libre) | Interface NAT | Interface Host-Only |
|---|---|---|---|---|---|
| `kali` | Kali GNU/Linux Rolling 2026.3 | 1,9 Gio (1,1 Gio) | 79 Go (54 Go) | `eth0` : 10.0.2.15/24 | `eth1` : **192.168.77.10/24** |
| `ubuntu` | Ubuntu 26.04.1 LTS « Resolute Raccoon » | 3,3 Gio (2,3 Gio) | 25 Go (15 Go) | `enp0s8` : 10.0.3.15/24 | `enp0s3` : **192.168.77.30/24** |
| `VM1` | Ubuntu 24.04.5 LTS « Noble Numbat » | 3,8 Gio (2,8 Gio) | 25 Go (18 Go) | `enp0s3` : 10.0.2.15/24 | `enp0s8` : **192.168.77.40/24** |
| `VM2` | Ubuntu 24.04.5 LTS (même configuration que VM1, à confirmer) | 3,8 Gio (à confirmer) | 25 Go (à confirmer) | `enp0s3` (à confirmer) | **IP à relever** (différente de .40) |

Notes :

- Les trois Ubuntu n'ont pas de swap (0 B). Kali a 953 Mi de swap.
- Les adresses NAT identiques (10.0.2.15 sur Kali et VM1) sont normales avec VirtualBox : chaque VM a son propre NAT isolé.
- **Attention à l'ordre des cartes** : sur `ubuntu` (26.04), la carte Host-Only est `enp0s3` et le NAT `enp0s8`. C'est l'inverse de VM1. Ne jamais copier un nom d'interface d'une VM à l'autre.
- Kali monte un dossier partagé de l'hôte : `/media/sf_ransomguard` (250 Go, 83 Go utilisés). C'est un accès direct aux fichiers de Windows depuis Kali.

## 3. Réseau

| Réseau | Plage | Usage actuel |
|---|---|---|
| Host-Only VirtualBox | 192.168.77.0/24 | Communication entre les VM et Windows (Kali .10, ubuntu .30, VM1 .40) |
| NAT VirtualBox | 10.0.X.15/24 (propre à chaque VM) | Accès Internet pour les mises à jour |

À relever : l'adresse de Windows sur l'adaptateur Host-Only (commande `ipconfig`, adaptateur « VirtualBox Host-Only Network »). Elle est probablement 192.168.77.1.

## 4. Comparaison avec les besoins de Wazuh (installation tout-en-un)

Repères relevés dans un guide d'installation récent testé sur Ubuntu 26.04 : 4 Go de RAM minimum, 8 Go recommandés, au moins 40 Go de disque. À recouper avec la documentation officielle de Wazuh avant l'installation.

| Machine | RAM actuelle | Disque actuel | Verdict pour le rôle SOC |
|---|---|---|---|
| `ubuntu` (26.04) | 3,3 Gio | 25 Go | Insuffisant, et système non listé comme officiellement pris en charge par l'installateur |
| `VM1` (24.04) | 3,8 Gio | 25 Go | Insuffisant tel quel, mais système officiellement pris en charge : **à agrandir** |
| `VM2` (24.04) | 3,8 Gio | 25 Go | Suffisant pour un capteur / agent (Suricata + agent Wazuh) |

## 5. Rôles retenus

| Rôle | Machine | Hostname cible | Adresse | RAM cible |
|---|---|---|---|---|
| SOC (Wazuh) | VM1 | `soc` | 192.168.77.40 | 6 Go au départ, 8 Go si l'hôte le permet |
| Capteur (Suricata + agent) | VM2 | `sensor` | 192.168.77.50 (à confirmer) | 2 Go |
| Cible / agent secondaire | `ubuntu` 26.04 | `target` | 192.168.77.30 | 2 Go, démarrée ponctuellement |
| Poste d'attaque | `kali` | `kali` | 192.168.77.10 | 2 Go, démarrée pour les tests d'audit |

## 6. Informations manquantes

- [ ] Adresse IP de VM2 sur la carte Host-Only (`ip -br addr` sur VM2)
- [ ] Adresse IP de Windows sur l'adaptateur Host-Only (`ipconfig`)
- [ ] Contenu de `/etc/netplan/*.yaml` sur chaque Ubuntu (IP fixes ou DHCP ?)
- [ ] Existence de snapshots VirtualBox sur chaque VM
- [ ] Confirmation que l'hyperviseur est bien VirtualBox

## 7. Commandes pour compléter l'inventaire

```bash
# Sur chaque VM
cat /etc/os-release
hostnamectl
free -h
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
ip -br addr
ls /etc/netplan/ && cat /etc/netplan/*.yaml
```

```powershell
# Sur Windows
ipconfig
```
