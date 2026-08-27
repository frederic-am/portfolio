# Sauvegarde Infrastructure Linux – rsync

Système de sauvegarde Linux centralisé, automatisé avec Ansible et basé sur un modèle **pull**.

Le projet met en œuvre la sauvegarde de services Docker, la protection des bases PostgreSQL, l'historisation par snapshots et la supervision de l'état des sauvegardes.

---

## Objectif

Mettre en place une architecture de sauvegarde permettant :

- la centralisation des sauvegardes ;
- l'automatisation des opérations ;
- la limitation des privilèges ;
- l'historisation et la restauration des données ;
- la restauration d'un service individuel ;
- la supervision de l'état des sauvegardes.

L'ensemble est intégré à une infrastructure Linux administrée avec Ansible.

---

## Contexte

Infrastructure Linux virtualisée sous Proxmox.

Le périmètre de sauvegarde concerne les hôtes Docker du lab.

Exemple de service sauvegardé :

- Gitea ;
- PostgreSQL ;
- Redis.

La supervision de l'infrastructure est assurée séparément par Prometheus et Grafana.

---

## Architecture

Le système repose sur un modèle **pull** :

```text
                    SSH / rsync
┌─────────────────┐                  ┌─────────────────────┐
│   Docker host   │                  │   Backup server     │
│                 │                  │                     │
│   Gitea         │                  │   backup_runner     │
│   PostgreSQL    │ ───────────────► │                     │
│   Redis         │                  │   snapshots         │
│                 │                  │                     │
│   /srv/docker   │                  │   /srv/backups      │
└─────────────────┘                  └─────────────────────┘
````

Le serveur de sauvegarde initie les connexions vers les hôtes Docker.

Les serveurs sources n'ont pas besoin d'initier de connexion vers le serveur de sauvegarde.

---

## Principe de fonctionnement

Pour un hôte Docker disposant d'une base de données :

```text
Docker host
     │
     ├── dump PostgreSQL
     │
     ▼
/srv/docker/backups/
     │
     ▼
rsync /srv/docker/
     │
     ▼
snapshot horodaté
     │
     ▼
validation
     │
     ▼
rotation
     │
     ▼
métriques Prometheus
```

Pour un hôte sans base de données, le dump SQL est simplement ignoré et la sauvegarde de `/srv/docker/` est effectuée normalement.

---

## Mise en œuvre

Le déploiement est automatisé avec Ansible.

Les rôles principaux sont :

* `backup-server`
* `backup-client`

Ils permettent notamment de déployer :

* les comptes dédiés ;
* les clés SSH ;
* les restrictions SSH ;
* les scripts de sauvegarde ;
* la configuration des bases ;
* les exclusions ;
* la planification cron ;
* les mécanismes de restauration ;
* les métriques Prometheus.

Le périmètre du rôle `backup-client` est défini par le groupe Ansible `docker`.

---

## Sécurisation

### Modèle pull

Les sauvegardes sont initiées depuis le serveur de sauvegarde.

Cela permet notamment :

* de ne pas stocker de clé privée de sauvegarde sur les serveurs sources ;
* de centraliser les accès ;
* de limiter les possibilités de déplacement latéral.

### Comptes dédiés

Les opérations utilisent des comptes distincts :

* `backup_runner` sur le serveur de sauvegarde ;
* `backup_ro` sur les hôtes Docker.

### Accès SSH restreint

La clé utilisée par le serveur de sauvegarde est associée à un wrapper SSH.

Le wrapper n'autorise que les opérations nécessaires à la sauvegarde :

* transfert `rsync` ;
* déclenchement du backup de base de données.

Les autres commandes SSH sont refusées.

Les privilèges sudo sont également limités aux commandes nécessaires.

---

## Sauvegarde des données

La sauvegarde porte sur :

```text
/srv/docker/
```

La synchronisation est réalisée avec :

```bash
rsync -aHAX --delete --numeric-ids
```

Les options permettent notamment de préserver :

* les permissions ;
* les propriétaires ;
* les ACL ;
* les attributs étendus ;
* les liens physiques.

Certaines données peuvent être explicitement exclues lorsque leur sauvegarde sous forme de fichiers n'est pas pertinente.

---

## Bases de données

Les bases PostgreSQL sont sauvegardées sous forme de dumps logiques avant la synchronisation.

Pour Gitea :

```text
gitea-db
    │
    ▼
gitea-db.sql
    │
    ▼
/srv/docker/backups/
```

Le dump SQL est ensuite inclus dans le snapshot.

Cette séparation permet d'éviter de dépendre d'une simple copie des fichiers internes de PostgreSQL.

---

## Snapshots

Chaque sauvegarde produit un snapshot horodaté.

Les snapshots utilisent `rsync --link-dest` afin de réutiliser les fichiers inchangés entre deux sauvegardes.

```text
/srv/backups/docker/<host>/
├── docker-YYYY-MM-DD-HHMMSS/
├── docker-YYYY-MM-DD-HHMMSS/
└── current -> docker-YYYY-MM-DD-HHMMSS
```

Le premier snapshot constitue une copie complète.

Les suivants conservent l'historique tout en limitant l'espace consommé par les données inchangées.

---

## Rotation

Les anciens snapshots sont automatiquement supprimés selon une durée de rétention définie dans le script de sauvegarde.

La configuration actuelle du lab conserve les snapshots pendant **15 jours**.

---

## Restauration

La restauration permet de récupérer un service individuel depuis un snapshot.

Le processus restaure :

1. les fichiers du service ;
2. la base PostgreSQL lorsqu'elle est configurée ;
3. les containers Docker.

Une restauration complète de Gitea a été testée sur le lab avec succès.

---

## Supervision

Le résultat des sauvegardes est exposé à Prometheus via le **Node Exporter Textfile Collector**.

Les métriques permettent notamment de suivre :

```text
backup_status
backup_last_success
backup_last_failure
backup_duration_seconds
```

La supervision permet de détecter :

* une sauvegarde en échec ;
* une sauvegarde devenue trop ancienne ;
* une durée de sauvegarde anormale.

Le système de backup ne repose donc pas uniquement sur la présence de fichiers ou la lecture manuelle des logs.

---

## Validation

Le système a été validé sur le lab avec notamment :

* déploiement Ansible ;
* vérification de l'idempotence ;
* sauvegarde de Gitea ;
* génération du dump PostgreSQL ;
* exclusions `rsync` ;
* création et rotation des snapshots ;
* conservation des permissions ;
* restauration complète de Gitea ;
* restauration de PostgreSQL ;
* vérification de la suppression du répertoire temporaire ;
* génération des métriques Prometheus ;
* comportement d'un hôte Docker sans base de données ;
* refus du déploiement lorsque la clé du serveur de sauvegarde est absente.

### Sécurité SSH

Le compte de sauvegarde a été testé afin de vérifier que le wrapper SSH limite effectivement les commandes disponibles.

Tests réalisés :

- [x] commande SSH arbitraire refusée (`Denied`)
- [x] exécution de `db-backup.sh` autorisée
- [x] transfert `rsync` autorisé

---

## Automatisation

Déploiement du système :

```bash
ansible-playbook playbooks/backup.yml
```

Déploiement du serveur de sauvegarde :

```bash
ansible-playbook playbooks/backup.yml --tags backup_server
```

Déploiement des clients Docker :

```bash
ansible-playbook playbooks/backup.yml --tags backup_client
```

Les sauvegardes sont ensuite exécutées automatiquement par cron sous l'utilisateur `backup_runner`.

---

## Compétences démontrées

* Administration Linux
* Stratégie de sauvegarde
* `rsync`
* SSH et restrictions de clés
* Gestion des privilèges Linux / sudo
* PostgreSQL
* Docker
* Ansible
* Cron
* Snapshots et hard links
* Prometheus / Node Exporter
* Diagnostic et validation de restauration
* Automatisation d'exploitation

---

## Documentation technique

La documentation détaillée du fonctionnement du système se trouve dans :

```text
backup/
├── README.md
└── Backup & Restore.md
```

Elle décrit l'architecture interne, les scripts, la configuration, le processus de sauvegarde, la supervision et la procédure de restauration.

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab