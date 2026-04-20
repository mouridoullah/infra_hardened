# Installation et Configuration du Pare-feu (FW-GW01)

Ce document detaille la transformation d'une Ubuntu Server en un routeur filtrant durci, servant de point d'entree principal pour l'infrastructure. La machine est nommee `fw-gw01` et occupe la zone **Border**.

## 1. Configuration Reseau (Interfaces)

Le routeur est equipe de quatre interfaces reseau virtuelles, chacune dediee a un segment distinct afin d'assurer une isolation stricte des domaines de diffusion.

| Interface | Zone        | Adresse IP          | Role                                   |
| :-------- | :---------- | :------------------ | :------------------------------------- |
| `enp0s3`  | WAN         | DHCP (Internet)     | Acces au reseau exterieur (NAT/Bridge) |
| `enp0s8`  | LAN         | 192.168.100.1/24    | Reseau interne securise (Corp)         |
| `enp0s9`  | DMZ         | 172.16.50.1/24      | Zone exposee pour les services publics |
| `enp0s10` | Management  | 192.168.56.10/24    | Reseau d'administration hors bande     |

**Fichier de configuration Netplan** (`/etc/netplan/50-cloud-init.yaml`)

```yaml
network:
  version: 2
  ethernets:
    # --- WAN (Internet via VirtualBox NAT) ---
    enp0s3:
      dhcp4: true
      
    # --- LAN Corporate ---
    enp0s8:
      addresses: [192.168.100.1/24]
      # Pas de gateway ici : cette machine est la passerelle par defaut du LAN
      
    # --- DMZ ---
    enp0s9:
      addresses: [172.16.50.1/24]
      routes:
        # Route statique vers le reseau VPN (WireGuard) situe derriere edge-access01
        - to: 10.8.0.0/24
          via: 172.16.50.10
      
    # --- Management ---
    enp0s10:
      addresses: [192.168.56.10/24]
```

**Application de la configuration**

```bash
sudo netplan apply
```

## 2. Activation du Routage IP (Kernel)

Par defaut, Linux ne transfere pas les paquets entre les interfaces. Le routage doit etre active pour que la machine fonctionne comme une passerelle.

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 3. Installation et Configuration de nftables

`nftables` est le successeur moderne d'`iptables`. Il permet une syntaxe plus concise et une meilleure gestion des performances pour le filtrage de paquets.

**Installation des paquets**

```bash
sudo apt update
sudo apt install nftables -y
sudo systemctl enable nftables
```

**Politique de filtrage adoptee**  
La politique par defaut de la chaine `FORWARD` est fixee a `drop`. Seul le trafic explicitement autorise par une regle peut traverser le routeur. Ceci applique le principe du moindre privilege.

**Fichier `/etc/nftables.conf`**

```nft
#!/usr/sbin/nft -f

table ip filter {
        chain FORWARD {
                type filter hook forward priority filter; policy drop;
                
                # Autoriser le trafic de retour des connexions deja etablies ou associees
                ct state established,related accept
                
                # Acces au serveur DNS/LDAP interne (infra01) depuis le LAN ou la DMZ
                ip daddr 192.168.100.10 tcp dport { 53, 636 } accept
                ip daddr 192.168.100.10 udp dport 53 accept
                
                # Acces au serveur de logs Loki (secops01) depuis les sources autorisees
                ip daddr 192.168.100.100 tcp dport 3100 accept
                
                # Acces SSH vers le serveur d'administration (mgmt01)
                ip saddr 192.168.100.50 tcp dport 22 accept
                ip saddr 172.16.50.10 ip daddr 192.168.100.50 tcp dport 22 accept
                
                # Acces au service WireGuard (edge-access01)
                ip daddr 172.16.50.10 udp dport 51820 accept
                
                # Acces HTTP/HTTPS vers le reverse proxy (proxy01)
                ip daddr 172.16.50.20 tcp dport { 80, 443 } accept
        }

        chain INPUT {
                type filter hook input priority filter; policy accept;
        }

        chain OUTPUT {
                type filter hook output priority filter; policy accept;
        }
}

table ip nat {
        chain PREROUTING {
                type nat hook prerouting priority dstnat; policy accept;
                
                # DNAT : Redirection du flux WireGuard entrant vers le bastion VPN
                iifname "enp0s3" udp dport 51820 dnat to 172.16.50.10
                
                # DNAT : Redirection du trafic Web vers le reverse proxy
                iifname "enp0s3" tcp dport { 80, 443 } dnat to 172.16.50.20
        }

        chain POSTROUTING {
                type nat hook postrouting priority srcnat; policy accept;
                
                # Masquerade : Permet aux machines internes d'acceder a Internet via l'IP publique du routeur
                oifname "enp0s3" masquerade
        }
}
```

**Chargement de la configuration**

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl restart nftables
```

## 4. Validation du Fonctionnement

Apres avoir applique la configuration, les tests suivants permettent de confirmer que le durcissement est actif.

| Test                                                        | Commande (executer depuis la source appropriee)                  | Resultat Attendu |
| :---------------------------------------------------------- | :-------------------------------------------------------------- | :--------------- |
| **1. Ping du LAN vers la DMZ**                              | `ping 172.16.50.10` (depuis une machine du LAN)                  | **Echec** (Timeout) |
| **2. Acces SSH depuis mgmt01 vers edge-access01**           | `ssh user@172.16.50.10` (depuis mgmt01)                         | **Succes** (Connexion etablie) |
| **3. Verification de la politique FORWARD par defaut**      | `sudo nft list chain ip filter FORWARD`                         | `policy drop;` doit apparaitre |
| **4. Verification du NAT (Masquerade)**                     | `curl ifconfig.me` (depuis une machine du LAN)                  | L'IP publique du routeur doit etre affichee |
| **5. Acces HTTP externe vers proxy01**                      | Navigateur externe -> `http://<IP_PUBLIQUE_ROUTEUR>`             | La page par defaut de Nginx s'affiche |

## 5. Justification Architecturale : Placement du VPN

**Question frequente :** *Pourquoi le service WireGuard n'est-il pas directement heberge sur `fw-gw01` ?*

**Reponse :**  
Le routeur frontal (`fw-gw01`) est l'equipement le plus expose de l'infrastructure. Il est directement joignable depuis Internet. Y installer un service supplementaire (WireGuard) augmente sa **surface d'attaque**. Une vulnerabilite ou une erreur de configuration dans WireGuard pourrait compromettre l'integralite du routeur et donc de tout le reseau interne.

En placant WireGuard sur une machine dediee (`edge-access01`) situee dans la **DMZ**, nous appliquons une segmentation de securite. Si `edge-access01` est compromis, l'attaquant n'obtient qu'un acces a la DMZ. Il lui faut encore franchir les regles de filtrage de `fw-gw01` pour atteindre le LAN sensible. Le routeur ne fait que rediriger le flux UDP/51820 vers cette machine dediee, sans executer lui-meme le code de l'application VPN. Cela reduit considerablement le risque.

---
