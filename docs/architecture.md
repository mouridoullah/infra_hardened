# Architecture détaillée de l'infrastructure Corp.Local

## 1. Vision Globale
L'infrastructure est conçue sur un modèle de **Défense en Profondeur**. [cite_start]Elle simule un environnement d'entreprise où les services sont segmentés en zones de confiance distinctes, régies par une passerelle centrale (FW-GW01).

## 2. Segmentation des Zones
L'architecture se découpe en quatre segments réseau isolés:

| Zone | Sous-réseau | Description |
| :--- | :--- | :--- |
| **Border (WAN)** | DHCP | Interface publique reliée à l'Internet. Seul le flux VPN WireGuard est autorisé en entrée via DNAT. |
| **DMZ** | 172.16.50.0/24 | Zone démilitarisée hébergeant les services exposés (Proxy, Bastion VPN). |
| **LAN Corporate** | 192.168.100.0/24 | Zone interne hautement sécurisée hébergeant les données et services critiques (LDAP, DNS, Monitoring). |
| **Management** | 192.168.56.0/24 | Réseau d'administration hors-bande (Out-of-band) pour la maintenance physique des nœuds. |

## 3. Rôles des Composants
* **FW-GW01 (Gateway)** : Routeur filtrant sous `nftables`. Il assure le routage inter-zones, le NAT et la sécurité périmétrique.
* **Edge-Access01 (Bastion)** : Terminaison du tunnel VPN WireGuard. Il isole l'accès distant du cœur du réseau.
* **Infra01 (Services Coeur)** : Serveur centralisant l'annuaire OpenLDAP et le DNS Bind9.
* **Mgmt01 (Administration)** : Nœud de contrôle Ansible. C'est l'unique point d'où partent les configurations vers le reste du parc.
* **SecOps01 (Observabilité)** : Centralisation des logs avec Loki et dashboarding avec Grafana.

## 4. Concepts de Sécurité Appliqués
* **Moindre Privilège** : La politique par défaut du pare-feu est à `DROP`. Chaque flux doit être explicitement justifié.
* **Silence Cryptographique** : Utilisation de WireGuard pour rendre le port VPN invisible aux scans non autorisés.
* **Isolation de l'Administration** : Aucune machine de la DMZ ne peut initier de connexion vers le LAN, sauf pour la collecte de logs.
