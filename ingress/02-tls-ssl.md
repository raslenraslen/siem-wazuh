# Ingress Nginx - TLS / SSL (HTTPS)

## Le probleme

Sans HTTPS, le trafic entre ton navigateur et le serveur passe **en clair**. Ca veut dire :
- Les passwords que tu tapes sont visibles sur le reseau
- Les donnees affichees (dashboards Grafana, metriques Prometheus) sont lisibles
- Un attaquant sur le meme reseau peut intercepter tout

## C'est quoi TLS/SSL

TLS (Transport Layer Security) est un protocole qui **chiffre** la communication entre le client et le serveur. SSL est l'ancien nom (SSL → TLS), mais on utilise souvent les deux termes.

HTTPS = HTTP + TLS. Le `S` veut dire "Secure".

Pour que TLS fonctionne, le serveur a besoin d'un **certificat TLS**. Ce certificat contient :
- La cle publique du serveur (pour chiffrer)
- Le nom de domaine pour lequel il est valide
- Qui l'a signe (l'autorite de certification)

## Comment ca marche dans notre projet

On utilise **2 composants** ensemble :

| Composant | Role |
|-----------|------|
| **cert-manager** | Genere et renouvelle les certificats TLS automatiquement |
| **ingress-nginx** | Utilise les certificats pour servir du HTTPS |

Le flux :

```
1. On deploie une ressource Ingress avec l'annotation cert-manager
2. cert-manager detecte l'annotation
3. cert-manager genere un certificat TLS pour le domaine
4. cert-manager stocke le certificat dans un Secret Kubernetes
5. ingress-nginx lit le Secret et active HTTPS
```

## La configuration dans les Ingress

Chaque Ingress a 2 parties pour le TLS :

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-cluster-issuer"   # (1)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"               # (2)

spec:
  tls:
    - hosts:
        - grafana.kadhem.kaascloud.auroraiq.cloud                  # (3)
      secretName: grafana-tls-secret                                # (4)
```

| # | Champ | Ce que ca fait |
|---|-------|---------------|
| 1 | `cert-manager.io/cluster-issuer` | Dit a cert-manager : "genere un certificat avec ce ClusterIssuer" |
| 2 | `ssl-redirect: "true"` | Si quelqu'un arrive en HTTP, le rediriger vers HTTPS |
| 3 | `tls.hosts` | Le domaine pour lequel le certificat est genere |
| 4 | `tls.secretName` | Le nom du Secret Kubernetes ou le certificat est stocke |

## Les 5 certificats de notre projet

| Service | Secret | Domaine |
|---------|--------|---------|
| Grafana | `grafana-tls-secret` | `grafana.CLIENT.kaascloud.auroraiq.cloud` |
| Prometheus | `prometheus-tls-secret` | `prometheus.CLIENT.kaascloud.auroraiq.cloud` |
| Alertmanager | `alertmanager-tls-secret` | `alertmanager.CLIENT.kaascloud.auroraiq.cloud` |
| Blackbox | `blackbox-tls-secret` | `blackbox.CLIENT.kaascloud.auroraiq.cloud` |
| Loki | `loki-tls-secret` | `loki.CLIENT.kaascloud.auroraiq.cloud` |

Chaque service a **son propre certificat** stocke dans un Secret separe.

## Le ClusterIssuer

Le ClusterIssuer definit **comment** les certificats sont generes. C'est configure dans `roles/cert_manager/templates/cluster-issuer.yaml.j2`.

Notre projet supporte 3 types :

### Type 1 : selfsigned (utilise actuellement)

```yaml
spec:
  selfSigned: {}
```

- Le certificat est signe par lui-meme
- **Avantage** : fonctionne sans configuration externe, pas besoin de domaine public
- **Inconvenient** : le navigateur affiche "connexion non securisee" (le certificat n'est pas reconnu par une autorite)
- **Le trafic est quand meme chiffre**, c'est juste le navigateur qui ne fait pas confiance au signataire

### Type 2 : acme (Let's Encrypt)

```yaml
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

- Let's Encrypt genere un certificat **gratuit** et **reconnu** par tous les navigateurs
- **Prerequis** : le domaine doit etre public et accessible depuis Internet
- `solvers.http01.ingress.class: nginx` → cert-manager utilise ingress-nginx pour prouver qu'on controle le domaine (challenge HTTP-01)
- Le certificat est renouvele automatiquement avant expiration

### Type 3 : ca (autorite interne)

```yaml
spec:
  ca:
    secretName: ca-key-pair
```

- Utilise une autorite de certification (CA) interne
- Pour les environnements d'entreprise qui ont leur propre CA

## La redirection HTTP → HTTPS

```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

Ce que ca fait concretement :

```
Client fait http://grafana.kadhem.kaascloud.auroraiq.cloud
       │
       ▼
ingress-nginx recoit sur port 80
       │
       ▼
Repond : 308 Redirect → https://grafana.kadhem.kaascloud.auroraiq.cloud
       │
       ▼
Le navigateur refait la requete en HTTPS (port 443)
```

Le code HTTP 308 (Permanent Redirect) dit au navigateur de toujours utiliser HTTPS a l'avenir.

## Ce qui se passe quand ingress-nginx recoit une requete HTTPS

```
1. Le navigateur se connecte au port 443
2. Handshake TLS :
   a. ingress-nginx envoie le certificat (depuis le Secret)
   b. Le navigateur verifie le certificat
   c. Ils negocient une cle de chiffrement
3. La connexion est chiffree
4. ingress-nginx dechiffre, lit le Host header, route vers le bon service
5. Le trafic entre ingress-nginx et le service K8s est en clair (reseau interne)
```

Le point 5 est important : le TLS est **termine** au niveau d'ingress-nginx. Entre ingress-nginx et Grafana/Prometheus/etc, le trafic est en clair. C'est normal car c'est le reseau interne du cluster.

## Commandes utiles

```bash
# Voir les certificats generes par cert-manager
k8s kubectl get certificates -n observability

# Voir le detail d'un certificat
k8s kubectl describe certificate grafana-tls-secret -n observability

# Verifier qu'un Secret TLS existe
k8s kubectl get secret grafana-tls-secret -n observability

# Voir le ClusterIssuer
k8s kubectl get clusterissuer
k8s kubectl describe clusterissuer selfsigned-cluster-issuer

# Tester HTTPS depuis la VM
curl -sk https://grafana.kadhem.kaascloud.auroraiq.cloud
# -s = silencieux, -k = ignorer les erreurs de certificat (self-signed)
```

## Fichiers lies

| Fichier | Role |
|---------|------|
| `roles/cert_manager/tasks/main.yml` | Installe cert-manager et cree le ClusterIssuer |
| `roles/cert_manager/templates/cluster-issuer.yaml.j2` | Template du ClusterIssuer |
| `group_vars/all/cert_manager.yml` | Variables cert-manager (version, type d'issuer, etc.) |
| `roles/monitoring_stack/templates/Ingress/*.yaml.j2` | Les 5 Ingress avec les annotations TLS |

## Resume

- TLS chiffre le trafic entre le navigateur et le serveur
- cert-manager genere les certificats automatiquement grace a l'annotation `cert-manager.io/cluster-issuer`
- Les certificats sont stockes dans des Secrets Kubernetes
- ingress-nginx utilise ces Secrets pour servir du HTTPS
- `ssl-redirect: "true"` force la redirection HTTP → HTTPS
- Actuellement on utilise des certificats self-signed (le navigateur affiche un warning mais le trafic est chiffre)
- Le TLS est termine au niveau d'ingress-nginx (TLS termination)
