# Hardened Enterprise Lab

## Objectif

Concevoir une infrastructure reseau hybride ultra-securisee simulant un environnement d'entreprise, isolant les services critiques du monde exterieur via une DMZ et un VPN WireGuard.

## Architecture

```text
==========================================================================================
                      INFRASTRUCTURE RÉSEAU DURCIE "CORP.LOCAL" (V5.1)
==========================================================================================

      [ INTERNET / WAN ]
              |
      +-------+---------+
      |    FW-GW01      | <--- (Relais DHCP / NAT / Nftables) 
      +-------+---------+
              |
      +-------+---------------------------+---------------------------+
      |       |                           |                           |
      |   [ ZONE DMZ ]              [ ZONE LAN ]              [ ZONE MGMT ]
      | (172.16.50.0/24)          (192.168.100.0/24)        (192.168.56.0/24)
      |       |                           |                           |
      |       +-- [edge-access01]         +-- [infra01] (.10)         +-- [Admin PC]
      |       |   (.10) VPN                      |   - DNS (Bind9)            |   (SSH/VPN)
      |       |                                  |   - LDAP (Slapd)           |
      |       +-- [proxy01]                      |   - DHCP (Kea)             |
      |           (.20) Nginx                    |                            |
      |                                          +-- [ca01] (.20)             |
      |                                          |   - PKI (Step-CA)          |
      |                                          |                            |
      |                                          +-- [mgmt01] (.50)           |
      |                                          |   - Ansible Master         |
      |                                          |                            |
      |                                          +-- [secops01] (.100)        |
      |                                              - Loki/Grafana           |
      +-----------------------------------------------------------------------+


==========================================================================================
INTEROPÉRABILITÉ & FLUX RÉSEAU CRITIQUES
==========================================================================================

1. PKI        : CA01 émet les certificats. Ansible (Mgmt01) les diffuse vers le Reverse Proxy (HTTPS) et LDAP (LDAPS).
2. DHCP       : Le Gateway (FW-GW01) sert de "DHCP Relay" pour rediriger les requêtes Discover du LAN vers Infra01 (Kea).
3. DNS        : Résolution Bind9 centralisée sur Infra01 pour tout le parc.
4. VPN ACCESS : [WAN] --(UDP/51820)--> [FW-GW01] --(DNAT)--> [Edge-Access]
5. ADMIN PKI  : [PC Admin] --(WireGuard)--> [Edge] --(SSH)--> [Mgmt01]
6. LOGS PUSH  : [DMZ Hosts] --(TCP/3100)--> [FW-GW01] --(Allow)--> [SecOps01]
7. DNS/LDAP   : [DMZ/LAN] --(UDP/53, TCP/636)--> [FW-GW01] --(Allow)--> [Infra01]
8. MAINTENANCE: [Mgmt01] --(TCP/22)--> [Tous les serveurs] (Ansible Orchestration)
==========================================================================================


```

## Points Cles de l'Architecture

- **Segmentation Stricte** : Separation physique (VLAN / sous-reseaux) entre la DMZ (services exposes) et le LAN (donnees sensibles).
- **Zero-Trust Entry** : Aucun acces direct a l'administration sans passage par un tunnel VPN WireGuard et une authentification par cle SSH.
- **Filtrage Dynamique** : Utilisation de `nftables` avec une politique par defaut `DROP`.
- **Observabilite** : Centralisation des logs (Loki) et monitoring en temps reel (Grafana).

## Structure du Depot

```text
infra_hardened
├── docs             
├── scripts                
├── ansible                
│   ├── hosts.ini
│   └── site.yml
├── diagrams               
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