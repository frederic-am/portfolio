# Sécurisation de l'infrastructure Linux

Conception d'un laboratoire d'administration Linux sécurisé reposant sur un bastion SSH, une segmentation réseau, un reverse proxy, une supervision centralisée et une automatisation complète avec Ansible.

---

# Objectif

Ce projet présente l'architecture de sécurité de mon laboratoire Linux.

L'objectif est de limiter la surface d'exposition des services tout en conservant une infrastructure simple à administrer, évolutive et facilement exploitable.

Les principes retenus sont les suivants :

- administration centralisée ;
- exposition minimale des services ;
- segmentation réseau ;
- filtrage explicite des communications ;
- supervision continue ;
- déploiement automatisé avec Ansible.

---

# Vue d'ensemble de l'infrastructure

> Architecture du laboratoire d'administration Linux mettant en œuvre un bastion SSH, un reverse proxy, des sauvegardes automatisées et une supervision centralisée.

![Infrastructure](../images/infrastructure.png)

Le laboratoire est constitué de plusieurs machines virtuelles spécialisées.

| Machine | Rôle |
|----------|------|
| vm-proxy-01 | Reverse Proxy Traefik |
| ct-bastion-01 | Bastion SSH |
| vm-app-01 | Hébergement des applications |
| ct-backup-01 | Sauvegardes |
| vm-monitor-01 | Supervision |

L'ensemble est hébergé sur Proxmox VE qui assure la virtualisation, le routage inter-VLAN, le NAT ainsi que le filtrage réseau via iptables.

---

# Architecture retenue

## Administration via un bastion

Toutes les opérations d'administration transitent par un bastion SSH dédié.

Ce choix centralise les accès d'administration, réduit la surface d'exposition des serveurs et permet d'appliquer les mécanismes de protection des connexions SSH, notamment Fail2ban.

---

## Reverse Proxy

L'ensemble des applications est publié exclusivement au travers de Traefik.

Cette approche centralise les points d'entrée HTTP/HTTPS ainsi que la gestion des certificats TLS tout en évitant d'exposer directement les applications internes.

L'ajout d'un nouveau service consiste uniquement à déclarer une nouvelle route dans Traefik, sans modifier l'architecture réseau.

---

## Segmentation réseau

Les fonctions du laboratoire sont réparties sur plusieurs VLAN dédiés :

- administration ;
- applications ;
- supervision ;
- sauvegardes.

Cette séparation limite les communications entre services, facilite leur évolution indépendante et permet d'appliquer des règles de filtrage adaptées à chaque zone.

---

## Filtrage réseau

Le laboratoire est volontairement cloisonné afin que chaque machine ne puisse communiquer qu'avec les services dont elle dépend réellement.

Le routage inter-VLAN est assuré par Proxmox tandis qu'iptables contrôle les flux autorisés entre les différentes zones de l'infrastructure.

Cette approche facilite le diagnostic tout en limitant les communications inutiles.

---

# Mesures de sécurité

L'architecture repose sur plusieurs choix visant à limiter la surface d'attaque :

- administration exclusivement via le bastion SSH ;
- publication des applications uniquement par Traefik ;
- segmentation des services sur plusieurs VLAN ;
- filtrage explicite des communications inter-VLAN ;
- authentification SSH exclusivement par clé publique afin d'éliminer les attaques par mot de passe ;
- surveillance des tentatives de connexion et bannissement automatique des adresses IP malveillantes avec Fail2ban.
- supervision continue des composants critiques.

---

# Exploitation et diagnostics

Cette infrastructure évolue régulièrement au fil des nouveaux projets.

Chaque évolution conduit à adapter les règles de filtrage, les flux réseau ou la supervision tout en conservant une architecture cohérente.

Par exemple, lors de l'intégration des métriques Traefik dans Prometheus, celles-ci restaient inaccessibles malgré une configuration correcte du proxy.

Le diagnostic a consisté à vérifier successivement la configuration de Traefik, l'exposition des métriques, la connectivité réseau puis les règles iptables de Proxmox.

L'analyse a permis d'identifier une règle FORWARD manquante entre le serveur de supervision et le proxy.

Seul le flux nécessaire a été ajouté afin de conserver le principe d'exposition minimale de l'infrastructure.

Ce type d'intervention illustre l'importance du diagnostic réseau et de la maîtrise des flux dans l'exploitation quotidienne d'une infrastructure Linux.

---

# Automatisation

L'ensemble de l'infrastructure est déployé et maintenu avec Ansible.

Les playbooks permettent notamment de gérer :

- la configuration des serveurs ;
- les accès SSH ;
- les règles iptables ;
- Traefik ;
- la supervision ;
- les sauvegardes ;
- les applications.

Cette approche garantit des déploiements reproductibles, simplifie les évolutions de l'infrastructure et limite les écarts de configuration.

---

# Compétences démontrées

- Administration Linux
- Conception d'architecture sécurisée
- Segmentation réseau
- Routage inter-VLAN
- Filtrage réseau avec iptables
- Administration SSH
- Reverse Proxy Traefik
- Gestion TLS / Let's Encrypt
- Diagnostic réseau
- Analyse des flux
- Supervision Prometheus / Grafana
- Automatisation Ansible

---

# Technologies utilisées

- Debian
- Proxmox VE
- Ansible
- Docker
- Traefik
- Prometheus
- Grafana
- Node Exporter
- SSH
- Fail2ban
- iptables
- Let's Encrypt
---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab