# Ingress Nginx - Reverse Proxy

## Le probleme

Dans Kubernetes, les services tournent dans un **reseau interne**. Ils ne sont pas accessibles depuis l'exterieur.

Exemple : Grafana tourne sur le port 3000, mais ce port 3000 est **interne au cluster**. Si tu fais `http://IP_VM:3000` depuis ton navigateur, ca ne marche pas.

## C'est quoi un Reverse Proxy

Un reverse proxy est un serveur qui se place **entre le client (toi) et les services**. Il recoit toutes les requetes et les redirige vers le bon service.

```
Client (navigateur)
       │
       ▼
Reverse Proxy (ingress-nginx)     ← ecoute sur les ports 80 et 443 de la VM
       │
       ├──> Grafana (port 3000)
       ├──> Prometheus (port 9090)
       ├──> Alertmanager (port 9093)
       ├──> Blackbox (port 9115)
       └──> Loki (port 3100)
```

Un seul point d'entree (port 443) pour acceder a tous les services.

## Comment il sait ou envoyer le trafic ?

Par le **nom de domaine** (Host header HTTP).

Quand ton navigateur fait une requete vers `https://grafana.kadhem.kaascloud.auroraiq.cloud`, il envoie un header HTTP :

```
Host: grafana.kadhem.kaascloud.auroraiq.cloud
```

ingress-nginx lit ce header et cherche dans ses regles quelle regle correspond a ce domaine. Chaque regle dit : "si le domaine est X, envoie vers le service Y".

## Les ressources Ingress

Dans Kubernetes, une **ressource Ingress** est un objet YAML qui definit une regle de routage. C'est un objet Kubernetes standard (comme un Pod, un Service, un Deployment).

On a **5 ressources Ingress** dans notre projet (une par service) :

| Fichier | Domaine | Service cible | Port |
|---------|---------|--------------|------|
| `ingress-grafana.yaml.j2` | `grafana.CLIENT.kaascloud.auroraiq.cloud` | grafana | 3000 |
| `ingress-prometheus.yaml.j2` | `prometheus.CLIENT.kaascloud.auroraiq.cloud` | prometheus | 9090 |
| `ingress-alertmanager.yaml.j2` | `alertmanager.CLIENT.kaascloud.auroraiq.cloud` | alertmanager | 9093 |
| `ingress-blackbox.yaml.j2` | `blackbox.CLIENT.kaascloud.auroraiq.cloud` | blackbox | 9115 |
| `ingress-loki.yaml.j2` | `loki.CLIENT.kaascloud.auroraiq.cloud` | loki | 3100 |

`CLIENT` est remplace par le nom du client (ex: `kadhem`).

## Structure d'une ressource Ingress

Voici l'Ingress de Grafana (partie routage uniquement) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: observability
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.kadhem.kaascloud.auroraiq.cloud
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

Ligne par ligne :

| Champ | Signification |
|-------|--------------|
| `kind: Ingress` | C'est une ressource Ingress |
| `namespace: observability` | L'Ingress est dans le namespace `observability` |
| `ingressClassName: nginx` | C'est ingress-nginx qui gere cette regle (pas un autre controller) |
| `host: grafana.kadhem...` | Cette regle s'applique quand le domaine est celui-ci |
| `path: /` | Pour tout le trafic (toutes les URLs) |
| `pathType: Prefix` | Match par prefixe (/ match tout) |
| `backend.service.name` | Le nom du Service Kubernetes cible |
| `backend.service.port.number` | Le port du Service |

## Comment les 5 regles fonctionnent ensemble

ingress-nginx charge **toutes** les ressources Ingress du cluster. Quand une requete arrive :

```
Requete arrive sur port 443
       │
       ▼
ingress-nginx lit le Host header
       │
       ├── Host = grafana.kadhem.kaascloud...     → forward vers grafana:3000
       ├── Host = prometheus.kadhem.kaascloud...   → forward vers prometheus:9090
       ├── Host = alertmanager.kadhem.kaascloud... → forward vers alertmanager:9093
       ├── Host = blackbox.kadhem.kaascloud...     → forward vers blackbox:9115
       ├── Host = loki.kadhem.kaascloud...         → forward vers loki:3100
       └── Host = inconnu                          → 404 Not Found
```

## Ou sont deployes les Ingress dans le code

Les templates sont dans :
```
roles/monitoring_stack/templates/Ingress/
├── ingress-grafana.yaml.j2
├── ingress-prometheus.yaml.j2
├── ingress-alertmanager.yaml.j2
├── ingress-blackbox.yaml.j2
└── ingress-loki.yaml.j2
```

Ils sont appliques dans `roles/monitoring_stack/tasks/main.yml` avec :

```yaml
- name: "Deploy Ingress resources"
  kubernetes.core.k8s:
    definition: "{{ lookup('template', item) | from_yaml }}"
  loop:
    - Ingress/ingress-grafana.yaml.j2
    - Ingress/ingress-prometheus.yaml.j2
    - Ingress/ingress-alertmanager.yaml.j2
    - Ingress/ingress-blackbox.yaml.j2
    - Ingress/ingress-loki.yaml.j2
```

## Commandes utiles

```bash
# Voir toutes les ressources Ingress
k8s kubectl get ingress -n observability

# Voir le detail d'un Ingress
k8s kubectl describe ingress grafana-ingress -n observability

# Voir les logs du controller (pour debugger le routage)
k8s kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## Resume

- Un reverse proxy redirige le trafic vers les bons services selon le nom de domaine
- Chaque service a sa propre ressource Ingress (fichier YAML)
- `ingressClassName: nginx` lie la regle a ingress-nginx
- Le routage se fait par `host` (domaine) + `backend` (service + port)
- Tous les services passent par le meme port 443
