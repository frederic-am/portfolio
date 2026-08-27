# Backup & Restore

Documentation technique du système de sauvegarde et de restauration du lab.

---

## 1. Architecture

Le système repose sur deux rôles Ansible :

* `backup-server` : déployé sur `ct-backup-01`
* `backup-client` : déployé sur les hôtes du groupe `docker`

Architecture actuelle :

```text
                    SSH / rsync
┌─────────────────┐                  ┌─────────────────────┐
│   Docker host   │                  │   ct-backup-01      │
│                 │                  │                     │
│ /srv/docker/    │ ──────────────► │ backup_runner       │
│                 │                  │                     │
│ backup_ro       │                  │ /srv/backups/docker │
└─────────────────┘                  └─────────────────────┘
```

Le serveur de sauvegarde initie les opérations.

---

# 2. Configuration Ansible

La configuration spécifique de `vm-app-01` est définie dans :

```text
inventory/lab/host_vars/vm-app-01.yml
```

Configuration actuelle :

```yaml
backup_db_services:
  - container: gitea-db
    type: postgres
    database: gitea
    user: gitea

backup_excludes:
  - gitea/postgres/
  - gitea/data/tmp/

backup_restore_services:
  - service: gitea
    db_type: postgres
    compose_service: db
    database: gitea
    user: gitea
```

Le rôle `backup-client` est appliqué au groupe :

```text
docker
```

Le rôle `backup-server` est appliqué au serveur :

```text
ct-backup-01
```

---

# 3. Fichiers générés

## Sur `vm-app-01`

Le rôle `backup-client` génère notamment :

```text
/etc/backup/db-backup.conf
/etc/backup/restore.conf
/usr/local/sbin/db-backup.sh
/usr/local/sbin/docker-restore.sh
/usr/local/sbin/rrsync-wrapper.sh
```

### `db-backup.conf`

Configuration des bases à sauvegarder :

```text
# container;type;database;user;password_env
gitea-db;postgres;gitea;gitea;
```

### `restore.conf`

Configuration nécessaire à la restauration :

```text
# service;db_type;compose_service;database;user
gitea;postgres;db;gitea;gitea
```

---

## Sur `ct-backup-01`

Le rôle `backup-server` génère notamment :

```text
/usr/local/sbin/docker-backup.sh
/usr/local/sbin/restore.sh
```

Les fichiers d'exclusion sont générés dans :

```text
/etc/backup/
```

Exemple :

```text
/etc/backup/vm-app-01.txt
```

---

# 4. Sauvegarde

## Lancement automatique

Les sauvegardes sont exécutées par cron sous l'utilisateur :

```text
backup_runner
```

Vérification :

```bash
sudo crontab -u backup_runner -l
```

Le cron appelle :

```bash
/usr/local/sbin/docker-backup.sh vm-app-01
```

---

## Lancement manuel

Depuis `ct-backup-01` :

```bash
sudo -u backup_runner \
    /usr/local/sbin/docker-backup.sh vm-app-01
```

Résultat attendu :

```text
[OK] Backup completed: vm-app-01
```

---

# 5. Déroulement d'un backup

Le script `docker-backup.sh` effectue les opérations suivantes :

```text
1. Acquisition du verrou
2. Vérification de la configuration DB
3. Dump de la base si nécessaire
4. Synchronisation de /srv/docker
5. Création du snapshot
6. Validation
7. Mise à jour de current
8. Rotation
9. Mise à jour du statut
10. Génération des métriques Prometheus
```

---

## 5.1 Verrouillage

Un verrou `flock` empêche plusieurs sauvegardes simultanées du même hôte :

```text
/run/lock/docker-backup-<host>.lock
```

---

## 5.2 Base de données configurée

Pour Gitea :

```text
gitea-db
    │
    ▼
db-backup.sh
    │
    ▼
gitea-db.sql
```

Le dump est créé sur l'hôte Docker dans :

```text
/srv/docker/backups/
```

Il est ensuite inclus dans la synchronisation de `/srv/docker/`.

Le log indique :

```text
[INFO] Running database backup check
[INFO] Backup gitea-db (postgres)
```

---

## 5.3 Hôte sans base

Si aucune base n'est configurée :

```text
[INFO] Running database backup check
[INFO] No database configured
```

Le processus continue normalement.

La totalité de :

```text
/srv/docker/
```

est alors sauvegardée par `rsync`.

L'absence de base ne provoque donc pas l'échec du backup.

---

# 6. Synchronisation

La synchronisation utilise :

```bash
rsync \
    -aHAX \
    --delete \
    --numeric-ids \
    --link-dest=...
```

Les exclusions sont fournies au `rsync` par :

```text
/etc/backup/<host>.txt
```

Pour `vm-app-01` :

```text
gitea/postgres/
gitea/data/tmp/
```

---

# 7. Snapshots

Les sauvegardes sont stockées dans :

```text
/srv/backups/docker/<host>/
```

Exemple :

```text
/srv/backups/docker/vm-app-01/
├── current -> docker-2026-07-29-130931
├── docker-2026-07-29-111549/
├── docker-2026-07-29-125407/
└── docker-2026-07-29-130931/
```

Chaque exécution réussie crée un nouveau snapshot.

---

## `current`

Le lien :

```text
current
```

pointe vers le dernier snapshot valide.

Il permet notamment à la sauvegarde suivante d'utiliser :

```text
--link-dest=current
```

---

# 8. Rotation

La rotation est effectuée après validation du snapshot.

La durée actuelle de conservation est :

```text
15 jours
```

Les snapshots dépassant cette durée sont supprimés.

Le snapshot courant n'est pas supprimé.

---

# 9. Validation

Après le `rsync`, le script vérifie notamment :

```text
- présence du répertoire du snapshot ;
- présence de contenu ;
- réussite du rsync.
```

Exemple :

```text
[INFO] Validation: directory exists and not empty
[INFO] Rotation: nothing to delete
[OK] Backup completed
```

---

# 10. Logs

Les logs sont stockés sur :

```text
/var/log/docker-backup/
```

Pour `vm-app-01` :

```text
/var/log/docker-backup/docker-backup-vm-app-01.log
```

Consultation :

```bash
tail -30 \
    /var/log/docker-backup/docker-backup-vm-app-01.log
```

Exemple :

```text
=== Backup vm-app-01 started: ... ===
[INFO] Target: vm-app-01
[INFO] Source: /srv/docker → /srv/backups/docker/vm-app-01
[INFO] Running database backup check
[INFO] Backup gitea-db (postgres)
[INFO] Starting rsync
[OK] rsync success
[INFO] Snapshot: docker-...
[INFO] Validation: directory exists and not empty
[OK] Backup completed
```

---

# 11. Sécurité SSH

L'accès distant utilise un compte dédié :

```text
backup_ro
```

La clé SSH est associée à un wrapper :

```text
/usr/local/sbin/rrsync-wrapper.sh
```

La clé est restreinte afin de n'autoriser que les opérations nécessaires au système de sauvegarde.

Restrictions appliquées :

```text
command="/usr/local/sbin/rrsync-wrapper.sh"
no-agent-forwarding
no-port-forwarding
no-X11-forwarding
no-pty
```

Le wrapper autorise :

* les commandes `rsync` nécessaires à la sauvegarde ;
* l'exécution de `db-backup.sh`.

Toute autre commande est refusée.

---

# 12. Sudo et séparation des privilèges

Les privilèges nécessaires aux opérations de sauvegarde et de restauration sont accordés explicitement via des fichiers dédiés dans :

```text
/etc/sudoers.d/
```

Ces fichiers sont déployés automatiquement par Ansible et validés avec `visudo`.

## `backup_runner` — serveur de sauvegarde

Sur `ct-backup-01` :

```text
/etc/sudoers.d/backup_runner
```

Contenu :

```text
backup_runner ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/bin/rm
```

`backup_runner` peut donc exécuter avec les privilèges `root` uniquement :

* `rsync`
* `rm`

Ces privilèges sont utilisés pour :

* effectuer le `rsync` avec conservation des attributs ;
* supprimer les anciens snapshots lors de la rotation.

---

## `backup_ro` — hôte Docker

Sur `vm-app-01` :

```text
/etc/sudoers.d/backup_ro
```

Contenu :

```text
backup_ro ALL=(root) NOPASSWD: /usr/local/sbin/db-backup.sh, /usr/bin/rrsync
Defaults:backup_ro env_keep += "SSH_ORIGINAL_COMMAND"
```

`backup_ro` peut donc exécuter avec les privilèges `root` uniquement :

* `db-backup.sh`
* `rrsync`

La conservation de `SSH_ORIGINAL_COMMAND` permet au wrapper SSH de transmettre la commande `rsync` à `rrsync`.

---

## `backup_admin` — restauration

Sur les hôtes Docker :

```text
/etc/sudoers.d/backup_admin
```

Contenu :

```text
backup_admin ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/local/sbin/docker-restore.sh
```

`backup_admin` dispose uniquement des privilèges nécessaires à la restauration :

* `rsync`
* `docker-restore.sh`

---

## Validation Ansible

Les fichiers sudoers sont installés avec :

```text
owner: root
group: root
mode: 0440
```

Ansible vérifie leur syntaxe avant installation avec :

```text
visudo -cf %s
```

La séparation des comptes permet donc de limiter les privilèges au strict nécessaire pour chaque fonction.

---

# 13. Supervision Prometheus

Les métriques sont générées dans :

```text
/var/lib/node_exporter/textfile_collector/
```

Pour `vm-app-01` :

```text
backup-vm-app-01.prom
```

Métriques :

```text
backup_status
backup_last_success
backup_last_failure
backup_duration_seconds
```

Exemple :

```text
backup_status{target="vm-app-01"} 1
backup_last_success{target="vm-app-01"} ...
backup_duration_seconds{target="vm-app-01"} ...
```

Interprétation :

```text
backup_status = 1
```

→ dernière sauvegarde réussie.

```text
backup_status = 0
```

→ dernière sauvegarde en échec.

La fraîcheur de la dernière sauvegarde peut être vérifiée avec :

```promql
time() - backup_last_success
```

La supervision permet donc de détecter :

* les échecs de sauvegarde ;
* les sauvegardes trop anciennes ;
* les variations de durée.

---

# 14. Organisation des sauvegardes

Les sauvegardes sont organisées par hôte :

```text
/srv/backups/docker/
├── vm-app-01/
│   ├── current
│   ├── docker-2026-07-29-111549/
│   └── docker-2026-07-29-130931/
│
└── vm-proxy-01/
    ├── current
    └── docker-2026-08-10-185459/
```

Chaque snapshot contient les données de `/srv/docker/` correspondant à l'hôte.

Pour Gitea :

```text
docker-2026-07-29-130931/
├── backups/
│   └── gitea-db.sql
├── gitea/
│   ├── config/
│   ├── data/
│   ├── docker-compose.yml
│   └── redis/
└── ...
```

---

# 15. Restauration

La restauration est lancée depuis :

```text
ct-backup-01
```

Commande :

```bash
sudo /usr/local/sbin/restore.sh \
    vm-app-01 \
    gitea \
    <snapshot>
```

Exemple :

```bash
sudo /usr/local/sbin/restore.sh \
    vm-app-01 \
    gitea \
    docker-2026-07-29-125407
```

Une confirmation explicite est demandée :

```text
Type YES to continue: YES
```

---

# 16. Déroulement d'une restauration

`restore.sh` effectue :

```text
1. Vérification de l'accès SSH
2. Vérification du snapshot
3. Vérification du service
4. Préparation du répertoire temporaire
5. Copie des fichiers du service
6. Copie du dump SQL
7. Appel de docker-restore.sh
```

Le script distant `docker-restore.sh` effectue :

```text
1. Arrêt des containers
2. Restauration des fichiers
3. Démarrage de PostgreSQL
4. Attente de PostgreSQL
5. Restauration du dump SQL
6. Démarrage des containers
7. Vérification des containers
8. Nettoyage du répertoire temporaire
```

---

# 17. Exemple de restauration Gitea

Commande :

```bash
sudo /usr/local/sbin/restore.sh \
    vm-app-01 \
    gitea \
    docker-2026-07-29-125407
```

Résultat attendu :

```text
[INFO] Preparing remote restore directory...
[INFO] Copying service files...
[INFO] Copying database dump...
[INFO] Starting restore...
[INFO] Stopping containers...
[INFO] Restoring files...
[INFO] Starting database...
[INFO] Waiting for database...
[INFO] Database is ready.
[INFO] Restoring database...
[INFO] Database restored.
[INFO] Starting containers...
[INFO] Checking containers...
```

Les containers attendus :

```text
gitea
gitea-db
gitea-redis
```

Résultat final :

```text
[INFO] Restore completed.
```

---

# 18. Répertoire temporaire

La restauration utilise un répertoire temporaire sur l'hôte cible :

```text
/var/tmp/restore-YYYYMMDD-HHMMSS
```

Ce répertoire est supprimé à la fin d'une restauration réussie.

---

# 19. Déploiement

Déploiement complet :

```bash
ansible-playbook playbooks/backup.yml
```

Serveur de sauvegarde :

```bash
ansible-playbook playbooks/backup.yml --tags backup_server
```

Clients Docker :

```bash
ansible-playbook playbooks/backup.yml --tags backup_client
```

Le rôle `backup-client` cible le groupe Ansible :

```text
docker
```

Le rôle n'est donc pas lié à la fonction applicative de l'hôte.

---

# 20. Tests de validation

## Déploiement

* [x] Déploiement `backup-server`
* [x] Déploiement `backup-client`
* [x] Vérification de l'idempotence
* [x] Vérification de la clé SSH
* [x] Vérification du refus lorsque la clé du serveur de backup est absente

## Sauvegarde

* [x] Backup manuel de `vm-app-01`
* [x] Backup de Gitea
* [x] Dump PostgreSQL
* [x] Exclusions `rsync`
* [x] Snapshot créé
* [x] `current` mis à jour
* [x] Rotation vérifiée
* [x] Backup d'un hôte Docker sans base
* [x] Métriques Prometheus générées

## Restauration

* [x] Restauration d'un snapshot Gitea
* [x] Fichiers restaurés
* [x] PostgreSQL restauré
* [x] Containers redémarrés
* [x] Gitea fonctionnel
* [x] Répertoire temporaire supprimé

---

# 21. Commandes de contrôle

### Vérifier le cron

```bash
sudo crontab -u backup_runner -l
```

### Lancer un backup

```bash
sudo -u backup_runner \
    /usr/local/sbin/docker-backup.sh vm-app-01
```

### Vérifier les snapshots

```bash
tree -L 3 /srv/backups/docker/vm-app-01/
```

### Vérifier le log

```bash
tail -30 \
    /var/log/docker-backup/docker-backup-vm-app-01.log
```

### Vérifier les métriques

```bash
cat \
    /var/lib/node_exporter/textfile_collector/backup-vm-app-01.prom
```

### Vérifier la configuration de restauration

Sur `vm-app-01` :

```bash
cat /etc/backup/restore.conf
```

### Vérifier la configuration des bases

```bash
cat /etc/backup/db-backup.conf
```

### Vérifier les droits sudo

Sur `ct-backup-01` :

```bash
sudo cat /etc/sudoers.d/backup_runner
```

Sur un Docker host :

```bash
sudo cat /etc/sudoers.d/backup_ro
sudo cat /etc/sudoers.d/backup_admin
```

---

# 22. Arborescence du composant

```text
backup/
├── README.md
├── Backup & Restore.md
├── Troubleshooting.md
├── docker-backup.sh
├── db-backup.sh
└── images/
    └── backup-architecture.png
```

Le présent document décrit le fonctionnement technique du système de sauvegarde et de restauration.

`Troubleshooting.md` regroupe les procédures de diagnostic et les cas d'erreur rencontrés en exploitation.

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab