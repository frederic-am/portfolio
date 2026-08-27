# Monitoring Prometheus & Grafana

## Présentation

Ce composant met en œuvre la plateforme de supervision de l'infrastructure Linux du laboratoire.

La supervision centralise les métriques provenant :

* des hôtes Linux ;
* des conteneurs Docker ;
* du reverse proxy Traefik ;
* du système de sauvegarde.

Les données sont collectées par **Prometheus** puis exploitées dans **Grafana** sous forme de dashboards et d'indicateurs de supervision.

---

## Architecture

La plateforme repose sur quatre machines :

```text
                    Grafana
                        ▲
                        │
                  Prometheus
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   vm-app-01      vm-proxy-01      ct-backup-01
        │               │               │
 Node Exporter     Node Exporter   Node Exporter
 cAdvisor          cAdvisor        Backup metrics
                        │
                 Traefik Metrics
```

`vm-monitor-01` héberge :

* Prometheus ;
* Grafana.

Les autres machines exposent leurs métriques à Prometheus.

---

## Collecte des métriques

### Node Exporter

**Node Exporter** fournit les métriques système des machines Linux.

Les principales données exploitées sont :

* disponibilité ;
* CPU ;
* mémoire ;
* système de fichiers ;
* réseau.

Prometheus interroge régulièrement les endpoints Node Exporter configurés dans ses `targets`.

---

### cAdvisor

**cAdvisor** fournit les métriques des conteneurs Docker.

Les données utilisées dans Grafana comprennent notamment :

* consommation CPU ;
* consommation mémoire ;
* trafic réseau ;
* nombre de conteneurs.

Cela permet de passer d'une supervision uniquement système à une supervision des services Docker.

---

### Traefik

Traefik expose ses propres métriques Prometheus.

Elles permettent notamment de suivre :

* le débit des requêtes ;
* les codes HTTP ;
* le temps de réponse ;
* les connexions actives ;
* les certificats TLS Let's Encrypt.

Ces métriques sont collectées par Prometheus puis utilisées dans le dashboard `Containers`.

---

## Métriques personnalisées des sauvegardes

Le système de sauvegarde génère des métriques personnalisées via le **Node Exporter Textfile Collector**.

Après chaque exécution de `docker-backup.sh`, le serveur de sauvegarde produit un fichier `.prom` contenant notamment :

```text
backup_status{target="vm-app-01"} 1
backup_last_success{target="vm-app-01"} ...
backup_duration_seconds{target="vm-app-01"} ...
```

En cas d'échec :

```text
backup_status{target="vm-app-01"} 0
backup_last_failure{target="vm-app-01"} ...
```

Les métriques sont générées sur `ct-backup-01`.

---

# Prometheus

## Targets

Prometheus est configuré pour interroger les différents endpoints de métriques de l'infrastructure.

Chaque target correspond à une source de métriques.

Le principe est :

```text
Exporter
   ↓
endpoint /metrics
   ↓
Prometheus scrape
   ↓
base de métriques
```

---

## Scrape

Prometheus interroge périodiquement les exporters configurés.

Le statut d'une target peut être vérifié directement dans Prometheus.

Une target indisponible peut notamment provoquer le déclenchement de l'alerte `VmDown`.

---

# PromQL

Les dashboards et les alertes utilisent des requêtes PromQL.

Exemple pour vérifier l'état d'une machine :

```promql
up{job="node"}
```

Exemple pour vérifier l'état d'une sauvegarde :

```promql
backup_status{target="vm-app-01"}
```

Pour déterminer l'ancienneté de la dernière sauvegarde réussie :

```promql
time() - backup_last_success{target="vm-app-01"}
```

Ces requêtes permettent de transformer les métriques brutes en indicateurs directement exploitables pour l'exploitation.

---

# Grafana

Trois dashboards ont été construits.

## Hosts

Le dashboard `Hosts` fournit une vue globale des machines :

* disponibilité ;
* CPU ;
* mémoire ;
* occupation disque.

Il permet d'identifier rapidement un problème au niveau système.

---

## Containers

Le dashboard `Containers` permet de superviser les services Docker.

Il regroupe notamment :

* nombre de conteneurs par machine ;
* CPU ;
* mémoire ;
* trafic réseau.

Les métriques Traefik y sont également exploitées pour suivre l'activité des services exposés :

* requêtes ;
* codes HTTP ;
* temps de réponse P95 ;
* connexions actives ;
* certificats TLS.

---

## Backups

Le dashboard `Backups` est dédié à la supervision des sauvegardes.

Il exploite les métriques :

```text
backup_status
backup_last_success
backup_duration_seconds
```

Deux notions sont volontairement distinguées.

### Backup availability

Indique qu'une sauvegarde exploitable est encore disponible pour une restauration.

L'indicateur utilise une fenêtre de conservation de **7 jours**.

### Backup freshness

Indique que la dernière sauvegarde respecte l'objectif quotidien.

La fraîcheur est considérée comme correcte lorsque la dernière sauvegarde réussie date de moins de **24 heures**.

Une sauvegarde peut donc être :

```text
AVAILABLE = 1
FRESH     = 0
```

La sauvegarde reste alors disponible pour une restauration, mais l'objectif de sauvegarde quotidienne n'est plus respecté.

---

# Alertes Prometheus

Quatre règles d'alerte sont actuellement utilisées.

| Alerte          | Objectif                                         |
| --------------- | ------------------------------------------------ |
| `VmDown`        | Détecter une machine indisponible                |
| `BackupFailed`  | Détecter un échec de sauvegarde                  |
| `BackupStale`   | Détecter une sauvegarde trop ancienne            |
| `DiskUsageHigh` | Détecter une occupation disque supérieure à 90 % |

Exemple :

```promql
up{job="node"} == 0
```

pour détecter une machine indisponible.

Les alertes sont évaluées par Prometheus et visibles dans son interface.

---

# Déploiement Ansible

La plateforme est déployée et configurée avec Ansible.

Le déploiement prend notamment en charge :

* installation de Prometheus ;
* configuration des targets ;
* installation/configuration de Grafana ;
* exporters ;
* règles d'alerte ;
* configuration des métriques personnalisées.

L'utilisation d'Ansible permet de conserver une configuration reproductible et versionnée.

---

# Validation

La supervision a été validée avec plusieurs scénarios :

* vérification de la collecte des métriques Node Exporter ;
* vérification de la collecte cAdvisor ;
* vérification des métriques Traefik ;
* génération des métriques de sauvegarde ;
* vérification des dashboards Grafana ;
* arrêt d'une machine supervisée ;
* vérification du déclenchement de `VmDown` ;
* simulation d'une sauvegarde ancienne ;
* vérification du comportement de `BackupStale` ;
* vérification de la métrique `backup_status`.

Les dashboards ont également été vérifiés pour s'assurer que les données collectées correspondent aux besoins d'exploitation.

---

# Arborescence

```text
monitoring/
├── README.md
├── prometheus/
├── grafana/
└── images/
    ├── monitoring-architecture.png
    ├── grafana-hosts.png
    ├── grafana-containers.png
    └── grafana-backups.png
```

Les captures et schémas utilisés dans la documentation sont stockés dans le répertoire global `images/` du portfolio lorsque les assets sont communs à plusieurs projets.

---

# Résultat

La plateforme permet de répondre à trois questions opérationnelles :

```text
Machines
→ Les VM/LXC sont-elles disponibles et en bonne santé ?

Containers
→ Les services Docker fonctionnent-ils correctement ?

Backups
→ Les sauvegardes sont-elles disponibles et suffisamment récentes ?
```

Prometheus assure la collecte et l'évaluation des métriques tandis que Grafana fournit leur visualisation et leur exploitation opérationnelle.

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab