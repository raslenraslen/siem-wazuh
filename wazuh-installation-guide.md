# Wazuh 4.7 - Guide d'Installation

Guide simple pour installer Wazuh sur n'importe quel projet.
Tu as besoin de **2 types de machines** : une VM serveur (tout Wazuh dessus) et des VM clients (les agents).

---

## C'est quoi Wazuh ?

Wazuh = un **SIEM** (Security Information and Event Management) open-source.
Il surveille la securite de tes serveurs et te previent en temps reel.

Exemples de ce qu'il detecte :
- Modification d'un fichier critique (`/etc/passwd`, `/etc/shadow`)
- Tentatives de connexion SSH echouees (brute-force)
- Packages avec des failles de securite connues (CVE)
- Connexions root, escalation de privileges

> **Prometheus/Grafana** = monitoring **infrastructure** (CPU, RAM, disk)
> **Wazuh** = monitoring **securite** (intrusions, failles, compliance)
> Les deux sont complementaires.

---

## Architecture (comment ca marche)

Tout tient en un schema simple :

```
VM SERVEUR (monitoring)                    VM CLIENTS
┌──────────────────────────┐
│                          │
│  Indexer (base de donnees)│◄──── Dashboard (interface web, port 9443)
│       ▲                  │
│       │                  │
│  Filebeat (transfert)    │
│       ▲                  │
│       │                  │
│  Manager (analyse)       │◄──── Agent VM 1  (port 1514)
│                          │◄──── Agent VM 2  (port 1514)
│                          │◄──── Agent VM 3  (port 1514)
└──────────────────────────┘
```

**4 composants sur la VM serveur :**

| Composant | C'est quoi ? | Port |
|-----------|-------------|------|
| **Indexer** | Base de donnees (OpenSearch). Stocke toutes les alertes | 9200 |
| **Manager** | Recoit les donnees des agents, analyse, genere des alertes | 1514, 55000 |
| **Filebeat** | Transfere les alertes du Manager vers l'Indexer | aucun |
| **Dashboard** | Interface web pour visualiser les alertes et gerer les agents | 9443 |

**1 composant sur chaque VM client :**

| Composant | C'est quoi ? | Port |
|-----------|-------------|------|
| **Agent** | Collecte les logs et events de securite, envoie au Manager | 1514 (sortant) |

**Ordre d'installation :** Indexer → Manager → Filebeat → Dashboard → Agents

---

## Prerequisites

**VM Serveur :**
- Ubuntu 22.04 ou 24.04
- Minimum 4 vCPU, 8 GB RAM, 50 GB disk
- Packages : `sudo apt update && sudo apt install -y gnupg apt-transport-https curl`

**VM Client :**
- Ubuntu 22.04 ou 24.04
- Minimum 1 vCPU, 1 GB RAM
- Memes packages

**Reseau :**
- Les clients doivent pouvoir joindre le serveur sur le port **1514/TCP**
- Le Dashboard est accessible sur le port **9443/TCP**

---

## Etape 1 : Preparer la VM Serveur

> Toutes les commandes de cette etape se font sur la **VM Serveur**

### 1.1 Configurer le kernel

OpenSearch (la base de donnees) a besoin de plus de memoire virtuelle :

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

> Sans ca, l'Indexer crash au demarrage.

### 1.2 Ajouter le depot Wazuh

```bash
curl -sO https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg GPG-KEY-WAZUH
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

### 1.3 Generer les certificats SSL

Wazuh utilise HTTPS partout. On genere les certificats une seule fois :

```bash
sudo mkdir -p /opt/wazuh/certs && cd /opt/wazuh/certs
sudo curl -sO https://packages.wazuh.com/4.7/wazuh-certs-tool.sh
sudo chmod 700 wazuh-certs-tool.sh
```

Creer le fichier de config des certificats :

```bash
sudo tee /opt/wazuh/certs/config.yml << 'EOF'
nodes:
  indexer:
    - name: wazuh-indexer
      ip: 127.0.0.1
  server:
    - name: wazuh-manager
      ip: 127.0.0.1
  dashboard:
    - name: wazuh-dashboard
      ip: 127.0.0.1
