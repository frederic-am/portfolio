# Monitoring Prometheus & Grafana — Troubleshooting

Guide de diagnostic de la plateforme de supervision du laboratoire.

L'objectif est d'identifier rapidement l'étape qui échoue avant de modifier la configuration.

---

# 1. Target Prometheus `DOWN`

Vérifier les targets dans Prometheus.

Pour un diagnostic direct, vérifier l'endpoint de métriques depuis `vm-monitor-01`.

Exemple pour Node Exporter :

```bash
curl http://<host>:9100/metrics
```

Résultat attendu : une réponse contenant des métriques Prometheus, par exemple :

```text
node_cpu_seconds_total
node_memory_MemTotal_bytes
```

Si l'endpoint ne répond pas, poursuivre le diagnostic sur l'exporter et le réseau.

---

# 2. Node Exporter inaccessible

Sur l'hôte concerné :

```bash
systemctl status node-exporter
```

Vérifier également l'écoute du port :

```bash
ss -lntp | grep 9100
```

Puis tester depuis `vm-monitor-01` :

```bash
curl http://<host>:9100/metrics
```

Si Node Exporter fonctionne localement mais reste inaccessible depuis `vm-monitor-01`, vérifier :

* l'adresse IP utilisée ;
* le routage ;
* les règles firewall ;
* le port TCP `9100`.

---

# 3. cAdvisor inaccessible

Sur l'hôte Docker concerné :

```bash
docker ps
```

Vérifier que le conteneur cAdvisor est présent et démarré.

Puis :

```bash
docker logs cadvisor
```

Tester l'endpoint depuis `vm-monitor-01` :

```bash
curl http://<host>:18080/metrics
```

Une réponse contenant des métriques `container_*` indique que cAdvisor expose correctement ses métriques.

Si le conteneur fonctionne mais que Prometheus ne peut pas le joindre, vérifier le réseau et le port exposé.

---

# 4. Métriques absentes dans Prometheus

Commencer par vérifier que la target correspondante est `UP`.

Si la target est `DOWN`, ne pas chercher le problème dans PromQL : corriger d'abord la collecte.

Si la target est `UP`, rechercher directement la métrique dans Prometheus.

Exemple :

```promql
up{job="node"}
```

Pour les sauvegardes :

```promql
backup_status
```

Pour les métriques système :

```promql
node_memory_MemTotal_bytes
```

Pour les conteneurs :

```promql
container_cpu_usage_seconds_total
```

Si aucune série n'est retournée alors que la target est `UP`, vérifier le nom exact de la métrique exposée par l'exporter.

---

# 5. Métriques présentes dans Prometheus mais absentes dans Grafana

Si une métrique apparaît dans Prometheus mais pas dans Grafana, le problème se situe généralement au niveau du dashboard ou de la requête PromQL.

Commencer par copier la requête du panneau concerné et l'exécuter directement dans Prometheus.

Vérifier notamment :

* le nom de la métrique ;
* le `job` ;
* les labels ;
* les filtres utilisés ;
* la période temporelle sélectionnée.

Exemple :

```promql
backup_status{target="vm-app-01"}
```

Si la requête fonctionne dans Prometheus mais pas dans Grafana, vérifier la datasource Prometheus configurée dans Grafana.

---

# 6. Problème de requête PromQL

Tester les requêtes directement dans Prometheus avant de modifier un dashboard.

Exemple :

```promql
time() - backup_last_success{target="vm-app-01"}
```

Cette requête permet de déterminer depuis combien de temps la dernière sauvegarde réussie a été enregistrée.

Pour vérifier l'état actuel :

```promql
backup_status{target="vm-app-01"}
```

Pour vérifier la disponibilité d'un hôte :

```promql
up{job="node"}
```

Lorsqu'une requête ne retourne aucun résultat, vérifier d'abord les labels disponibles :

```promql
up
```

puis affiner progressivement la requête.

---

# 7. Conteneur Docker absent des métriques

Vérifier que cAdvisor fonctionne :

```bash
docker ps
```

Puis :

```bash
docker logs cadvisor
```

Tester les métriques :

```bash
curl http://<host>:8080/metrics
```

Rechercher ensuite le conteneur dans Prometheus :

```promql
container_last_seen
```

ou :

```promql
container_cpu_usage_seconds_total
```

Si cAdvisor expose les métriques mais qu'un conteneur n'apparaît pas, vérifier les labels et la manière dont le conteneur est identifié dans les métriques.

---

# 8. Métriques Traefik absentes

Vérifier que Traefik fonctionne :

```bash
docker ps
```

Puis consulter ses logs :

```bash
docker logs traefik
```

Vérifier ensuite que les métriques Prometheus sont exposées par Traefik.

Depuis l'hôte concerné :

```bash
curl http://localhost:<port>/metrics
```

Le port doit correspondre au endpoint de métriques configuré pour Traefik.

Rechercher ensuite dans Prometheus une métrique Traefik connue.

