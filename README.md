# Hardened Enterprise Lab (V5)

## Objectif

Concevoir une infrastructure reseau hybride ultra-securisee simulant un environnement d'entreprise, isolant les services critiques du monde exterieur via une DMZ et un VPN WireGuard.

## Architecture

```text
==========================================================================================
                      INFRASTRUCTURE RÉSEAU DURCIE "CORP.LOCAL"
==========================================================================================

      [ INTERNET / WAN ]
              |
      +-------+---------+
      |    FW-GW01      | <--- NAT / Forwarding / Nftables
      +-------+---------+
              |
      +-------+---------------------------+---------------------------+
      |       |                           |                           |
      |   [ ZONE DMZ ]              [ ZONE LAN ]              [ ZONE MGMT ]
      | (172.16.50.0/24)          (192.168.100.0/24)        (192.168.56.0/24)
      |       |                           |                           |
      |       +-- [edge-access01]         +-- [infra01]               +-- [Admin PC]
      |       |   (.10) VPN               |   (.10)                   |
      |       |                           |   (DNS Bind9)             |
      |       +-- [proxy01]               |   (LDAP Slapd)            |
      |           (.20) Nginx             |   (DHCP Kea) <----[Nouveau]|
      |                                   |                           |
      |                                   +-- [mgmt01]                |
      |                                   |   (.50)                   |
      |                                   |   (Ansible)               |
      |                                   |   (CA Root / PKI) <-------[Nouveau]
      |                                   |                           |
      |                                   +-- [secops01]              |
      |                                       (.100) Loki/Grafana     |
      +---------------------------------------------------------------+

------------------------------------------------------------------------------------------
INTEROPÉRABILITÉ DES SERVICES :
------------------------------------------------------------------------------------------
1. PKI  : Mgmt01 génère les certificats -> Diffusés via Ansible vers Proxy/Infra/SecOps.
2. DHCP : Infra01 (Kea) écoute les requêtes DHCP Discover via le relais du Gateway.
3. DNS  : Résolution des noms internes pour tous les serveurs du parc.
------------------------------------------------------------------------------------------

```

## Points Cles de l'Architecture

- **Segmentation Stricte** : Separation physique (VLAN / sous-reseaux) entre la DMZ (services exposes) et le LAN (donnees sensibles).
- **Zero-Trust Entry** : Aucun acces direct a l'administration sans passage par un tunnel VPN WireGuard et une authentification par cle SSH.
- **Filtrage Dynamique** : Utilisation de `nftables` avec une politique par defaut `DROP`.
- **Observabilite** : Centralisation des logs (Loki) et monitoring en temps reel (Grafana).

## Structure du Depot

```text
infra_hardened/
├── docs/                 
├── scripts/                
├── ansible/                
│   ├── hosts.ini
│   └── site.yml
├── diagrams/               
└── README.md
```

## Roles et Flux (Extrait de la Matrice)

| Machine        | Zone   | Role Principal        | Services Cles               |
| :------------- | :----- | :-------------------- | :-------------------------- |
| **fw-gw01**    | Border | Routeur / Pare-feu    | `nftables`, NAT, Forwarding |
| **edge-access01** | DMZ    | Bastion VPN           | `WireGuard`, IP Forwarding  |
| **proxy01**    | DMZ    | Reverse Proxy         | `Nginx`, SSL Termination    |
| **infra01**    | LAN    | Services Coeur        | `Bind9 (DNS)`, `Slapd (LDAP)` |
| **mgmt01**     | LAN    | Administration        | `Ansible`, SSH Master       |
| **secops01**   | LAN    | Surveillance          | `Loki`, `Grafana`, `Promtail` |

Pour plus de details sur les flux autorises et interdits, consulter `docs/architecture.md`.

## Documentation Complementaire

- [Architecture detaillee](docs/architecture.md)
- [Flux reseau](docs/network_flows.md)
- [Gestion de la Configuration avec Ansible](docs/setup_ansible.md)
- [Guide d'installation du pare-feu](docs/setup_gateway.md)
- [Guide d'installation du Bastion VPN](docs/setup_wireguard.md)
- [Guide d'installation du DNS et de l'Annuaire](docs/setup_dns_ldap.md)
- [Mise en Place de l'Observabilite](docs/setup_monitoring.md)
- [Scripts de maintenance](scripts/)

## Licence

Ce projet est un laboratoire personnel destine a la demonstration et a la validation de competences en infrastructure securisee.