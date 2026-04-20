# Matrice des Flux Réseau (Network Flows)

Ce document répertorie les flux autorisés à travers le pare-feu central `fw-gw01`. [cite_start]Tout flux non listé ici est rejeté par la politique `FORWARD: DROP`[cite: 155, 156, 157].

## 1. Flux Entrants (Depuis Internet)
| Source | Destination | Port/Proto | Description |
| :--- | :--- | :--- | :--- |
| Any (Internet) | Edge-Access01 | UDP/51820 | [cite_start]Tunnel VPN WireGuard (DNAT)[cite: 160, 164]. |
| Any (Internet) | Proxy01 | TCP/80, 443 | [cite_start]Accès aux services Web (DNAT)[cite: 160, 165]. |

## 2. Flux Inter-Zones (DMZ vers LAN)
| Source | Destination | Port/Proto | Justification |
| :--- | :--- | :--- | :--- |
| Proxy01 / Edge | Infra01 | UDP-TCP/53 | [cite_start]Résolution DNS interne[cite: 158, 212]. |
| Proxy01 / Edge | Infra01 | TCP/636 | [cite_start]Authentification centralisée (LDAPS)[cite: 158, 212]. |
| DMZ (All) | SecOps01 | TCP/3100 | [cite_start]Envoi des logs via Promtail (Push mode)[cite: 158, 104]. |

## 3. Flux d'Administration (LAN vers LAN/DMZ)
| Source | Destination | Port/Proto | Justification |
| :--- | :--- | :--- | :--- |
| Mgmt01 | All Hosts | TCP/22 | [cite_start]Gestion de configuration via Ansible[cite: 17, 70, 159]. |
| Edge-Access01 | Mgmt01 | TCP/22 | [cite_start]Accès administratif SSH via VPN[cite: 90, 159]. |

## 4. Flux Sortants (Accès Internet)
[cite_start]L'accès vers l'extérieur est restreint et masqué (Masquerade) par l'interface WAN du Gateway[cite: 166].

* [cite_start]**Maintenance temporaire** : L'accès Internet pour les mises à jour (`apt update`) est activé dynamiquement par le playbook Ansible `maintenance.yml` puis refermé immédiatement[cite: 80, 81].

## 5. Schéma logique des flux
`[PC Admin] --(VPN)--> [Edge] --(SSH)--> [Mgmt01] --(Ansible)--> [Toutes les machines]`