Par exemple :

```promql
traefik_entrypoint_requests_total
```

Si les métriques sont visibles dans Prometheus mais pas dans Grafana, poursuivre avec le diagnostic de la requête PromQL et du dashboard.

---

# 9. Métriques de sauvegarde absentes

Les métriques de sauvegarde sont générées sur `ct-backup-01`.

Vérifier le répertoire du Textfile Collector :

```bash
ls -la /var/lib/node_exporter/textfile_collector/
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

Après une sauvegarde réussie, on doit notamment trouver :

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

Si le fichier n'existe pas, vérifier :

```bash
ls -ld /var/lib/node_exporter/textfile_collector/
```

Puis vérifier l'utilisateur qui exécute le script de sauvegarde.

Le problème doit être diagnostiqué sur `ct-backup-01` avant de chercher un problème dans Prometheus.

---

# 10. Alerte non déclenchée

Commencer par exécuter directement l'expression de l'alerte dans Prometheus.

Pour `VmDown` :

```promql
up{job="node"} == 0
```

Pour `BackupFailed` :

```promql
backup_status == 0
```

Pour `BackupStale` :

```promql
(time() - backup_last_success) > 86400
```

Pour `DiskUsageHigh`, vérifier l'expression utilisée dans les règles Prometheus.

Si l'expression ne retourne aucune série, l'alerte ne peut pas se déclencher.

Si l'expression retourne une série mais que l'alerte n'apparaît pas immédiatement, vérifier notamment la durée `for:` définie dans la règle.

---

# 11. Vérifier les règles Prometheus

Vérifier le fichier des règles sur `vm-monitor-01`.

Puis vérifier la configuration chargée par Prometheus :

```bash
systemctl status prometheus
```

Consulter les logs :

```bash
journalctl -u prometheus -n 50 --no-pager
```

Après une modification des règles, vérifier que Prometheus redémarre correctement.

Une erreur YAML ou une erreur de syntaxe dans une règle peut empêcher Prometheus de charger les alertes.

---

# 12. Prometheus ne démarre plus après une modification

Vérifier immédiatement le service :

```bash
systemctl status prometheus
```

Puis :

```bash
journalctl -u prometheus -n 100 --no-pager
```

Rechercher notamment :

```text
error
failed
yaml
loading groups
```

Une erreur dans le fichier de règles ou dans la configuration Prometheus doit être corrigée avant de poursuivre.

Après correction :

```bash
systemctl restart prometheus
```

Puis :

```bash
systemctl status prometheus
```

---

# 13. Grafana n'affiche plus les données

Vérifier que Grafana fonctionne :

```bash
systemctl status grafana-server
```

Vérifier ensuite que Prometheus est disponible.

Dans Grafana, vérifier la datasource Prometheus utilisée par les dashboards.

Si la datasource fonctionne mais qu'un panneau reste vide :

1. tester la requête PromQL directement dans Prometheus ;
2. vérifier les labels ;
3. vérifier la période temporelle ;
4. vérifier les variables Grafana utilisées par le panneau.

---

# 14. Problème de déploiement Ansible

Relancer le déploiement du rôle concerné.

Exemple :

```bash
ansible-playbook playbooks/monitoring.yml
```

Puis vérifier l'idempotence en relançant :

```bash
ansible-playbook playbooks/monitoring.yml
```

La seconde exécution ne doit pas provoquer de modifications inutiles.

Après un déploiement, vérifier les services concernés :

```bash
systemctl status prometheus
systemctl status grafana-server
systemctl status node-exporter
```

Sur les hôtes Docker, vérifier également cAdvisor.

---

# 15. Méthode de diagnostic

Pour une métrique absente :

```text
Exporter
   ↓
endpoint /metrics
   ↓
réseau
   ↓
Prometheus target
   ↓
métrique Prometheus
   ↓
requête PromQL
   ↓
Grafana
```

Pour une alerte :

```text
Exporter
   ↓
Prometheus
   ↓
métrique
   ↓
expression PromQL
   ↓
règle d'alerte
   ↓
durée `for`
   ↓
alerte
```

Pour les sauvegardes :

```text
docker-backup.sh
       ↓
fichier .prom
       ↓
Node Exporter
       ↓
Prometheus
       ↓
PromQL
       ↓
Dashboard / Alert
```

Le diagnostic doit suivre cette chaîne dans l'ordre.

---

# 16. Principes de diagnostic

Avant toute modification :

* identifier la couche qui échoue ;
* vérifier l'état du service concerné ;
* vérifier l'endpoint `/metrics` ;
* vérifier la target Prometheus ;
* tester la métrique directement dans Prometheus ;
* tester ensuite la requête PromQL ;
* vérifier enfin Grafana ou la règle d'alerte.

Éviter de modifier simultanément plusieurs composants.

Une modification doit être suivie d'un nouveau test afin de confirmer précisément la correction.

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab