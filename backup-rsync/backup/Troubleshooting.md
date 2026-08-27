# Troubleshooting

Guide de diagnostic du système de sauvegarde et de restauration.

L'objectif est d'identifier rapidement l'origine d'un échec avant de modifier la configuration.

---

# 1. Backup en échec

Commencer par consulter le log de l'hôte concerné :

```bash
tail -50 \
    /var/log/docker-backup/docker-backup-<host>.log
````

Exemple :

```bash
tail -50 \
    /var/log/docker-backup/docker-backup-vm-app-01.log
```

Identifier la première erreur rencontrée.

Ne pas se limiter au dernier message :

```text
[ERROR] Backup failed
```

Le message précédent indique généralement l'étape qui a réellement échoué.

---

# 2. `Remote DB backup failed`

Message :

```text
[ERROR] Remote DB backup failed
```

Le serveur de sauvegarde n'a pas réussi à exécuter le backup de base de données sur l'hôte distant.

## Vérifier l'accès SSH et le wrapper

Depuis `ct-backup-01` :

```bash
sudo -u backup_runner \
    ssh -i /home/backup_runner/.ssh/backup_rsync \
    -o BatchMode=yes \
    backup_ro@<host> \
    "/usr/local/sbin/db-backup.sh"
```

La commande doit permettre l'exécution de `db-backup.sh` sans demander de mot de passe. Si elle échoue, consulter le message retourné avant de poursuivre le diagnostic.

## Vérifier le compte distant

Sur l'hôte Docker :

```bash
id backup_ro
```

## Tester directement le backup DB

Sur l'hôte Docker :

```bash
sudo /usr/local/sbin/db-backup.sh
```

Puis vérifier :

```bash
ls -lh /srv/docker/backups/
```

Si le script échoue localement, le problème se situe sur l'hôte Docker et non dans `docker-backup.sh`.

---

# 3. Hôte sans base de données

Message :

```text
[INFO] No database configured
```

Ce message est normal lorsqu'aucune base n'est définie pour l'hôte.

Le backup doit continuer avec :

```text
[INFO] Starting rsync
[OK] rsync success
```

L'absence de base ne doit jamais empêcher la sauvegarde de :

```text
/srv/docker/
```

Si aucun snapshot n'est créé malgré `No database configured`, poursuivre le diagnostic sur l'étape `rsync`.

---

# 4. Erreur SSH

## `Connection timed out`

Exemple :

```text
ssh: connect to host <host> port 22: Connection timed out
```

Vérifier depuis `ct-backup-01` :

```bash
ping -c 3 <host>
```

Puis :

```bash
nc -vz -w 5 <host> 22
```

Si le port 22 est inaccessible :

* vérifier la connectivité réseau ;
* vérifier les règles firewall ;
* vérifier que le serveur SSH est actif ;
* vérifier que l'adresse IP utilisée est correcte.

Dans le lab, les règles réseau doivent autoriser le flux :

```text
ct-backup-01
      │
      └── TCP/22 ──► Docker host
```

---

## `No route to host`

Exemple :

```text
ssh: connect to host <host> port 22:
No route to host
```

Vérifier :

```bash
ip route
```

et :

```bash
ping -c 3 <IP>
```

Si le serveur est accessible depuis un autre segment mais pas depuis `ct-backup-01`, vérifier le routage et les règles `iptables` du lab.

---

## `Permission denied (publickey)`

Exemple :

```text
Permission denied (publickey).
```

Vérifier que la clé utilisée par `backup_runner` est bien celle déployée pour `backup_ro`.

Tester :

```bash
sudo -u backup_runner \
    ssh -v \
    -i /home/backup_runner/.ssh/backup_rsync \
    backup_ro@<host> \
    "/usr/local/sbin/db-backup.sh"
```

Ainsi, le test vérifie **à la fois la clé et une commande autorisée par le wrapper**.

---

# 5. Wrapper SSH 

Le wrapper n'autorise que :

* les commandes `rsync` nécessaires aux sauvegardes ;
* `/usr/local/sbin/db-backup.sh`.

Toute autre commande doit être refusée.

## Tester le refus d'une commande arbitraire

Depuis `ct-backup-01` :

```bash
sudo -u backup_runner ssh \
    -i /home/backup_runner/.ssh/backup_rsync \
    backup_ro@vm-app-01 \
    "id"
```

Résultat attendu :

```text
Denied
```

Le code retour doit être `1`.

Ce test vérifie que le compte `backup_ro` ne permet pas d'obtenir un shell SSH classique.

## Tester l'exécution du backup DB

Depuis `ct-backup-01` :

```bash
sudo -u backup_runner ssh \
    -i /home/backup_runner/.ssh/backup_rsync \
    backup_ro@vm-app-01 \
    "/usr/local/sbin/db-backup.sh"
