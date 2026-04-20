# Hardened Enterprise Lab (V5)

## Objectif

Concevoir une infrastructure reseau hybride ultra-securisee simulant un environnement d'entreprise, isolant les services critiques du monde exterieur via une DMZ et un VPN WireGuard.

## Architecture

```text
==========================================================================================
                      INFRASTRUCTURE RÉSEAU DURCIE "CORP.LOCAL" (V5)
==========================================================================================

      [ INTERNET / WAN ]
              |
              | (Port UDP/51820 - VPN WireGuard) [cite: 30, 164]
              | (Ports TCP/80, 443 - HTTP/S) [cite: 165]
              v
      +-----------------+
      |    FW-GW01      | <--- (Rôle: Routeur/Pare-feu Nftables) [cite: 11, 138]
      |  (Le Douanier)  |      (Policy: FORWARD DROP) [cite: 4, 155, 157]
      +-------+---------+
              |
      +-------+---------------------------+---------------------------+
      |       |                           |                           |
      |   [ ZONE DMZ ]              [ ZONE LAN ]              [ ZONE MGMT ]
      | (172.16.50.0/24)          (192.168.100.0/24)        (192.168.56.0/24) [cite: 149]
      |       |                           |                           |
      |       +-- [edge-access01]         +-- [infra01]               +-- [Host PC]
      |       |   (.10) [cite: 147, 28]         |   (.10) [cite: 146, 191]        |   (Admin)
      |       |   (WireGuard) [cite: 12, 27]    |   (DNS/LDAP) [cite: 15, 189]    |
      |       |                           |                           |
      |       +-- [proxy01]               +-- [mgmt01]                |
      |           (.20) [cite: 147, 198]        |   (.50) [cite: 146, 191]        |
      |           (Nginx) [cite: 14, 160]       |   (Ansible) [cite: 17, 60]      |
      |                                   |                           |
      |                                   +-- [secops01]              |
      |                                   |   (.100) [cite: 146, 101]     |
      |                                   |   (Loki/Grafana) [cite: 18, 100]  |
      |                                   |                           |
      |                                   +-- [Windows-VM] <--- (Optionnelle)
      |                                       (.11) (Audit/Workstation)
      |                                                               |
      +---------------------------------------------------------------+

------------------------------------------------------------------------------------------
FLUX RÉSEAU CRITIQUES (MATRICE DE SÉCURITÉ)
------------------------------------------------------------------------------------------

1.  VPN ACCESS : [WAN] --(UDP/51820)--> [FW-GW01] --(DNAT)--> [Edge-Access] [cite: 30, 164]
2.  ADMIN PKI  : [PC Admin] --(WireGuard)--> [Edge] --(SSH)--> [Mgmt01] [cite: 90, 92]
3.  LOGS PUSH  : [DMZ Hosts] --(TCP/3100)--> [FW-GW01] --(Allow)--> [SecOps01] [cite: 104, 158]
4.  DNS/LDAP   : [DMZ/LAN] --(UDP/53, TCP/636)--> [FW-GW01] --(Allow)--> [Infra01] [cite: 157, 158]
5.  MAINTENANCE: [Mgmt01] --(TCP/22)--> [Tous les serveurs] (Ansible Orchestration) [cite: 65, 80]
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