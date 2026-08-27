# Automatisation Infrastructure Linux – Déploiement multi-services avec Ansible

## 🎯 Objectif

Automatiser le déploiement et l’administration d’une infrastructure Linux multi-serveurs de manière :

- reproductible
- sécurisée
- maintenable
- standardisée

Infrastructure déployée :

- reverse proxy Traefik
- Gitea
- sauvegarde centralisée
- supervision Prometheus / Grafana
- durcissement SSH
- services Docker

---
## 🖥️ Contexte

Infrastructure Linux virtualisée sous Proxmox avec plusieurs rôles serveur :

- proxy
- applications
- monitoring
- backup
- bastion

L'administration manuelle de plusieurs serveurs entraîne rapidement :

- des écarts de configuration ;
- des opérations répétitives ;
- des difficultés de maintenance ;
- un risque d'erreur lors des évolutions.

Ansible permet de centraliser la configuration et de rendre les déploiements reproductibles.

---

## ❗ Problème

Dans une infrastructure multi-services :

- chaque serveur possède des configurations spécifiques
- les déploiements manuels sont difficilement reproductibles
- les services doivent être sécurisés et supervisés
- les dépendances entre services doivent être maîtrisées

Sans automatisation :

- le risque d’erreur humaine augmente
- les écarts entre environnements deviennent fréquents
- la maintenance devient difficile
- les temps de reconstruction sont élevés

---

## 🏗️ Architecture

Infrastructure organisée autour de plusieurs groupes Ansible :

```text
bastion
proxy
web
app
backup
monitoring
```

Services principaux :

```text
vm-proxy-01
 └── Traefik

vm-app-01
 └── Gitea

vm-monitor-01
 ├── Prometheus
 └── Grafana

ct-backup-01
 └── sauvegarde rsync
```

Déploiement centralisé via :

```text
site.yml
├── bootstrap.yml
├── security.yml
├── docker.yml
├── node_exporter.yml
├── monitoring.yml
├── backup.yml
├── apps.yml
├── proxy.yml
└── proxmox.yml
```

---

## ⚙️ Mise en œuvre

Déploiement entièrement automatisé via Ansible.

Rôles utilisés :

- `common`
- `ssh_hardening`
- `traefik`
- `gitea`
- `backup-server`
- `backup-client`
- `prometheus`
- `grafana`
- `node_exporter`
- `docker_utils`

---

## 🔐 Sécurisation

### SSH

Durcissement SSH automatisé :

- désactivation de l’authentification par mot de passe
- limitation du login root
- validation de la configuration avant application

Validation :

```yaml
validate: 'sshd -t -f %s'
```

---

### Sécurité réseau

Le filtrage réseau est assuré par :

- UFW sur le bastion SSH ;
- iptables sur l'hyperviseur Proxmox pour le routage, le NAT et le filtrage inter-VLAN.

Les règles sont déployées et maintenues avec Ansible.

---

### Reverse proxy

Traefik centralise :

- le routage HTTP/HTTPS
- les certificats TLS
- les middlewares de sécurité
- l’exposition des services

Services exposés :

- Gitea
- Grafana
- Prometheus
- dashboard Traefik

---

## 💾 Sauvegarde

Architecture de sauvegarde automatisée :

- modèle pull via rsync
- snapshots incrémentiels
- dump PostgreSQL
- rotation automatique
- accès SSH restreint

Déploiement automatisé des :

- scripts
- ACL
- clés SSH
- cron jobs
- wrappers de sécurité

---

## 📊 Supervision

Mise en place de :

- Prometheus
- Grafana
- node_exporter

Collecte des métriques système :

- CPU
- mémoire
- espace disque
- état des serveurs

---

## 🔎 Choix techniques

### Infrastructure modulaire

Chaque composant possède son rôle dédié.

Avantages :

- lisibilité
- réutilisation
- maintenance simplifiée

---

### Inventaire

L'infrastructure actuelle utilise un inventaire unique :

```text
inventory/dev
```

Cet inventaire regroupe les différents environnements et rôles du laboratoire.

---
### Déploiement idempotent

Le déploiement peut être rejoué sans modification fonctionnelle inutile.

L'idempotence a été vérifiée par deux exécutions successives du playbook complet :

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/site.yml
```
Les deux exécutions se terminent sans erreur ni hôte inaccessible.

Certaines tâches réseau du rôle Proxmox utilisent command ou shell pour appliquer les configurations et sont donc signalées changed à chaque exécution, même lorsque l'état fonctionnel reste inchangé.

Le mode --check permet de valider une partie du déploiement, mais ne permet pas de valider intégralement les rôles node_exporter et prometheus, qui utilisent des archives téléchargées puis extraites pendant l'exécution réelle.

---

### Validation avant redémarrage

Les configurations critiques sont validées avant application.

Exemples :

```yaml
validate: docker compose -f %s config
validate: 'sshd -t -f %s'
```

→ réduction des interruptions de service.

---

### Gestion des secrets

Secrets chiffrés avec Ansible Vault.

Exemples :

- mots de passe PostgreSQL
- credentials Traefik
- email Let's Encrypt

Fichiers utilisés :

```text
inventory/*/group_vars/*/vault.yml
```

---

## 🧪 Validation

Tests réalisés après déploiement :

```bash
curl -k https://git.farnet.tech
curl -k https://grafana.farnet.tech
curl -k https://prometheus.farnet.tech
```

Vérifications complémentaires :

```bash
docker compose ps
systemctl status prometheus
systemctl status grafana-server
```

Éléments validés :

- services accessibles
- HTTPS opérationnel
- reverse proxy fonctionnel
- monitoring actif
- sauvegardes exécutées
- accès sécurisés

---

## 🔁 Exploitation

L’infrastructure peut être :

- reconstruite via Ansible
- rejouée sans reconfiguration manuelle
- maintenue via rôles séparés
- supervisée centralement

Les opérations courantes sont standardisées :

- déploiement
- mise à jour
- supervision
- sauvegarde
- validation des services

---

## ✅ Résultat

Infrastructure Linux multi-services entièrement automatisée :

- déploiement reproductible
- administration centralisée
- services sécurisés
- supervision opérationnelle
- sauvegarde automatisée
- séparation dev / prod
- maintenance simplifiée

---

## 🛠️ Compétences démontrées

- administration Linux
- Ansible avancé
- automatisation infrastructure
- orchestration multi-services
- Docker
- reverse proxy Traefik
- supervision Prometheus / Grafana
- sauvegarde Linux
- sécurisation SSH
- architecture multi-hôtes
- exploitation d’infrastructure

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab