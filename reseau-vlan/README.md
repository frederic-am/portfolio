
# Réseau VLAN segmenté

## Objectif

Refonte du lab pour passer d’un réseau plat à une segmentation logique par rôle, avec routage centralisé, filtrage inter-VLAN et accès d’administration contrôlé.

Objectifs :

* isoler les rôles techniques
* réduire la surface de communication
* centraliser le contrôle réseau
* préparer une base exploitable pour monitoring, backup et déploiement
* aligner l’infrastructure avec une logique de production

---

## Architecture

| VLAN | Rôle            |        Subnet |    Gateway |
| ---- | --------------- | ------------: | ---------: |
| 10   | Proxy / Bastion | 10.35.10.0/24 | 10.35.10.1 |
| 20   | Applications    | 10.35.20.0/24 | 10.35.20.1 |
| 30   | Monitoring      | 10.35.30.0/24 | 10.35.30.1 |
| 40   | Backup          | 10.35.40.0/24 | 10.35.40.1 |

Le routage inter-VLAN est assuré par l’hyperviseur via sous-interfaces VLAN sur `vmbr2`.

---

## Infrastructure

| Host          | Role                            |          IP |
| ------------- | ------------------------------- | ----------: |
| vm-proxy-01   | Reverse proxy (Traefik)         | 10.35.10.10 |
| ct-bastion-01 | Bastion SSH / point de rebond   | 10.35.10.14 |
| vm-app-01     | Applications (Gitea)            | 10.35.20.10 |
| vm-monitor-01 | Monitoring (Prometheus/Grafana) | 10.35.30.10 |
| ct-backup-01  | Sauvegardes                     | 10.35.40.10 |

---

## Routage

Le routage est centralisé sur Proxmox.

Interfaces :

* `vmbr2`
* `vmbr2.10`
* `vmbr2.20`
* `vmbr2.30`
* `vmbr2.40`

Configuration :

* bridge VLAN-aware
* gateway dédiée par VLAN
* forwarding activé au niveau interface (`ip-forward on`)

Le bastion dispose d’une route persistante vers l’ensemble du réseau lab :

```bash
ip route add 10.35.0.0/16 via 10.35.10.1
```

---

## NAT

Sortie Internet centralisée via `vmbr0`.

Règles :

* `10.10.0.0/24`
* `10.34.0.0/24`
* `10.35.0.0/16`

Objectifs :

* accès aux dépôts
* mises à jour système
* accès DNS externe

---

## Filtrage inter-VLAN

Politique restrictive.

Flux autorisés :

| Source  | Destination  |       Port |
| ------- | ------------ | ---------: |
| Bastion | All nodes    |         22 |
| Proxy   | App          |       3000 |
| Monitor | Nodes        |       9100 |
| Backup  | App          |         22 |
| Backup  | App          |        873 |
| Lab     | DNS resolver | 53 TCP/UDP |
| Lab     | Internet     |         80 |
| Lab     | Internet     |        443 |

Politique finale :

* allow explicite
* deny implicite via DROP final

---

## Sécurisation hyperviseur

Accès management conservés hors firewall Proxmox :

| Service        | Port |
| -------------- | ---: |
| SSH            |   22 |
| Web UI Proxmox | 8006 |

Règles intégrées dans `INPUT`.

---

## Automatisation

Déploiement via Ansible.

Rôles :

### `proxmox-network`

Responsabilités :

* déploiement règles iptables
* restauration persistante
* configuration réseau
* NAT
* policy inter-VLAN

### `network-routing`

Responsabilités :

* routes statiques sur bastion

Objectif :

source de vérité unique.

---

## Validation

Tests réalisés :

### Réseau

* ping gateways VLAN
* ping inter-VLAN
* ping bastion → toutes machines

### Accès

* SSH via bastion
* accès proxy → app
* monitoring → nodes

### Système

* DNS externe
* NAT sortant
* persistence iptables après reboot
* persistence routes après reboot
* persistence forwarding après reboot

### Automation

* `ansible all -m ping`

Résultat :

100% success.

---

## Incident rencontré

### 1. `iptables-restore` failure

Symptôme :

échec complet de chargement.

Cause :

absence de newline finale dans le template.

Erreur :

```text
Bad argument `COMMIT'
```

Correction :

ajout du newline EOF.

---

### 2. Confusion sur `ip_forward`

Observation initiale :

```bash
sysctl net.ipv4.ip_forward
= 0
```

Hypothèse :

forwarding non persistant.

Investigation :

* validation sysctl
* validation pve-firewall
* validation networking
* analyse ifupdown2

Résultat :

le forwarding global restait à `0`, mais le forwarding effectif sur interface était bien actif :

```bash
cat /proc/sys/net/ipv4/conf/vmbr2/forwarding
= 1
```

Cause réelle :

`ifupdown2` applique le forwarding au niveau interface (`ip-forward on`), pas nécessairement au niveau global.

Conclusion :

pas de bug, seulement une mauvaise métrique observée.

---

## Compétences démontrées

* segmentation réseau
* VLAN
* routage inter-VLAN
* NAT
* firewalling L3/L4
* bastion SSH
* troubleshooting Linux réseau
* debugging iptables
* compréhension ifupdown2
* automatisation Ansible
* validation post-reboot
* investigation incident réel

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab