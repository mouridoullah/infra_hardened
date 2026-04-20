# Hardened Enterprise Lab (V5)

## Objectif

Concevoir une infrastructure reseau hybride ultra-securisee simulant un environnement d'entreprise, isolant les services critiques du monde exterieur via une DMZ et un VPN WireGuard.

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