EOF
```

> On met `127.0.0.1` partout car tout est sur la meme machine.

Generer et archiver :

```bash
sudo bash /opt/wazuh/certs/wazuh-certs-tool.sh -A
cd /opt/wazuh/certs
sudo tar -cvf wazuh-certificates.tar -C wazuh-certificates .
```

---

## Etape 2 : Installer l'Indexer

> Toujours sur la **VM Serveur**

L'Indexer est la base de donnees (OpenSearch). Il stocke toutes les alertes.

### 2.1 Installer

```bash
sudo apt install -y wazuh-indexer=4.7.2-1
```

### 2.2 Configurer

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Remplacer **tout** le contenu par :

```yaml
network.host: 127.0.0.1
node.name: wazuh-indexer
cluster.name: wazuh-cluster
cluster.initial_master_nodes:
  - wazuh-indexer

path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

discovery.type: single-node

# --- TLS Transport (communication interne) ---
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/wazuh-indexer.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/wazuh-indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

# --- TLS HTTP (API) ---
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/wazuh-indexer.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/wazuh-indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem

plugins.security.authcz.admin_dn:
  - "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.nodes_dn:
  - "CN=wazuh-indexer,OU=Wazuh,O=Wazuh,L=California,C=US"

plugins.security.allow_default_init_securityindex: true
plugins.security.allow_unsafe_democertificates: false

plugins.security.restapi.roles_enabled:
  - "all_access"
  - "security_rest_api_access"

compatibility.override_main_response_version: true
```

> Tu n'as rien a changer dans cette config. Copie-colle tel quel.

### 2.3 Mettre les certificats

```bash
sudo mkdir -p /etc/wazuh-indexer/certs
cd /opt/wazuh/certs
sudo tar -xf wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ \
  ./wazuh-indexer.pem ./wazuh-indexer-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs/
sudo chmod 400 /etc/wazuh-indexer/certs/*
sudo chmod 500 /etc/wazuh-indexer/certs
```

### 2.4 Configurer la memoire

```bash
sudo tee /etc/wazuh-indexer/jvm.options.d/heap.options << 'EOF'
-Xms1g
-Xmx1g
EOF
```

> Regle simple : mets la moitie de ta RAM disponible, max 2g. Les deux valeurs doivent etre identiques.

### 2.5 Demarrer et verifier

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
```

Attendre 30 secondes, puis :

```bash
# Verifier que le service tourne
sudo systemctl status wazuh-indexer

# Verifier que le port 9200 est ouvert
sudo ss -tlnp | grep 9200
```

### 2.6 Initialiser la securite

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

### 2.7 Tester

```bash
curl -sk -u admin:admin https://127.0.0.1:9200
```

Tu dois voir un JSON avec `"cluster_name" : "wazuh-cluster"`. Si oui, l'Indexer marche.

### 2.8 Changer le mot de passe admin

Le password par defaut `admin` n'est pas securise :

```bash
sudo curl -sO https://packages.wazuh.com/4.7/wazuh-passwords-tool.sh
sudo chmod 700 wazuh-passwords-tool.sh
sudo bash wazuh-passwords-tool.sh -u admin -p 'TON_PASSWORD_ICI'
```

Verifier :

```bash
curl -sk -u admin:TON_PASSWORD_ICI https://127.0.0.1:9200
```

> **IMPORTANT** : Note ce password ! Tu en auras besoin pour Filebeat et le Dashboard.

---

## Etape 3 : Installer le Manager

> Toujours sur la **VM Serveur**

Le Manager recoit les donnees des agents, les analyse avec ses regles de detection, et genere les alertes.

### 3.1 Installer et demarrer

```bash
sudo apt install -y wazuh-manager=4.7.2-1
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
```

### 3.2 Verifier

```bash
# Le service doit etre active
sudo systemctl status wazuh-manager

# 3 ports doivent etre ouverts
sudo ss -tlnp | grep -E '1514|1515|55000'
```

- Port 1514 = communication avec les agents
- Port 1515 = enregistrement des agents
- Port 55000 = API REST

### 3.3 Tester l'API

```bash
curl -sk -u wazuh:wazuh -X POST https://127.0.0.1:55000/security/user/authenticate
```

Tu dois voir un JSON avec un `"token"`. Si oui, l'API marche.

### 3.4 Changer le mot de passe de l'API

```bash
# Recuperer un token
TOKEN=$(curl -sk -u wazuh:wazuh -X POST https://127.0.0.1:55000/security/user/authenticate | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['data']['token'])")

# Changer le password
curl -sk -X PUT https://127.0.0.1:55000/security/users/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password": "TON_API_PASSWORD_ICI"}'
```

Verifier :

```bash
curl -sk -u wazuh:TON_API_PASSWORD_ICI -X POST https://127.0.0.1:55000/security/user/authenticate
```

> **IMPORTANT** : Note ce password aussi ! Le Dashboard en a besoin.

---

## Etape 4 : Installer Filebeat

> Toujours sur la **VM Serveur**

Filebeat transfere les alertes du Manager vers l'Indexer.

> Sans Filebeat, le Manager genere des alertes mais elles ne sont pas visibles dans le Dashboard.

### 4.1 Installer

```bash
sudo apt install -y filebeat=7.10.2
```

### 4.2 Configurer

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Remplacer **tout** le contenu par :

```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: false

setup.template.json.enabled: true
setup.template.json.path: "/etc/filebeat/wazuh-template.json"
setup.template.json.name: "wazuh"
setup.template.json.overwrite: true
setup.template.settings:
  index.number_of_shards: 1
  index.number_of_replicas: 0

output.elasticsearch:
  hosts: ["https://127.0.0.1:9200"]
  protocol: https
  username: ${username}
  password: ${password}
  ssl.certificate_authorities:
    - /etc/filebeat/certs/root-ca.pem
  ssl.certificate: /etc/filebeat/certs/wazuh-manager.pem
  ssl.key: /etc/filebeat/certs/wazuh-manager-key.pem
```

> `${username}` et `${password}` ne sont PAS des variables Ansible. C'est Filebeat qui les lit depuis son keystore (etape 4.4).

### 4.3 Installer le module Wazuh + template

```bash
# Module Wazuh
sudo curl -so /tmp/wazuh-filebeat-module.tar.gz \
  https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.4.tar.gz
sudo tar -xzf /tmp/wazuh-filebeat-module.tar.gz -C /usr/share/filebeat/module
rm /tmp/wazuh-filebeat-module.tar.gz

# Template d'index
sudo curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.7.2/extensions/elasticsearch/7.x/wazuh-template.json
```

### 4.4 Configurer les credentials (keystore)

Le keystore garde les passwords en securite (pas en clair dans le fichier de config) :

```bash
sudo filebeat keystore create --force
echo 'admin' | sudo filebeat keystore add username --stdin --force
echo 'TON_PASSWORD_ICI' | sudo filebeat keystore add password --stdin --force
```

> Utilise le password de l'Indexer (etape 2.8).

### 4.5 Mettre les certificats

```bash
sudo mkdir -p /etc/filebeat/certs
cd /opt/wazuh/certs
sudo tar -xf wazuh-certificates.tar -C /etc/filebeat/certs/ \
  ./wazuh-manager.pem ./wazuh-manager-key.pem ./root-ca.pem
sudo chmod 400 /etc/filebeat/certs/*
sudo chmod 500 /etc/filebeat/certs
```

### 4.6 Demarrer et tester

```bash
sudo systemctl daemon-reload
sudo systemctl enable filebeat
sudo systemctl start filebeat

# Tester la connexion Filebeat → Indexer
sudo filebeat test output
```

Tu dois voir `talk to server... OK`. Si oui, Filebeat marche.

---

## Etape 5 : Installer le Dashboard

> Toujours sur la **VM Serveur**

Le Dashboard est l'interface web pour visualiser les alertes et gerer les agents.

### 5.1 Installer

```bash
sudo apt install -y wazuh-dashboard=4.7.2-1
```

### 5.2 Configurer

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Remplacer le contenu par :

```yaml
server.host: 0.0.0.0
server.port: 9443
server.ssl.enabled: true
server.ssl.key: "/etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/wazuh-dashboard.pem"

opensearch.hosts: ["https://127.0.0.1:9200"]
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersAllowlist: ["securitytenant", "Authorization"]
opensearch.security.multitenancy.enabled: false
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]

opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
opensearch_security.cookie.secure: true
```

> `server.host: 0.0.0.0` = accessible depuis l'exterieur. C'est voulu.

### 5.3 Mettre les certificats

```bash
sudo mkdir -p /etc/wazuh-dashboard/certs
cd /opt/wazuh/certs
sudo tar -xf wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ \
  ./wazuh-dashboard.pem ./wazuh-dashboard-key.pem ./root-ca.pem
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs/
sudo chmod 400 /etc/wazuh-dashboard/certs/*
sudo chmod 500 /etc/wazuh-dashboard/certs
```

### 5.4 Connecter le Dashboard a l'API du Manager

```bash
sudo nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

```yaml
hosts:
  - default:
      url: https://127.0.0.1
      port: 55000
      username: wazuh
      password: TON_API_PASSWORD_ICI
      run_as: false
```

> Utilise le password de l'API (etape 3.4).

### 5.5 Demarrer

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
```

> Le Dashboard met **1-2 minutes** a demarrer. Sois patient.

### 5.6 Se connecter

Ouvre dans ton navigateur : `https://IP_DE_TA_VM:9443`

- **Username** : `admin`
- **Password** : le password de l'Indexer (etape 2.8)

> Le navigateur va dire "certificat non securise" — c'est normal (self-signed). Clique "Avance" → "Continuer".

### 5.7 Verrouiller les versions

Pour eviter qu'une mise a jour casse tout :

```bash
sudo apt-mark hold wazuh-indexer wazuh-manager filebeat wazuh-dashboard
```

---

## Etape 6 : Installer l'Agent (sur chaque VM client)

> **ATTENTION** : Cette etape se fait sur les **VM Clients**, PAS sur la VM Serveur

L'Agent est un service leger (~50 MB RAM) qui collecte les logs et events de securite de la machine et les envoie au Manager.

### 6.1 Ajouter le depot Wazuh

```bash
curl -sO https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg GPG-KEY-WAZUH
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

### 6.2 Installer

```bash
WAZUH_MANAGER="IP_DU_SERVEUR" sudo apt install -y wazuh-agent=4.7.2-1
```

> Remplace `IP_DU_SERVEUR` par l'IP de ta VM Serveur (IP Tailscale si tu utilises Tailscale).

### 6.3 Verifier la config

```bash
sudo grep -A3 '<server>' /var/ossec/etc/ossec.conf
```

Tu dois voir :

```xml
<server>
  <address>IP_DU_SERVEUR</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
```

Si l'IP est mauvaise, edite : `sudo nano /var/ossec/etc/ossec.conf`

### 6.4 Demarrer

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 6.5 Verifier la connexion

Attendre 30 secondes puis :

```bash
sudo grep "Connected to the server" /var/ossec/logs/ossec.log
```

Tu dois voir : `Connected to the server (IP_DU_SERVEUR:1514/tcp)`

Si ca ne marche pas :

```bash
# Tester la connectivite reseau
nc -zv IP_DU_SERVEUR 1514
# Si ca echoue → probleme reseau ou firewall
```

### 6.6 Verrouiller la version

```bash
sudo apt-mark hold wazuh-agent
```

---

## Verification Finale

### Checklist rapide

| Quoi | Commande (sur VM serveur) | OK si |
|------|--------------------------|-------|
| Indexer | `sudo systemctl status wazuh-indexer` | active (running) |
| Manager | `sudo systemctl status wazuh-manager` | active (running) |
| Filebeat | `sudo systemctl status filebeat` | active (running) |
| Dashboard | `sudo systemctl status wazuh-dashboard` | active (running) |
| Port 9200 | `sudo ss -tlnp \| grep 9200` | LISTEN |
| Port 1514 | `sudo ss -tlnp \| grep 1514` | LISTEN |
| Port 9443 | `sudo ss -tlnp \| grep 9443` | LISTEN |
| Filebeat | `sudo filebeat test output` | talk to server... OK |
| Dashboard | Browser `https://IP:9443` | Page de login |

| Quoi | Commande (sur VM client) | OK si |
|------|--------------------------|-------|
| Agent | `sudo systemctl status wazuh-agent` | active (running) |
| Connexion | `grep Connected /var/ossec/logs/ossec.log` | Connected to the server |

### Verifier les agents depuis l'API

```bash
TOKEN=$(curl -sk -u wazuh:TON_API_PASSWORD_ICI -X POST \
  https://127.0.0.1:55000/security/user/authenticate | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['data']['token'])")

curl -sk -X GET "https://127.0.0.1:55000/agents?pretty" \
  -H "Authorization: Bearer $TOKEN"
```

Les agents doivent avoir `status: active`.

---

## Depannage

### L'Indexer ne demarre pas

```bash
sudo journalctl -u wazuh-indexer -n 50 --no-pager

# Cause #1 : max_map_count pas configure
sudo sysctl vm.max_map_count   # doit etre 262144

# Cause #2 : permissions des certificats
ls -la /etc/wazuh-indexer/certs/   # doit appartenir a wazuh-indexer
```

### L'API ne repond pas (port 55000)

```bash
sudo journalctl -u wazuh-manager -n 50 --no-pager
sudo cat /var/ossec/logs/api.log | tail -20

# L'API peut mettre 2-3 minutes a demarrer. Patiente.
```

### Filebeat ne se connecte pas

```bash
sudo filebeat test output
sudo filebeat keystore list   # doit afficher username et password
```

### L'Agent ne se connecte pas

```bash
# Sur le client
sudo cat /var/ossec/logs/ossec.log | tail -30
nc -zv IP_DU_SERVEUR 1514   # tester la connectivite
```

### Le Dashboard n'affiche rien

```bash
# Verifier que Filebeat a bien envoye des donnees
curl -sk -u admin:TON_PASSWORD_ICI https://127.0.0.1:9200/_cat/indices | grep wazuh
# Tu dois voir des index "wazuh-alerts-*"
```

---

## Fichiers importants (reference)

| Fichier | C'est quoi |
|---------|-----------|
| `/etc/wazuh-indexer/opensearch.yml` | Config de l'Indexer |
| `/etc/wazuh-indexer/jvm.options.d/heap.options` | Memoire de l'Indexer |
| `/var/ossec/etc/ossec.conf` | Config du Manager (ou de l'Agent sur le client) |
| `/var/ossec/logs/ossec.log` | Logs du Manager (ou de l'Agent) |
| `/var/ossec/logs/alerts/alerts.json` | Alertes generees par le Manager |
| `/etc/filebeat/filebeat.yml` | Config de Filebeat |
| `/etc/wazuh-dashboard/opensearch_dashboards.yml` | Config du Dashboard |
| `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml` | Connexion Dashboard → API |
