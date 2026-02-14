# Ingress Nginx - Proxy Tuning (Timeouts et Limites)

## Le probleme

ingress-nginx est un proxy : il recoit les requetes des clients et les transmet aux services. Par defaut, il a des limites strictes :

| Parametre | Valeur par defaut | Signification |
|-----------|-------------------|---------------|
| `proxy-body-size` | 1m (1 MB) | Taille maximale du corps d'une requete |
| `proxy-read-timeout` | 60s | Temps max pour lire la reponse du service |
| `proxy-send-timeout` | 60s | Temps max pour envoyer la requete au service |

Ces valeurs par defaut ne conviennent pas a tous les services.

## Pourquoi on doit les changer

### Taille du body (`proxy-body-size`)

Quand tu importes un dashboard JSON dans Grafana, le fichier JSON peut faire **5-20 MB**. Avec la limite de 1 MB, l'import echoue avec une erreur `413 Request Entity Too Large`.

Quand Alloy envoie des logs a Loki via l'Ingress, les batches peuvent faire **plusieurs MB**.

### Timeouts (`proxy-read-timeout`, `proxy-send-timeout`)

Quand tu fais une requete PromQL sur Prometheus avec un intervalle de 30 jours, Prometheus peut mettre **plusieurs minutes** a calculer le resultat. Avec un timeout de 60s, la requete est coupee et tu obtiens une erreur `504 Gateway Timeout`.

## Comment on configure

On utilise des **annotations** sur chaque ressource Ingress. Chaque service peut avoir des valeurs differentes selon ses besoins.

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
```

## Configuration par service dans notre projet

### Grafana

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "50m"
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

- **body 50m** : import de dashboards JSON, upload d'images
- **read timeout 300s** (5 min) : certaines requetes Grafana (explore, gros dashboards) prennent du temps

### Prometheus

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "50m"
nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
```

- **body 50m** : requetes PromQL complexes
- **read timeout 600s** (10 min) : les requetes sur de gros intervalles de temps sont longues
- **send timeout 600s** : les reponses avec beaucoup de donnees prennent du temps a transmettre

Prometheus a les timeouts les plus longs car les requetes PromQL sont les plus couteuses.

### Alertmanager

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

- **body 10m** : les silences et configurations Alertmanager sont legers
- Pas de timeout personnalise : les requetes sont rapides

### Blackbox

```
(aucune annotation de proxy tuning)
```

- Les requetes vers Blackbox sont legeres et rapides
- Les valeurs par defaut (1m body, 60s timeout) suffisent

### Loki

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "50m"
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

- **body 50m** : Alloy envoie des batches de logs qui peuvent etre gros
- **read timeout 300s** : les recherches dans les logs sur de longues periodes prennent du temps

## Tableau recapitulatif

| Service | body-size | read-timeout | send-timeout |
|---------|----------|-------------|-------------|
| Grafana | 50m | 300s (5 min) | defaut (60s) |
| Prometheus | 50m | 600s (10 min) | 600s (10 min) |
| Alertmanager | 10m | defaut (60s) | defaut (60s) |
| Blackbox | defaut (1m) | defaut (60s) | defaut (60s) |
| Loki | 50m | 300s (5 min) | defaut (60s) |

## Les 3 annotations expliquees

### `proxy-body-size`

Limite la taille du corps de la requete HTTP. Si depasse → erreur `413 Request Entity Too Large`.

```
Client envoie une requete de 25 MB
       │
       ▼
proxy-body-size = 50m → OK, on laisse passer
proxy-body-size = 1m  → BLOQUE, erreur 413
```

La valeur s'ecrit avec un suffixe : `m` = megabytes, `k` = kilobytes. `0` = pas de limite.

### `proxy-read-timeout`

Temps maximum qu'ingress-nginx attend pour recevoir une reponse du service backend. Si depasse → erreur `504 Gateway Timeout`.

```
Client envoie une requete PromQL
       │
       ▼
ingress-nginx forward vers Prometheus
       │
       ▼
Prometheus calcule pendant 3 minutes...
       │
       ▼
proxy-read-timeout = 600s → OK, on attend
proxy-read-timeout = 60s  → TIMEOUT apres 60s, erreur 504
```

La valeur est en **secondes**.

### `proxy-send-timeout`

Temps maximum qu'ingress-nginx attend pour envoyer la requete au service backend. Utilise quand le body de la requete est gros et prend du temps a transmettre.

En pratique, `proxy-send-timeout` est rarement le goulot d'etranglement. `proxy-read-timeout` est plus important.

## Autres annotations de proxy disponibles

On ne les utilise pas dans notre projet, mais elles existent :

| Annotation | Description |
|------------|-------------|
| `proxy-connect-timeout` | Timeout pour etablir la connexion avec le backend (defaut 5s) |
| `proxy-buffer-size` | Taille du buffer pour les headers de reponse |
| `proxy-buffers-number` | Nombre de buffers |
| `proxy-max-temp-file-size` | Taille max des fichiers temporaires |

## Comment debugger les erreurs de proxy

### Erreur 413 Request Entity Too Large

```
Cause : la requete depasse proxy-body-size
Fix : augmenter proxy-body-size dans l'annotation de l'Ingress
```

### Erreur 504 Gateway Timeout

```
Cause : le service a mis trop longtemps a repondre
Fix : augmenter proxy-read-timeout dans l'annotation de l'Ingress
Attention : si le service est lent, c'est peut-etre un probleme de performance
```

### Erreur 502 Bad Gateway

```
Cause : le service backend est down ou inaccessible
Fix : verifier que le pod du service est en Running
      k8s kubectl get pods -n observability
```

## Resume

- ingress-nginx a des limites par defaut (1 MB body, 60s timeout) qui sont trop strictes pour certains services
- On personnalise les limites par service avec des annotations
- `proxy-body-size` = taille max de la requete
- `proxy-read-timeout` = temps max pour recevoir la reponse
- `proxy-send-timeout` = temps max pour envoyer la requete
- Prometheus a les timeouts les plus longs (10 min) car les requetes PromQL sont les plus couteuses
- Blackbox n'a pas besoin de tuning car ses requetes sont legeres
