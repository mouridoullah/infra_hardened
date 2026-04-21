# Matrice des Flux Réseau (Network Flows)

Ce document répertorie les flux autorisés à travers le pare-feu central `fw-gw01`. Tout flux non listé ici est rejeté par la politique `FORWARD: DROP`.

## 1. Flux Entrants (Depuis Internet)
| Source | Destination | Port/Proto | Description |
| :--- | :--- | :--- | :--- |
| Any (Internet) | Edge-Access01 | UDP/51820 | Tunnel VPN WireGuard (DNAT). |
| Any (Internet) | Proxy01 | TCP/80, 443 | Accès aux services Web (DNAT). |

## 2. Flux Inter-Zones (DMZ vers LAN)
| Source | Destination | Port/Proto | Justification |
| :--- | :--- | :--- | :--- |
| Proxy01 / Edge | Infra01 | UDP-TCP/53 | Résolution DNS interne. |
| Proxy01 / Edge | Infra01 | TCP/636 | Authentification centralisée (LDAPS). |
| DMZ (All) | SecOps01 | TCP/3100 | Envoi des logs via Promtail (Push mode). |

## 3. Flux d'Administration (LAN vers LAN/DMZ)
| Source | Destination | Port/Proto | Justification |
| :--- | :--- | :--- | :--- |
| Mgmt01 | All Hosts | TCP/22 | Gestion de configuration via Ansible. |
| Edge-Access01 | Mgmt01 | TCP/22 | Accès administratif SSH via VPN. |

## 4. Flux Sortants (Accès Internet)
L'accès vers l'extérieur est restreint et masqué (Masquerade) par l'interface WAN du Gateway.

* **Maintenance temporaire** : L'accès Internet pour les mises à jour (`apt update`) est activé dynamiquement par le playbook Ansible `maintenance.yml` puis refermé immédiatement.

## 5. Schéma logique des flux
`[PC Admin] --(VPN)--> [Edge] --(SSH)--> [Mgmt01] --(Ansible)--> [Toutes les machines]`
