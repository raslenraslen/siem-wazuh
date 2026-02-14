# Ingress Nginx - HostNetwork (Acces Direct a la VM)

## Le probleme

Quand on deploie ingress-nginx dans Kubernetes, il cree un **Service** pour recevoir le trafic. Il y a 3 types de Service possibles :

| Type | Comment ca marche | Quand l'utiliser |
|------|-------------------|-----------------|
| `LoadBalancer` | Le cloud provider (AWS, GCP, Azure) assigne une IP publique | Clusters cloud |
| `NodePort` | Kubernetes ouvre un port aleatoire entre 30000-32767 | Developpement |
| `ClusterIP` | Accessible seulement a l'interieur du cluster | Jamais pour un Ingress |

Notre projet tourne sur **Aurora Cloud** (bare-metal / VM). Il n'y a pas de cloud provider qui fournit un LoadBalancer. Le manifest qu'on utilise (`provider/baremetal/deploy.yaml`) cree un Service de type **NodePort**.

Le probleme avec NodePort :

```
https://IP_VM:31234    ← port aleatoire, pas standard
https://IP_VM:443      ← ce qu'on veut
```

Le port 31234 (ou autre port entre 30000-32767) n'est pas pratique. On veut acceder aux services sur les ports **standard** 80 et 443.

## La solution : hostNetwork

On **patch** le deployment ingress-nginx pour activer `hostNetwork: true`. Ca veut dire : le pod utilise **directement le reseau de la VM** au lieu du reseau interne Kubernetes.

```yaml
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
```

### Avant (sans hostNetwork)

```
                    Reseau K8s (interne)
                    ┌─────────────────────┐
VM (port 31234) ───>│ Service NodePort     │──> Pod ingress-nginx (port 443)
                    └─────────────────────┘
```

Le pod ecoute sur le port 443 a l'interieur du reseau K8s. Le NodePort expose ca sur un port aleatoire de la VM.

### Apres (avec hostNetwork)

```
VM (port 80, 443) ──> Pod ingress-nginx (directement sur le reseau de la VM)
```

Le pod ecoute **directement** sur les ports 80 et 443 de la VM. Pas de NodePort, pas d'intermediaire.

## Comment c'est fait dans notre code

Dans `roles/ingress_nginx/tasks/main.yml` :

```yaml
# Verifier si hostNetwork est deja active
- name: "Check if hostNetwork is already enabled"
  ansible.builtin.command: >
    k8s kubectl get deployment ingress-nginx-controller
    -n ingress-nginx -o jsonpath='{.spec.template.spec.hostNetwork}'
  register: hostnetwork_check

# Patcher si pas encore active
- name: "Patch Ingress Controller to enable hostNetwork"
  ansible.builtin.command: >
    k8s kubectl patch deployment ingress-nginx-controller
    -n ingress-nginx
    --type=strategic
    -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
  when: hostnetwork_check.stdout != 'true'
```

On verifie d'abord si c'est deja active (idempotence), puis on patch si necessaire.

## Pourquoi `dnsPolicy: ClusterFirstWithHostNet`

Quand un pod est en `hostNetwork: true`, il utilise le DNS de la VM (pas celui de Kubernetes). Ca veut dire qu'il ne peut plus resoudre les noms de services Kubernetes comme `grafana.observability.svc.cluster.local`.

`ClusterFirstWithHostNet` corrige ca : le pod utilise d'abord le DNS Kubernetes, puis le DNS de la VM si le nom n'est pas trouve dans K8s.

```
Sans ClusterFirstWithHostNet :
  grafana.observability.svc.cluster.local → ECHEC (DNS de la VM ne connait pas)

Avec ClusterFirstWithHostNet :
  grafana.observability.svc.cluster.local → DNS K8s → OK, resolu
  google.com                               → DNS K8s → pas trouve → DNS VM → OK
```

C'est necessaire car ingress-nginx doit pouvoir resoudre les noms des services K8s pour router le trafic.

## Consequences de hostNetwork

### Ports occupes sur la VM

Le pod ingress-nginx occupe les ports **80** et **443** de la VM. Si un autre service essaie d'utiliser ces ports sur la VM, il y aura un conflit.

C'est pour ca que Wazuh Dashboard (port 9443) et l'API Wazuh (port 55000) n'ont pas de conflit — ils utilisent des ports differents.

### Un seul pod par VM

Avec `hostNetwork: true`, on ne peut pas avoir **2 pods ingress-nginx sur la meme VM** (conflit de ports). Ce n'est pas un probleme pour nous car on a un seul noeud observability.

### Acces direct

L'avantage principal : on accede aux services directement via l'IP de la VM sur le port 443 :

```
https://100.100.206.61          ← IP Tailscale de la VM
https://grafana.kadhem.kaascloud.auroraiq.cloud  ← si le DNS pointe vers cette IP
```

## Les 3 alternatives a hostNetwork

On utilise `hostNetwork`, mais il existe d'autres solutions :

| Solution | Description | Inconvenient |
|----------|-------------|-------------|
| **hostNetwork** (notre choix) | Pod utilise le reseau de la VM | Ports 80/443 occupes sur la VM |
| **NodePort** (defaut baremetal) | Port aleatoire 30000-32767 | Port non standard |
| **MetalLB** | LoadBalancer pour bare-metal, assigne une IP virtuelle | Plus complexe a configurer |
| **hostPort** | Expose uniquement les ports 80/443 (sans tout le reseau) | Similar a hostNetwork mais plus limite |

`hostNetwork` est la solution la plus simple pour notre cas (single node, pas de LoadBalancer cloud).

## Commandes utiles

```bash
# Verifier que hostNetwork est active
k8s kubectl get deployment ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.template.spec.hostNetwork}'
# Doit afficher : true

# Verifier que le pod tourne bien
k8s kubectl get pods -n ingress-nginx -o wide
# La colonne IP doit etre l'IP de la VM (pas une IP de pod 10.x.x.x)

# Verifier que les ports 80 et 443 sont ecoutes sur la VM
sudo ss -tlnp | grep -E ':80 |:443 '

# Tester l'acces direct
curl -sk https://IP_DE_LA_VM
# Doit repondre (404 ou redirect, mais pas "connection refused")
```

## Resume

- Sans hostNetwork, ingress-nginx est accessible sur un port aleatoire (30000-32767) via NodePort
- `hostNetwork: true` permet au pod d'ecouter directement sur les ports 80 et 443 de la VM
- `dnsPolicy: ClusterFirstWithHostNet` est necessaire pour que le pod puisse resoudre les noms de services K8s
- C'est la solution la plus simple pour les environnements bare-metal / VM sans LoadBalancer cloud
- Les ports 80 et 443 de la VM sont occupes par ingress-nginx
