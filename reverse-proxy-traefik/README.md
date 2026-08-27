# Reverse Proxy Traefik – Exposition sécurisée de services Docker

## 🎯 Objectif

Mettre en place un reverse proxy centralisé permettant :

- l’exposition sécurisée des services internes
- la terminaison TLS
- le routage HTTP/HTTPS
- l’isolation des services backend
- la standardisation des accès

Infrastructure intégrée à un environnement Linux multi-services administré via Ansible.

---

## 🖥️ Contexte

Infrastructure Linux virtualisée sous Proxmox.

Services hébergés :

- Gitea
- Grafana
- Prometheus
- dashboard Traefik

Les services applicatifs ne doivent pas être exposés directement.

Le reverse proxy sert de point d’entrée unique pour l’infrastructure.

---

## ❗ Problème

Dans une architecture multi-services :

- les applications exposent chacune leurs ports
- les accès HTTPS doivent être gérés individuellement
- les règles de sécurité deviennent difficiles à maintenir
- les services backend sont directement accessibles

Une exposition directe des services augmente :

- la surface d’attaque
- le risque d’erreur de configuration
- la complexité d’administration

---

## 🏗️ Architecture

```text
Internet
   ↓
vm-proxy-01 (Traefik)
   ├── git.farnet.tech → Gitea
   ├── grafana.farnet.tech → Grafana
   ├── prometheus.farnet.tech → Prometheus
   └── traefik.farnet.tech → Dashboard Traefik
```

Architecture :

- Traefik exécuté dans Docker
- routage via providers file
- services backend isolés
- terminaison TLS centralisée

---

## ⚙️ Mise en œuvre

Déploiement automatisé via Ansible.

Rôle utilisé :

```text
roles/traefik
```

Déploiement de :

- configuration Traefik
- docker-compose
- middlewares
- configuration TLS
- secrets
- routage des services

---

## 🔀 Routage des services

Exemple de router Traefik :

```yaml
http:
  routers:
    gitea:
      rule: Host(`git.farnet.tech`)
      entryPoints: websecure
      service: gitea
      tls:
        certResolver: letsencrypt
```

---

## 🔐 HTTPS et TLS

Traefik assure :

- redirection HTTP → HTTPS
- gestion TLS centralisée
- génération automatique des certificats
- configuration TLS commune

Configuration TLS :

```yaml
tls:
  options:
    tls-opts:
      minVersion: VersionTLS12
```

---

## 🛡️ Middlewares de sécurité

Middlewares appliqués :

- headers de sécurité
- rate limiting
- authentification basique

Exemple :

```yaml
security-headers:
  headers:
    contentTypeNosniff: true
    browserXssFilter: true
    forceSTSHeader: true
```

---

## 🔎 Choix techniques

### Reverse proxy centralisé

Traefik centralise :

- le routage
- le TLS
- les règles de sécurité
- l’exposition des services

→ simplification de l’administration.

---

### Providers file

Utilisation d’une configuration déclarative :

- lisible
- versionnable
- maintenable

---

### Isolation des backends

Les services backend :

- ne sont pas exposés publiquement
- restent accessibles uniquement via le proxy

→ réduction de la surface d’attaque.

---

### Déploiement via Ansible

Déploiement reproductible :

- configuration centralisée dans le rôle `traefik`
- standardisation
- limitation des erreurs manuelles

---

## 🧪 Validation

Tests réalisés :

```bash
curl -k https://git.farnet.tech
curl -k https://grafana.farnet.tech
curl -k https://prometheus.farnet.tech
```

Vérifications :

- accès HTTPS fonctionnel
- routage correct des services
- certificats TLS valides
- services backend accessibles uniquement via Traefik

---

## 📊 Exploitation

Logs Traefik activés :

- logs applicatifs
- access logs
- erreurs HTTP 4xx / 5xx

Exemple :

```text
/logs/traefik.log
/logs/access.log
```

---

## ✅ Résultat

Infrastructure web centralisée et sécurisée :

- point d’entrée unique
- exposition contrôlée des services
- HTTPS opérationnel
- routage multi-services
- administration simplifiée
- intégration complète avec l’infrastructure Ansible

---

## 🛠️ Compétences démontrées

- administration Linux
- reverse proxy Traefik
- routage HTTP/HTTPS
- TLS
- Docker
- sécurisation des services
- architecture multi-services
- automatisation Ansible
- exploitation d’infrastructure

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab