# Installation et Configuration du Bastion VPN (Edge-Access01)

Ce document decrit la mise en place du service WireGuard sur la machine `edge-access01` (situee dans la zone **DMZ**). Ce bastion constitue l'unique point d'entree chiffre autorisant l'administration a distance de l'ensemble de l'infrastructure.

## 1. Pre-requis et Positionnement Reseau

La machine `edge-access01` possede les caracteristiques suivantes :

- **Adresse IP DMZ** : `172.16.50.10/24`
- **Passerelle par defaut** : `172.16.50.1` (fw-gw01)
- **Role** : Serveur WireGuard et routeur de rebond vers le LAN interne.

### 1.1 Redirection de Port (DNAT) sur le Routeur Frontal

Afin que le flux WireGuard puisse atteindre ce bastion depuis Internet, une regle de redirection de port a ete configuree sur `fw-gw01`.

**Rappel de la regle active (`nftables`)** :
```
iifname "enp0s3" udp dport 51820 dnat to 172.16.50.10
```

Sans cette regle, le paquet UDP entrant serait bloque par la politique `DROP` de la chaine `FORWARD` du pare-feu.

## 2. Installation de WireGuard

WireGuard est selectionne pour sa legerete, ses performances et son integration native au noyau Linux recent.

```bash
sudo apt update
sudo apt install wireguard resolvconf -y
```

Le paquet `resolvconf` facilite la gestion des serveurs DNS associes a l'interface VPN.

## 3. Generation des Cles Cryptographiques

Chaque homologue (serveur et client) necessite une paire de cles publique/privee.

```bash
cd /etc/wireguard
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

**Important** : Les permissions strictes (`umask 077`) garantissent que seule la racine (`root`) peut lire la cle privee.

## 4. Configuration de l'Interface Serveur (`wg0`)

Fichier de configuration : `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <CONTENU_DU_FICHIER_PRIVATEKEY>

# Activation du routage entre l'interface VPN et la DMZ/LAN
PostUp = sysctl -w net.ipv4.ip_forward=1
PostUp = iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o enp0s3 -j MASQUERADE

# Client administrateur (PC Host)
[Peer]
PublicKey = <CLE_PUBLIQUE_DU_POSTE_ADMIN>
AllowedIPs = 10.8.0.2/32
```

**Explication des directives** :
- `Address` : Adresse IP du serveur dans le sous-reseau virtuel du VPN.
- `PostUp` / `PostDown` : Active temporairement le NAT pour que les paquets en provenance du VPN puissent rebondir vers les reseaux internes (LAN et DMZ).
- `AllowedIPs` : Restreint ce client specifique a l'adresse IP `10.8.0.2`.

## 5. Configuration du Client Administrateur (Poste de Travail)

Exemple de configuration pour un client Linux (`/etc/wireguard/wg0.conf`) ou tout autre client compatible WireGuard.

```ini
[Interface]
PrivateKey = <CLE_PRIVEE_DU_POSTE_ADMIN>
Address = 10.8.0.2/24
DNS = 192.168.100.10

[Peer]
PublicKey = <CLE_PUBLIQUE_DU_SERVEUR_EDGE_ACCESS01>
Endpoint = <IP_PUBLIQUE_DU_ROUTEUR_FW_GW01>:51820
AllowedIPs = 10.8.0.0/24, 192.168.100.0/24, 172.16.50.0/24
PersistentKeepalive = 25
```

**Remarques importantes pour le client** :
- **AllowedIPs** : Cette directive definit les routes qui seront poussees dans la table de routage du client. L'administrateur peut ainsi acceder directement aux IP de la DMZ et du LAN.
- **Endpoint** : Pointe vers l'adresse publique du routeur frontal (`fw-gw01`), qui redirige ensuite le flux vers la DMZ.
- **PersistentKeepalive** : Recommande si le client est situe derriere un NAT ou un pare-feu avec des delais d'expiration de session courts.

## 6. Activation et Demarrage du Service

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Verification de l'etat de l'interface :

```bash
sudo wg show
```

## 7. Validation et Tests de Connectivite

Une fois le tunnel etabli depuis le client, les tests suivants confirment le bon fonctionnement :

| Test                                         | Commande (executee depuis le client VPN)         | Resultat Attendu          |
| :------------------------------------------- | :----------------------------------------------- | :------------------------ |
| **1. Ping de l'interface VPN du serveur**    | `ping 10.8.0.1`                                  | Reponse                   |
| **2. Acces SSH au serveur d'administration** | `ssh user@192.168.100.50`                        | Connexion etablie         |
| **3. Verification de la table de routage**   | `ip route show \| grep 192.168`                   | `192.168.100.0/24 dev wg0` |

## 8. Propriete de Furtivite de WireGuard (Silence Cryptographique)

**Question d'audit frequente** : *Si un attaquant scanne le port UDP 51820 avec Nmap, ne verra-t-il pas que le port est ouvert et qu'un VPN WireGuard est actif ?*

**Reponse** : Non, un scan reseau standard ne revelera pas que le port `51820` est ouvert pour WireGuard.

Contrairement a OpenVPN ou IPsec qui repondent aux demandes de connexion initiales (par exemple par un `SYN-ACK` pour TCP ou une reponse IKE pour IPsec), WireGuard applique le principe du **Silence Cryptographique**.

- Si un paquet UDP arrive sur le port `51820` sans etre chiffre avec la cle publique correcte du serveur, WireGuard **ignore silencieusement le paquet**.
- Le systeme d'exploitation ne renvoie aucun message `ICMP Port Unreachable`.
- Par consequent, un scanner comme `nmap` conclura que le port est **`filtered`** (ou silencieux), exactement comme si le port etait ferme et bloque par un pare-feu.

Cette propriete reduit considerablement la surface d'attaque et rend la presence du VPN quasi invisible aux scans automatiques.