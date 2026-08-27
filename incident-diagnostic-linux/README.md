# Incident Diagnostic Linux
## Incident : 502 Bad Gateway sur Gitea via Traefik

### Objectif

Documenter un incident réel de diagnostic sur une infrastructure Linux composée de :

* Traefik (reverse proxy)
* Gitea (service applicatif)
* PostgreSQL
* Redis

Infrastructure :

```text
Client
↓
Traefik (vm-proxy-01)
↓
Gitea (vm-app-01)
```

---

## Symptôme

Le service Git :

```text
https://git.farnet.tech
```

retourne :

```text
502 Bad Gateway
```

Observation initiale :

```bash
curl -I https://git.farnet.tech
HTTP/2 502
```

---

## Hypothèses initiales

Plusieurs causes possibles :

* Traefik indisponible
* erreur de routage Traefik
* backend Gitea arrêté
* port backend inaccessible
* erreur réseau
* configuration applicative invalide

L’objectif est d’isoler la cause réelle.

---

## Étape 1 — Vérification du reverse proxy

Validation du service Traefik :

```bash
docker ps | grep traefik
```

Résultat :

```text
running
```

Conclusion :

Traefik fonctionne.

---

## Étape 2 — Analyse des logs Traefik

Consultation :

```bash
docker logs traefik --tail 50
```

Logs observés :

```text
"gitea@file" "http://10.35.20.10:3000"
502
```

Analyse :

* le router Traefik correspond correctement
* le backend cible est bien identifié
* Traefik tente bien la connexion

Conclusion :

le problème est situé côté backend.

---

## Étape 3 — Vérification du conteneur Gitea

Contrôle :

```bash
docker ps
```

Résultat :

```text
gitea   running
```

Conclusion :

le conteneur est actif.

Le problème n’est pas un arrêt de service.

---

## Étape 4 — Test applicatif local

Sur le serveur applicatif :

```bash
curl http://localhost:3000
```

Résultat :

```text
Recv failure: Connexion ré-initialisée
```

Puis :

```bash
curl http://10.35.20.10:3000
```

Résultat :

```text
Failed to connect
```

Conclusion :

le service ne répond pas correctement.

---

## Étape 5 — Inspection du socket interne

Inspection réseau dans le conteneur :

```bash
docker exec -it gitea netstat -lntp
```

Résultat :

```text
tcp 127.0.0.1:3000 LISTEN 7/gitea
```

Analyse :

Gitea écoute uniquement sur :

```text
127.0.0.1
```

et non :

```text
0.0.0.0
```

Cela empêche toute connexion réseau externe au conteneur.

Cause identifiée.

---

## Cause racine

Dans :

```text
/data/gitea/conf/app.ini
```

Configuration incorrecte :

```ini
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3000
```

Cette configuration limite l’écoute au loopback interne.

Traefik ne peut donc pas joindre le service.

---

## Correction

Modification :

```ini
HTTP_ADDR = 0.0.0.0
HTTP_PORT = 3000
```

Redémarrage :

```bash
docker restart gitea
```

---

## Validation

Test depuis le proxy :

```bash
curl -I https://git.farnet.tech
```

Résultat :

```text
HTTP/2 200
```

Service restauré.

---

## Analyse post-incident

Points importants :

* un conteneur `running` ne garantit pas qu’un service soit accessible
* un port Docker publié ne garantit pas que l’application écoute correctement
* la vérification du bind applicatif est critique
* l’analyse doit suivre toute la chaîne de service

Méthode appliquée :

1. vérifier le point d’entrée
2. vérifier le reverse proxy
3. vérifier les logs
4. vérifier le backend
5. vérifier le service
6. vérifier l’écoute réseau
7. corriger
8. valider

---

## Compétences mobilisées

* diagnostic Linux
* analyse réseau
* reverse proxy Traefik
* Docker
* troubleshooting applicatif
* lecture de logs
* validation de service

---
## 🔗 Dépôt Ansible

→ https://github.com/frederic-am/ansible-lab