```

Résultat attendu pour l'environnement de test Gitea :

```text
[INFO] Backup gitea-db (postgres)
```

Le code retour doit être `0`.

## Tester le chemin rsync

Effectuer un transfert à blanc :

```bash
sudo -u backup_runner rsync \
    -aHAX \
    --dry-run \
    -e "ssh -i /home/backup_runner/.ssh/backup_rsync -o BatchMode=yes" \
    backup_ro@vm-app-01:/srv/docker/ \
    /tmp/test-backup/
```

Le wrapper doit accepter la commande `rsync` et aucune erreur `Denied` ne doit apparaître.

## Vérifier le sudo associé au wrapper

Sur l'hôte Docker :

```bash
sudo cat /etc/sudoers.d/backup_ro
```

Le fichier doit autoriser :

```text
backup_ro ALL=(root) NOPASSWD: /usr/local/sbin/db-backup.sh, /usr/bin/rrsync
Defaults:backup_ro env_keep += "SSH_ORIGINAL_COMMAND"
```

Ces tests permettent de vérifier séparément les trois comportements attendus :

```text
commande arbitraire  → refusée
db-backup.sh         → autorisé
rsync                → autorisé
```

---

# 6. Erreur de permissions sur `/srv/docker/backups`

Vérifier les permissions :

```bash
ls -ld /srv/docker/backups
ls -la /srv/docker/backups/
````

Tester directement le script avec les privilèges root :

```bash
sudo /usr/local/sbin/db-backup.sh
```

Puis tester le chemin réellement utilisé par `backup_ro` :

```bash
sudo -u backup_ro sudo -n /usr/local/sbin/db-backup.sh
```

Vérifier les droits sudo :

```bash
sudo cat /etc/sudoers.d/backup_ro
```

Le fichier doit autoriser :

```text
backup_ro ALL=(root) NOPASSWD: /usr/local/sbin/db-backup.sh, /usr/bin/rrsync
```

Le script `db-backup.sh` est ainsi exécuté avec les privilèges root tout en restant accessible à `backup_ro` uniquement via la règle sudo explicitement définie.

---

# 7. Échec `rsync`

Message possible :

```text
[ERROR] rsync failed
```

Consulter le log :

```bash
tail -50 \
    /var/log/docker-backup/docker-backup-<host>.log
```

Tester l'accès SSH avec les mêmes identifiants que le script de backup, en utilisant une commande autorisée par le wrapper :

```bash
sudo -u backup_runner ssh \
    -i /home/backup_runner/.ssh/backup_rsync \
    -o BatchMode=yes \
    backup_ro@<host> \
    "/usr/local/sbin/db-backup.sh"
```

Si cette commande fonctionne mais que `rsync` échoue, poursuivre le diagnostic  
sur les exclusions, la source distante et le transfert `rsync`.

Vérifier ensuite les exclusions :

```bash
cat /etc/backup/<host>.txt
```

Vérifier la présence de la source distante avec le même compte :

```bash
sudo -u backup_runner ssh \
    -i /home/backup_runner/.ssh/backup_rsync \
    -o BatchMode=yes \
    backup_ro@<host> \
    "ls -ld /srv/docker"
```

Vérifier également le répertoire de destination :

```bash
ls -ld /srv/backups/docker/<host>
```

Si l'accès SSH fonctionne mais que `rsync` échoue, consulter le message d'erreur détaillé dans le log afin de distinguer un problème :

* de permissions ;
* d'exclusion ;
* de connexion SSH ;
* de lecture de `/srv/docker` ;
* d'écriture sur le serveur de sauvegarde.

---

# 8. Snapshot absent ou vide

Après un backup, vérifier :

```bash
ls -lah /srv/backups/docker/<host>/
```

Puis :

```bash
tree -L 3 /srv/backups/docker/<host>/
```

Le snapshot doit contenir des données.

Exemple :

```text
/srv/backups/docker/vm-app-01/
├── current
└── docker-2026-07-29-130931/
    ├── backups/
    ├── gitea/
    └── ...
```

Vérifier également :

```bash
readlink -f /srv/backups/docker/<host>/current
```

`current` doit pointer vers le dernier snapshot valide.

---

# 9. Vérifier `--link-dest`

Pour confirmer que les snapshots réutilisent les fichiers inchangés :

```bash
stat <snapshot1>/path/to/file
stat <snapshot2>/path/to/file
```

Un même inode sur deux snapshots indique que le fichier est partagé par hard link.

Exemple :

```text
917582 -rw-r--r-- ... app.ini
917582 -rw-r--r-- ... app.ini
```

Le même numéro d'inode indique que les deux entrées utilisent le même fichier physique.

---

# 10. Métriques Prometheus absentes

Les métriques sont générées sur `ct-backup-01` dans :

```bash
ls -la \
    /var/lib/node_exporter/textfile_collector/
```

Pour `vm-app-01` :

```bash
ls -l \
    /var/lib/node_exporter/textfile_collector/backup-vm-app-01.prom
```

Puis :

```bash
cat \
    /var/lib/node_exporter/textfile_collector/backup-vm-app-01.prom
```

Les métriques attendues après un backup réussi sont notamment :

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

Si le fichier n'existe pas ou ne peut pas être écrit, vérifier les permissions du répertoire :

```bash
ls -ld \
    /var/lib/node_exporter/textfile_collector/
```

et l'utilisateur qui exécute `docker-backup.sh`:

```bash
ps -o user,group,cmd -C docker-backup.sh
```

Le fichier `.prom` doit être créé sur `ct-backup-01`, et non sur l'hôte Docker sauvegardé.

---

# 11. Vérifier la fraîcheur d'une sauvegarde

Dans Prometheus :

```promql
time() - backup_last_success
```

Pour un hôte donné :

```promql
time() - backup_last_success{target="vm-app-01"}
```

Une valeur élevée indique que la dernière sauvegarde réussie est ancienne.

Pour vérifier directement l'état :

```promql
backup_status{target="vm-app-01"}
```

---

# 12. Restauration en échec

Consulter d'abord le résultat de :

```bash
sudo /usr/local/sbin/restore.sh \
    <host> \
    <service> \
    <snapshot>
```

Les étapes sont exécutées dans l'ordre :

```text
SSH
 ↓
copie des fichiers
 ↓
copie du dump
 ↓
docker-restore.sh
 ↓
arrêt des containers
 ↓
restauration des fichiers
 ↓
PostgreSQL
 ↓
restauration SQL
 ↓
redémarrage
 ↓
vérification
```

Identifier l'étape qui échoue avant de modifier la configuration.

---

# 13. PostgreSQL ne devient pas disponible

Message :

```text
[ERROR] Database did not become ready.
```

Vérifier les containers :

```bash
docker compose \
    -f /srv/docker/gitea/docker-compose.yml \
    ps
```

Vérifier les logs PostgreSQL :

```bash
docker compose \
    -f /srv/docker/gitea/docker-compose.yml \
    logs db
```

Vérifier également les paramètres de :

```text
/etc/backup/restore.conf
```

Exemple :

```text
gitea;postgres;db;gitea;gitea
```

Les valeurs doivent correspondre au service Docker et à la base réellement utilisée.

---

# 14. Restauration PostgreSQL échoue

Vérifier la présence du dump dans le snapshot :

```bash
ls -lh \
    /srv/backups/docker/<host>/<snapshot>/backups/
```

Pour Gitea :

```text
gitea-db.sql
```

Vérifier ensuite :

```bash
cat /etc/backup/restore.conf
```

et notamment :

```text
db_type
compose_service
database
user
```

Le dump doit correspondre à la configuration du service restauré.

---

# 15. Containers non démarrés après restauration

Vérifier :

```bash
docker compose \
    -f /srv/docker/gitea/docker-compose.yml \
    ps
```

Puis :

```bash
docker compose \
    -f /srv/docker/gitea/docker-compose.yml \
    logs --tail=100
```

Vérifier en particulier :

* PostgreSQL ;
* Gitea ;
* Redis.

Une restauration considérée comme réussie doit se terminer avec les containers attendus en état `running`.

---

# 16. Vérifier les privilèges sudo

Sur `ct-backup-01` :

```bash
sudo cat /etc/sudoers.d/backup_runner
```

Attendu :

```text
backup_runner ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/bin/rm
```

Sur un Docker host :

```bash
sudo cat /etc/sudoers.d/backup_ro
sudo cat /etc/sudoers.d/backup_admin
```

Vérifier également la syntaxe :

```bash
sudo visudo -cf /etc/sudoers.d/backup_runner
sudo visudo -cf /etc/sudoers.d/backup_ro
sudo visudo -cf /etc/sudoers.d/backup_admin
```

---

# 17. Vérifier le déploiement Ansible

En cas de configuration incorrecte ou de fichier manquant, relancer le déploiement :

```bash
ansible-playbook playbooks/backup.yml
```

Puis vérifier l'idempotence :

```bash
ansible-playbook playbooks/backup.yml
```

Une seconde exécution ne doit pas provoquer de modifications inutiles.

---

# 18. Vérifications rapides

Pour un problème de backup, suivre cet ordre :

```text
1. Log du backup
       ↓
2. Connectivité SSH
       ↓
3. Compte backup_ro
       ↓
4. Wrapper SSH
       ↓
5. sudoers
       ↓
6. db-backup.sh
       ↓
7. /srv/docker/backups/
       ↓
8. rsync
       ↓
9. snapshot
       ↓
10. métriques Prometheus
```

Pour un problème de restauration :

```text
1. Snapshot
       ↓
2. Service demandé
       ↓
3. Copie des fichiers
       ↓
4. Dump SQL
       ↓
5. PostgreSQL
       ↓
6. Restauration SQL
       ↓
7. Containers
       ↓
8. Application
```

---

# 19. Principes de diagnostic

Avant toute modification :

* identifier l'étape exacte qui échoue ;
* reproduire le problème manuellement ;
* vérifier les logs ;
* vérifier les permissions ;
* vérifier SSH et le réseau ;
* vérifier la configuration générée par Ansible.

Éviter de modifier simultanément plusieurs composants.

Une modification doit être suivie d'un nouveau test de sauvegarde ou de restauration afin de confirmer précisément la correction.

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab