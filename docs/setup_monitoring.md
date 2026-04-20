# Mise en Place de l'Observabilite : Loki, Promtail et Grafana (SecOps01)

Ce document decrit l'installation et la configuration de la solution de centralisation des logs et de supervision sur la machine **secops01** (192.168.100.100), situee dans la zone **LAN**. L'objectif est d'obtenir une visibilite en temps reel sur l'etat et la securite de l'infrastructure.

## 1. Architecture de la Pile de Supervision

La pile retenue est la suite Grafana LGTM (Loki, Grafana, Tempo, Mimir), reduite ici a ses composants essentiels pour la gestion des logs :

- **Promtail** : Agent leger deploye sur chaque machine cliente (DMZ et LAN) responsable de la collecte des fichiers de logs locaux (`/var/log/*.log`, `journald`).
- **Loki** : Serveur de stockage et d'indexation des logs. Il expose une API HTTP sur le port `3100`.
- **Grafana** : Interface web de visualisation permettant d'interroger Loki et de creer des tableaux de bord.

### 1.1 Flux de Donnees et Justification Securitaire

**Principe retenu : PUSH des logs depuis les clients vers le serveur.**

L'agent Promtail installe sur une machine cliente (par exemple `proxy01` en DMZ) initie la connexion vers `secops01:3100` pour y deposer ses journaux.

**Justification securitaire :**
Ce modele est strictement preferable a une approche "PULL" (ou le serveur irait chercher les logs sur les clients) pour deux raisons :

1.  **Reduction de la surface d'attaque du serveur central** : Dans un modele PUSH, `secops01` se contente d'ecouter passivement sur son port `3100`. Il n'a pas besoin de posseder des identifiants SSH ou des certificats clients pour acceder a chaque machine du parc. Une compromission de `secops01` ne donne donc pas immediatement un acces en lecture a tous les autres serveurs.
2.  **Contournement des restrictions de pare-feu** : La DMZ n'autorise pas les connexions entrantes depuis le LAN (politique `DROP` par defaut). Un modele PULL serait bloque par `fw-gw01`. Le modele PUSH respecte la topologie existante : c'est la DMZ qui "parle" au LAN sur un port specifique et autorise (`3100`).

## 2. Installation et Configuration de Loki (Serveur)

Loki est deploye sous forme de binaire statique pour une meilleure maitrise des versions.

### 2.1 Installation du Service

```bash
# Telechargement du binaire (exemple pour amd64)
sudo wget -O /usr/local/bin/loki-linux-amd64 https://github.com/grafana/loki/releases/download/v2.9.0/loki-linux-amd64.zip
sudo chmod +x /usr/local/bin/loki-linux-amd64
```

Creation du fichier de service systemd `/etc/systemd/system/loki.service` :

```ini
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/loki-linux-amd64 -config.file /etc/loki/loki-local-config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 2.2 Configuration de Loki (`/etc/loki/loki-local-config.yaml`)

```yaml
auth_enabled: true
server:
  http_listen_address: 0.0.0.0
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  # Retention des logs fixee a 31 jours (744h)
  retention_period: 744h
  allow_structured_metadata: false
```

**Demarrage du service :**

```bash
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
```

## 3. Installation et Configuration de Promtail (Clients)

Promtail est l'agent de collecte. Sa configuration est centralisee et peut etre copiee sur chaque machine cliente (`proxy01`, `infra01`, `mgmt01`, etc.).

### 3.1 Fichier de Configuration (`/etc/promtail/promtail-local-config.yaml`)

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.100.100:3100/loki/api/v1/push
    tenant_id: corp_prod
    basic_auth:
      username: admin
      password: motDePasseFort

scrape_configs:
  - job_name: system
    static_configs:
    - targets:
        - localhost
      labels:
        job: varlogs
        host: "192.168.100.100" # A personnaliser par machine
        __path__: /var/log/*log
```

**Points d'attention par machine :**
- Le label `host` doit etre ajuste a l'adresse IP reelle de la machine pour faciliter le filtrage dans Grafana.
- Le `tenant_id` et les identifiants `basic_auth` sont communs a tout le parc.

### 3.2 Service Promtail

Un fichier de service systemd similaire a celui de Loki est cree pour assurer l'execution permanente de l'agent.

## 4. Configuration de Grafana

Grafana est installe et configure pour interroger Loki en tant que source de donnees.

```bash
sudo apt install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Ajout de la source de donnees Loki :**
- URL : `http://192.168.100.100:3100`
- Authentification : Basic Auth (admin / Bayedame1476)

## 5. Dashboards de Securite (Exemples)

Les tableaux de bord suivants ont ete crees pour assurer une surveillance proactive :

| Dashboard               | Requete Loki associee                                                                 | Objectif de detection                                          |
| :---------------------- | :------------------------------------------------------------------------------------ | :------------------------------------------------------------- |
| **NGINX - HTTP Errors** | `{job="varlogs", host="172.16.50.20"} \|~ " 404 "`                                      | Detection de scans de repertoires ou de tentatives d'acces non autorises sur le proxy. |
| **SSH - Failed Logins** | `{job="varlogs"} \|~ "Failed password"`                                                | Identification des attaques par brute-force sur l'infrastructure. |
| **Firewall - Drops**    | `{host="192.168.100.1"} \|~ "DROP"`                                                   | Visualisation des paquets rejetes par `fw-gw01`, indicateur de scans reseau ou de regles mal configurees. |

## 6. Gestion d'Incident : Etude de Cas (Disque Saturé)

Cette section documente un incident reel survenu lors de la phase de developpement du laboratoire et les mesures correctives appliquees.

**Symptome :**
- Echec systematique des playbooks Ansible avec erreur `No space left on device`.
- Commande `df -h` affichant une utilisation disque de 100% sur la partition `/var` de `secops01`.

**Analyse :**
- La retention des logs Loki etait initialement illimitee.
- Les journaux systemd (`journald`) s'accumulaient egalement sans purge automatique.
- Le volume de logs genere par l'ensemble des machines du laboratoire a rapidement depasse la taille du disque virtuel alloue.

**Remediation Appliquee :**
1.  **Configuration de la retention Loki** : Ajout du parametre `retention_period: 744h` dans `loki-local-config.yaml` (environ 1 mois de conservation).
2.  **Nettoyage automatique de Systemd Journal** : Mise en place des parametres suivants dans `/etc/systemd/journald.conf` :
    ```ini
    SystemMaxUse=500M
    MaxRetentionSec=7day
    ```
3.  **Surveillance Alerte** : Mise en place d'une alerte Grafana declenchee lorsque l'espace disque depasse 80% d'utilisation.

**Resultat :** L'infrastructure est desormais stable et l'espace disque est maitrise, permettant une execution fiable des operations d'automatisation.