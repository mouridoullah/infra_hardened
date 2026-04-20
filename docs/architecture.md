# Architecture détaillée de l'infrastructure Corp.Local

## 1. Vision Globale
L'infrastructure est conçue sur un modèle de **Défense en Profondeur**. [cite_start]Elle simule un environnement d'entreprise où les services sont segmentés en zones de confiance distinctes, régies par une passerelle centrale (FW-GW01)[cite: 1, 2, 11].

## 2. Segmentation des Zones
[cite_start]L'architecture se découpe en quatre segments réseau isolés[cite: 139, 140, 143]:

| Zone | Sous-réseau | Description |
| :--- | :--- | :--- |
| **Border (WAN)** | DHCP | Interface publique reliée à l'Internet. [cite_start]Seul le flux VPN WireGuard est autorisé en entrée via DNAT[cite: 144, 145, 164]. |
| **DMZ** | 172.16.50.0/24 | [cite_start]Zone démilitarisée hébergeant les services exposés (Proxy, Bastion VPN)[cite: 147, 148]. |
| **LAN Corporate** | 192.168.100.0/24 | [cite_start]Zone interne hautement sécurisée hébergeant les données et services critiques (LDAP, DNS, Monitoring)[cite: 146]. |
| **Management** | 192.168.56.0/24 | [cite_start]Réseau d'administration hors-bande (Out-of-band) pour la maintenance physique des nœuds[cite: 148, 149]. |

## 3. Rôles des Composants
* **FW-GW01 (Gateway)** : Routeur filtrant sous `nftables`. [cite_start]Il assure le routage inter-zones, le NAT et la sécurité périmétrique[cite: 11, 137, 153].
* **Edge-Access01 (Bastion)** : Terminaison du tunnel VPN WireGuard. [cite_start]Il isole l'accès distant du cœur du réseau[cite: 12, 26, 184].
* [cite_start]**Infra01 (Services Coeur)** : Serveur centralisant l'annuaire OpenLDAP et le DNS Bind9[cite: 15, 16, 189].
* **Mgmt01 (Administration)** : Nœud de contrôle Ansible. [cite_start]C'est l'unique point d'où partent les configurations vers le reste du parc[cite: 17, 60].
* [cite_start]**SecOps01 (Observabilité)** : Centralisation des logs avec Loki et dashboarding avec Grafana[cite: 18, 19, 98].

## 4. Concepts de Sécurité Appliqués
* **Moindre Privilège** : La politique par défaut du pare-feu est à `DROP`. [cite_start]Chaque flux doit être explicitement justifié[cite: 4, 155, 156].
* [cite_start]**Silence Cryptographique** : Utilisation de WireGuard pour rendre le port VPN invisible aux scans non autorisés[cite: 54, 55, 59].
* [cite_start]**Isolation de l'Administration** : Aucune machine de la DMZ ne peut initier de connexion vers le LAN, sauf pour la collecte de logs[cite: 89, 91, 107, 108].