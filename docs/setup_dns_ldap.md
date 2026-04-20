# Installation des Services Coeur : DNS et Annuaire (Infra01)

Ce document detaille la configuration des services critiques de l'infrastructure interne sur la machine `infra01` (192.168.100.10). Ce serveur heberge le service de resolution de noms **Bind9** et l'annuaire centralise **OpenLDAP**, piliers de la zone **LAN**.

## 1. Configuration du Reseau Local

La machine `infra01` est situee dans le LAN avec l'adresse IP fixe `192.168.100.10/24`. Sa passerelle par defaut est `fw-gw01` (`192.168.100.1`).

## 2. Service DNS : Bind9

Le service DNS permet aux differentes machines de l'infrastructure de communiquer par nom d'hote (`mgmt01.corp.local`) plutot que par adresse IP.

### 2.1 Installation

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

### 2.2 Configuration de la Zone Directe

Le fichier de zone `/etc/bind/db.corp.local` definit les enregistrements pour le domaine `corp.local`.

```dns
$TTL    604800
@       IN      SOA     infra01.corp.local. admin.corp.local. (
                              3         ; Serial (incremente a chaque modification)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Serveur de noms faisant autorite pour la zone
@           IN      NS      infra01.corp.local.

; Enregistrements A (Adresses IPv4)
infra01     IN      A       192.168.100.10
fw-gw01     IN      A       192.168.100.1
proxy01     IN      A       172.16.50.20
secops01    IN      A       192.168.100.100
edge-access IN      A       172.16.50.10
mgmt01      IN      A       192.168.100.50
ca01        IN      A       192.168.100.20

; Le domaine principal pointe vers le reverse proxy (zone DMZ)
@           IN      A       172.16.50.20
```

### 2.3 Hardening DNS : Restriction des Requetes Recursives

Pour eviter que le serveur DNS ne soit utilise dans des attaques par amplification, les requetes recursives ne sont autorisees que pour les reseaux de confiance (LAN et DMZ). Cela est configure dans `/etc/bind/named.conf.options` :

```
acl "trusted" {
    192.168.100.0/24;   # LAN
    172.16.50.0/24;     # DMZ
    127.0.0.1;
};

options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { trusted; };
    listen-on { 192.168.100.10; };
    allow-transfer { none; };
    forwarders {
        1.1.1.1;
        8.8.8.8;
    };
};
```

**Redemarrage du service :**

```bash
sudo systemctl restart bind9
```

## 3. Service Annuaire : OpenLDAP

OpenLDAP centralise les identites des utilisateurs et des groupes, permettant une authentification unifiee sur l'ensemble de l'infrastructure (SSH, services Web).

### 3.1 Installation

```bash
sudo apt install slapd ldap-utils -y
```

Lors de l'installation, le mot de passe administrateur LDAP est defini. La base de l'annuaire est `dc=corp,dc=local`.

### 3.2 Structure de l'Annuaire (DIT)

L'annuaire est structure comme suit :

```
dc=corp,dc=local
├── ou=users
│   ├── uid=admin1
│   └── uid=operator1
└── ou=groups
    ├── cn=admins
    └── cn=sudoers
```

Les utilisateurs membres du groupe `sudoers` disposent des privileges `sudo` sur les serveurs configures pour utiliser l'authentification LDAP.

### 3.3 Securisation avec LDAPS (Port 636)

Le trafic LDAP contenant les mots de passe ne doit jamais circuler en clair. OpenLDAP est configure pour accepter les connexions chiffrees via **LDAPS** (LDAP over TLS).

- Un certificat serveur est genere par l'autorite de certification interne **CA01** (`ca01.corp.local`).
- Les certificats sont places dans `/etc/ldap/sasl2/`.
- Le demon `slapd` est re-demarre pour ecouter sur le port `636`.

```bash
sudo systemctl enable slapd
sudo systemctl restart slapd
```

**Verification de l'ecoute sur le port securise :**

```bash
sudo ss -tulpn | grep 636
```

## 4. Validation des Flux Reseau : Communication DMZ vers LAN

**Question de validation architecturale** : *Comment le serveur `proxy01` (situe en DMZ) peut-il interroger le DNS `infra01` (situe en LAN) sans que toute la DMZ ait acces au LAN ?*

**Reponse** : Le pare-feu `fw-gw01` applique une politique de filtrage stricte basee sur les adresses IP et les ports de destination.

Dans la configuration `nftables` de `fw-gw01`, les regles suivantes ont ete definies :

```nft
# Autoriser les requetes DNS vers le serveur Infra01
ip daddr 192.168.100.10 udp dport 53 accept
ip daddr 192.168.100.10 tcp dport 53 accept

# Autoriser egalement LDAPS (port 636) si necessaire
ip daddr 192.168.100.10 tcp dport 636 accept
```

Ces regles autorisent **explicitement** le trafic entrant vers `infra01` sur les ports DNS (53) et LDAPS (636). Comme la politique par defaut de la chaine `FORWARD` est `drop`, tout autre trafic initie depuis la DMZ vers le LAN (par exemple une tentative SSH vers `mgmt01`) est bloque.

Ainsi, `proxy01` peut resoudre `infra01.corp.local` (via UDP/53) mais ne peut pas initier de connexion non autorisee vers d'autres ressources du LAN. Cela repond au principe du **moindre privilege** au niveau du routage.
