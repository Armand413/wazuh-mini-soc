# wazuh-mini-soc

**Laboratoire SOC virtualisé : supervision, détection et analyse d'attaques avec Wazuh**

![Statut](https://img.shields.io/badge/statut-en%20cours-orange)
![Wazuh](https://img.shields.io/badge/Wazuh-4.14-005EB8)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420)
![Kali](https://img.shields.io/badge/Kali-2026.3-557C94)
![VirtualBox](https://img.shields.io/badge/VirtualBox-lab-183A61)

## Résumé

Ce projet reproduit à petite échelle le fonctionnement d'un centre des opérations de sécurité (SOC). Un serveur Wazuh centralise et analyse les journaux de machines supervisées ; des attaques jouées depuis Kali Linux contre mes propres machines servent à vérifier ce que la détection capte réellement.

L'objectif est de boucler le cycle **attaque, journal, détection, alerte, réponse**, puis de **mesurer** la qualité de la détection (taux de détection, faux positifs, temps de détection) plutôt que de se limiter à une démonstration.

**État actuel :** l'infrastructure est déployée et opérationnelle (Wazuh installé, agent enrôlé). Les campagnes d'attaque et les mesures sont en cours.

## Compétences mises en œuvre

| Domaine | Réalisation dans ce projet |
|---|---|
| Virtualisation et réseau | Conception d'un laboratoire sous VirtualBox : réseau Host-Only `192.168.77.0/24` pour le trafic du laboratoire, NAT pour les mises à jour, adressage fixe, validation de la connectivité |
| Administration Linux | Dimensionnement des VM (vCPU, RAM, disque), gestion des services avec `systemd`, sessions `tmux`, lecture des journaux système |
| SIEM | Déploiement de Wazuh en tout-en-un (indexer, manager, dashboard, Filebeat), enrôlement d'un agent, supervision depuis le tableau de bord |
| Diagnostic | Analyse de deux échecs d'installation à partir des journaux (`wazuh-install.log`, `ossec.log`, `journalctl`), démarche par hypothèses et tests |
| Rigueur opérationnelle | Snapshots avant chaque changement, documentation des écarts et des erreurs, périmètre d'attaque strictement limité au laboratoire |
| Détection (en cours) | Attaques SSH depuis Kali, règles personnalisées, cartographie MITRE ATT&CK, mesures |

## Architecture

```mermaid
flowchart LR
    K[kali<br/>192.168.77.10<br/>poste d'attaque] -->|attaques SSH| S[sensor<br/>192.168.77.50<br/>Ubuntu 24.04<br/>agent Wazuh]
    S -->|journaux, port 1514| W[soc<br/>192.168.77.40<br/>Ubuntu 24.04<br/>Wazuh 4.14]
    T[target<br/>192.168.77.30<br/>Ubuntu 26.04] -.->|agent secondaire, prévu| W
```

| Rôle | Hostname | Système | Adresse | Ressources |
|---|---|---|---|---|
| Serveur SOC (Wazuh tout-en-un) | `soc` | Ubuntu 24.04.5 LTS | 192.168.77.40 | 4 vCPU, 8 Go de RAM, 25 Go de disque |
| Machine supervisée (agent) | `sensor` | Ubuntu 24.04.5 LTS | 192.168.77.50 | 2 Go de RAM |
| Cible secondaire | `target` | Ubuntu 26.04.1 LTS | 192.168.77.30 | 2 Go de RAM, allumée ponctuellement |
| Poste d'attaque | `kali` | Kali Linux Rolling 2026.3 | 192.168.77.10 | 2 Go de RAM |

Hôte : PC Windows, Intel Core i7-1185G7, 16 Go de RAM. Le détail est dans [`architecture/inventaire.md`](architecture/inventaire.md).

## Stack technique

| Composant | Rôle |
|---|---|
| Wazuh 4.14 (indexer, manager, dashboard, Filebeat) | SIEM : collecte, analyse, corrélation et visualisation des événements |
| Agent Wazuh | Remontée des journaux de la machine supervisée |
| Ubuntu Server 24.04 LTS | Système du serveur SOC et de la machine supervisée |
| Kali Linux | Génération d'attaques contre le laboratoire |
| VirtualBox | Virtualisation et réseau isolé |
| OpenSSH, tmux, systemd | Services, accès distant et exécution robuste |

## Captures

**Tableau de bord Wazuh**, accessible sur `https://192.168.77.40` :

![Page de connexion du tableau de bord Wazuh](screenshots/wazuh-login.png)

**Agent `sensor` enrôlé** (ID 002, 192.168.77.50, Ubuntu 24.04.5 LTS) à l'état « active » :

![Liste des agents Wazuh avec sensor à l'état active](screenshots/agent-sensor-active.jpeg)

## Déploiement

Sur `soc`, avec les autres VM éteintes (l'hôte est à 16 Go de RAM) :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg ca-certificates tmux
tmux new -s wazuh
curl -s -o wazuh-install.sh https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -v 2>&1 | tee ~/install-output.txt
sudo systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard filebeat
```

Le mot de passe `admin` généré est conservé dans un gestionnaire de mots de passe, jamais dans le dépôt.

Agent sur `sensor` :

```bash
sudo apt install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER="192.168.77.40" apt install -y wazuh-agent
sudo systemctl enable --now wazuh-agent
```

## Retour d'expérience

| Problème | Analyse | Résolution |
|---|---|---|
| Deux installations de Wazuh échouées | Le journal d'installation montre que `wazuh-manager` n'a pas démarré dans le délai imparti (timeout), puis que l'installateur a tout annulé. | Retour à un snapshot sain, passage de 2 à 4 vCPU et à 8 Go de RAM, autres VM éteintes, installation dans `tmux` avec journalisation détaillée. Installation réussie ensuite. La cause exacte n'a pas été isolée ; un démarrage trop lent faute de ressources est l'hypothèse la plus probable. |
| `ping` vers le dépôt Wazuh sans réponse | Le serveur ne répond pas aux pings, ce n'est pas une panne réseau. | Validation par téléchargement (`curl -sI` renvoie `HTTP/2 200`) et par `ping 8.8.8.8`. |
| Services `inactive` après restauration d'un snapshot | `inactive` s'affiche aussi pour un service jamais installé. | Contrôle de l'installation avec `dpkg -l` avant de conclure à un échec. |
| Noms d'interface réseau différents selon les VM | Le Host-Only est `enp0s3` sur une VM et `enp0s8` sur une autre. | Vérification systématique avec `ip -br addr` avant toute modification réseau. |

## Avancement

- [x] Inventaire des machines, choix des rôles, plan d'adressage
- [x] Réseau Host-Only et accès Internet validés
- [x] Wazuh installé, tableau de bord accessible
- [x] Agent enrôlé sur `sensor`, état « active »
- [ ] Aligner les versions du manager et de l'agent
- [ ] Première détection : échecs SSH et force brute depuis Kali
- [ ] Mesures : taux de détection, faux positifs, temps de détection
- [ ] Règles de détection personnalisées, cartographiées sur MITRE ATT&CK
- [ ] Agrandissement du disque de `soc` (25 Go, juste à moyen terme)
- [ ] Intégration de Suricata (IDS réseau)
- [ ] Prototype de réponse automatisée en Python (blocage d'IP avec dry-run et retour arrière)
- [ ] Scénario Windows (Sysmon, Active Directory)

## Résultats

*À compléter après les campagnes d'attaque. Aucune valeur n'est publiée avant d'avoir été mesurée.*

| Scénario | Essais | Détectés | Faux positifs | Temps de détection moyen |
|---|---|---|---|---|
| Échecs SSH, utilisateur inexistant | | | | |
| Échecs SSH, utilisateur existant | | | | |
| Force brute SSH (hydra depuis Kali) | | | | |

Protocole : plusieurs essais par scénario, heure de l'attaque et heure de l'alerte relevées, résultats consignés dans [`reports/`](reports/).

## Sécurité et éthique

- Toutes les attaques visent **uniquement mes propres machines virtuelles**, dans un réseau isolé.
- Le dossier partagé de l'hôte, le presse-papiers partagé et le glisser-déposer sont désactivés sur Kali avant les tests.
- Un snapshot est pris avant chaque changement important.
- Aucun mot de passe, jeton ni donnée personnelle n'est publié dans ce dépôt.

## Structure du dépôt

```
wazuh-mini-soc/
├── README.md
├── architecture/
│   └── inventaire.md
├── installation/
│   └── setup-guide.md
├── screenshots/
│   ├── wazuh-login.png
│   └── agent-sensor-active.jpeg
└── reports/
```

## Auteur

**[Prénom Nom]**, étudiant ingénieur en informatique et télécommunications, École Polytechnique de Ouagadougou (EPO), Burkina Faso.
Objectif : opérations de sécurité (SOC) et conception de solutions de cybersécurité.

[LinkedIn](https://www.linkedin.com/in/votre-profil) · [email@exemple.com](mailto:email@exemple.com)

## Journal de progression

| Date | Avancement |
|---|---|
| 2 nov. 2026 | Inventaire des machines, réseau validé. Première installation de Wazuh : échec (timeout du manager, annulation automatique). |
| 4 nov. 2026 | Deuxième échec, puis ajustement des ressources (4 vCPU, 8 Go) et installation réussie. Agent installé sur `sensor`. |
| 9 nov. 2026 | Agent `sensor` confirmé « active ». Captures ajoutées au dépôt. |

*Prochaine étape : première détection depuis Kali et relevé des temps de détection.*
