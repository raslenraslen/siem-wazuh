# Ingress Nginx - Basic Authentication

## Le probleme

Grafana a son propre systeme de login (username/password dans l'interface). Mais **Prometheus, Alertmanager, Blackbox et Loki** n'ont **aucune authentification**. Si quelqu'un connait l'URL, il peut acceder aux donnees directement.

Ca veut dire :
- N'importe qui peut voir les metriques de tes serveurs (CPU, RAM, erreurs)
- N'importe qui peut voir les alertes en cours
- N'importe qui peut voir les logs (Loki)

## C'est quoi Basic Auth

Basic Auth est un mecanisme d'authentification standard du protocole HTTP (RFC 7617).

Quand Basic Auth est active :
1. Le navigateur affiche une popup qui demande username et password
2. Le navigateur envoie les credentials dans le header `Authorization`
3. Le serveur verifie les credentials
4. Si OK → acces autorise
5. Si NOK → erreur 401 Unauthorized

Le format du header :

```
Authorization: Basic base64(username:password)
```

Exemple : pour `admin:monpassword`, le header est :

```
Authorization: Basic YWRtaW46bW9ucGFzc3dvcmQ=
```

C'est encode en base64 (pas chiffre). C'est pourquoi Basic Auth doit **toujours** etre utilise avec HTTPS — sinon les credentials sont visibles en clair sur le reseau.

## Comment ca marche avec ingress-nginx

ingress-nginx gere le Basic Auth **avant** de transmettre la requete au service. Le service ne voit meme pas la requete si l'authentification echoue.

```
Requete HTTPS arrive
       │
       ▼
ingress-nginx verifie les credentials
       │
       ├── Credentials correctes → forward vers le service
       │
       └── Credentials incorrectes ou absentes → reponse 401 Unauthorized
```

Le service (Prometheus, Loki, etc.) ne gere rien. Toute la logique est dans ingress-nginx.

## Configuration dans notre projet

### Etape 1 : Creer les credentials (htpasswd)

Dans `roles/monitoring_stack/tasks/main.yml`, on genere un hash htpasswd :

```bash
htpasswd -nb 'admin' 'monpassword'
# Resultat : admin:$apr1$R2x5Q...$Hx7K...
```

`htpasswd` est un format standard (utilise par Apache httpd depuis 20+ ans). Le password est hashe, pas stocke en clair.

### Etape 2 : Creer le Secret Kubernetes

Le hash htpasswd est stocke dans un Secret Kubernetes :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
  namespace: observability
type: Opaque
data:
  auth: YWRtaW46JGFwcjEk...    # htpasswd encode en base64
```

Le champ doit s'appeler `auth`. C'est ce qu'ingress-nginx attend.

### Etape 3 : Activer Basic Auth dans l'Ingress

Sur chaque Ingress qui doit etre protege, on ajoute 3 annotations :

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
  nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Prometheus"
```

| Annotation | Ce que ca fait |
|------------|---------------|
| `auth-type: basic` | Active le Basic Auth sur cet Ingress |
| `auth-secret: basic-auth-secret` | Nom du Secret qui contient les credentials |
| `auth-realm` | Message affiche dans la popup du navigateur |

## Quels services sont proteges dans notre projet

| Service | Basic Auth | Raison |
|---------|-----------|--------|
| **Grafana** | Non | Grafana a son propre systeme de login (admin/password dans l'interface) |
| **Prometheus** | Oui | Pas d'authentification native |
| **Alertmanager** | Oui | Pas d'authentification native |
| **Blackbox** | Oui | Pas d'authentification native |
| **Loki** | Oui | Pas d'authentification native |

Les 4 services proteges utilisent le **meme Secret** (`basic-auth-secret`) donc les memes credentials.

## D'ou viennent les credentials

Les credentials sont dans le **vault Ansible** (`group_vars/all/vault.yml`) :

```yaml
vault_basic_auth_user: "admin"
vault_basic_auth_password: "le_vrai_password"
```

Le playbook lit ces variables pour generer le hash htpasswd et creer le Secret Kubernetes. Les credentials ne sont jamais ecrites en clair dans les fichiers du projet (sauf dans le vault qui est chiffre).

## Comparaison Grafana vs Basic Auth

| | Grafana Login | Basic Auth (ingress-nginx) |
|---|---|---|
| Ou | Dans l'interface Grafana | Popup du navigateur |
| Gere par | Grafana | ingress-nginx |
| Utilisateurs multiples | Oui (admin, viewer, etc.) | Possible mais un seul Secret par Ingress |
| Roles/permissions | Oui | Non (juste oui/non) |
| Session | Cookie de session | Header envoye a chaque requete |

Basic Auth est simple : soit tu as acces, soit tu n'as pas acces. Pas de roles, pas de permissions, pas de gestion d'utilisateurs avancee.

## Commandes utiles

```bash
# Verifier que le Secret basic-auth-secret existe
k8s kubectl get secret basic-auth-secret -n observability

# Voir le contenu du Secret (decode base64)
k8s kubectl get secret basic-auth-secret -n observability -o jsonpath='{.data.auth}' | base64 -d

# Tester l'acces sans credentials (doit retourner 401)
curl -sk https://prometheus.kadhem.kaascloud.auroraiq.cloud
# Reponse : 401 Authorization Required

# Tester l'acces avec credentials (doit retourner 200)
curl -sk -u admin:monpassword https://prometheus.kadhem.kaascloud.auroraiq.cloud
```

## Fichiers lies

| Fichier | Role |
|---------|------|
| `roles/monitoring_stack/tasks/main.yml` | Cree le Secret basic-auth-secret |
| `roles/monitoring_stack/templates/Ingress/ingress-prometheus.yaml.j2` | Annotations Basic Auth |
| `roles/monitoring_stack/templates/Ingress/ingress-alertmanager.yaml.j2` | Annotations Basic Auth |
| `roles/monitoring_stack/templates/Ingress/ingress-blackbox.yaml.j2` | Annotations Basic Auth |
| `roles/monitoring_stack/templates/Ingress/ingress-loki.yaml.j2` | Annotations Basic Auth |
| `group_vars/all/vault.yml` | Credentials (chiffre) |

## Resume

- Basic Auth protege les services qui n'ont pas d'authentification native
- ingress-nginx gere l'authentification avant de transmettre au service
- Les credentials sont stockees dans un Secret Kubernetes au format htpasswd
- Les credentials viennent du vault Ansible (chiffre)
- 4 services proteges (Prometheus, Alertmanager, Blackbox, Loki), Grafana non car il a son propre login
- Basic Auth doit toujours etre utilise avec HTTPS (sinon credentials visibles en clair)
