# Automatisation et Gestion de Configuration avec Ansible (Mgmt01)

Ce document presente l'architecture d'automatisation centralisee sur la machine **mgmt01** (192.168.100.50). Ansible est utilise pour garantir la coherence, la reproductibilite et la conformite securitaire de l'ensemble du parc informatique.

## 1. Positionnement et Philosophie

Ansible est un outil **Agentless** : il ne necessite aucun demon ou logiciel specifique sur les machines administrees. L'unique prerequis est un serveur SSH fonctionnel et un interpreteur Python. Cette approche reduit considerablement la surface d'attaque des serveurs de production.

- **Noeud de controle** : `mgmt01.corp.local` (192.168.100.50)
- **Cibles** : Toutes les machines des zones LAN, DMZ et Border.
- **Methode d'authentification** : Cle SSH uniquement (mot de passe desactive).

## 2. Structure du Projet Ansible

Le repertoire `~/ansible_corp/` sur `mgmt01` est organise comme suit :

```text
ansible_corp/
├── ansible.cfg                 # Configuration Ansible (inventaire, verification des cles)
├── hosts.ini                   # Inventaire segmenté par rôles
├── harden_ssh.yml              # Durcissement SSH (PasswordAuthentication no)
├── ldap.yml                    # Déploiement silencieux d'OpenLDAP
├── monitoring.yml              # Installation de Prometheus/Grafana
├── pki.yml                     # Installation de Step-CA
├── sssd.yml                    # Intégration LDAP pour les clients
├── trust_ca.yml                # Distribution du certificat racine
├── logging.yml                 # Déploiement de Promtail sur tout le parc
├── maintenance.yml             # Gestion dynamique des flux nftables (Ouverture/Fermeture)
└── site.yml                    # Playbook principal (agrège les étapes de construction)
```

## 3. Inventaire Segmenté (`hosts.ini`)

L'inventaire utilise des groupes fonctionnels pour limiter l'impact des operations et cibler precisement les composants.

```ini
[webservers]
172.16.50.20

[infrastructure]
192.168.100.10

[monitoring]
192.168.100.100

[gateway]
fw-gw01 ansible_host=172.16.50.1

[pki]
192.168.100.20

[management]
192.168.100.50

[all:vars]
ansible_user=bdvm
ansible_become=yes
ansible_become_method=sudo
```

## 4. Securisation des Acces SSH

Le playbook `harden_ssh.yml` est applique a l'ensemble des machines (`hosts: all`) pour imposer l'authentification par cle et supprimer le risque d'attaque par force brute sur les mots de passe.

**Extrait de `harden_ssh.yml`** :
```yaml
- name: Désactiver l'authentification par mot de passe
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    backup: yes
```

La cle publique SSH du compte `bdvm` de `mgmt01` a ete prealablement copiee sur chaque cible via `ssh-copy-id`.

## 5. Playbooks d'Infrastructure Essentiels

### 5.1 Déploiement de l'Annuaire LDAP (`ldap.yml`)

Automatise l'installation silencieuse de `slapd` en evitant les invites interactives grace au module `debconf`.

### 5.2 Déploiement de la PKI Interne (`pki.yml`)

Telecharge et installe les paquets `step-cli` et `step-ca` pour etablir une autorite de certification privee.

### 5.3 Intégration des Clients LDAP (`sssd.yml`)

Configure SSSD sur les machines clientes pour permettre l'authentification centralisee. Le fichier de configuration est deploye avec des droits stricts `0600` (obligatoire pour SSSD).

### 5.4 Distribution du Certificat Racine (`trust_ca.yml`)

Inscrit le certificat de l'autorite interne dans le magasin de confiance de chaque machine du parc, garantissant la validite des certificats TLS emis en interne (LDAPS, HTTPS interne).

### 5.5 Supervision Centralisee (`logging.yml` et `monitoring.yml`)

- `monitoring.yml` installe Prometheus, Node Exporter et Grafana sur `secops01`.
- `logging.yml` deploye l'agent `promtail` sur toutes les machines et les configure pour pousser leurs logs vers Loki.

## 6. Gestion Dynamique du Pare-feu : Le Playbook de Maintenance

Le playbook `maintenance.yml` constitue une piece maitresse de la securite operationnelle. Il permet d'ouvrir ou de fermer temporairement des flux sur le routeur `fw-gw01` via `nftables`, uniquement pendant la duree d'une intervention.

**Principe de fonctionnement** :
- Le playbook utilise le module `ansible.builtin.script` ou `command` pour ajouter/supprimer des regles `nftables` de maniere atomique.
- A la fin de l'operation de maintenance, le playbook assure la reapplication de la politique de filtrage nominale (retour a l'etat `DROP` par defaut).

## 7. Executer le Projet (How to Run)

Pour reconstruire l'integralite de l'infrastructure a partir d'un jeu de serveurs vierges, l'administrateur execute simplement :

```bash
ansible-playbook site.yml
```

Ce playbook principal orchestre l'ensemble des etapes :
1. Mise a jour des paquets (`update.yml`).
2. Configuration DNS cliente (`set_dns_client.yml`).
3. Déploiement de l'annuaire et de la PKI.
4. Distribution du certificat racine.
5. Déploiement des agents de supervision.
6. Durcissement SSH et application des politiques de securite.

Cette commande unique garantit une infrastructure **industrialisee, reproductible et documentee**.

## 8. Protection du Bastion d'Administration (Mgmt01)

**Question d'audit** : *Mgmt01 centralise tout le pouvoir d'administration. N'est-il pas devenu le point de defaillance unique (SPOF) et la cible prioritaire ?*

**Reponse** : La conception de l'infrastructure integre deux mesures critiques pour proteger `mgmt01` contre les attaques provenant du reste du reseau, notamment de la DMZ.

1.  **Isolation par Filtrage Reseau (nftables)** : La configuration du pare-feu `fw-gw01` n'autorise **strictement aucune connexion entrante** vers `mgmt01` depuis la DMZ ou Internet. La seule exception est la regle suivante, limitee au bastion VPN :
    ```nft
    ip saddr 172.16.50.10 ip daddr 192.168.100.50 tcp dport 22 accept
    ```
    Cette regle signifie que seul `edge-access01` (le serveur WireGuard) peut initier une connexion SSH vers `mgmt01`. Aucune autre machine de la DMZ, et encore moins l'exterieur, ne peut joindre le port 22 de `mgmt01`.

2.  **Authentification Multi-Facteurs Implicite (Tunnel VPN + Cle SSH)** : L'acces administrateur est structure en deux couches successives :
    - **Couche 1** : Le tunnel WireGuard. L'utilisateur doit posseder une cle privee autorisee pour simplement "voir" le reseau interne. Sans cela, `mgmt01` est invisible.
    - **Couche 2** : L'authentification SSH par cle publique. Meme si un attaquant parvenait a percer le tunnel VPN (ce qui necessiterait une compromission cryptographique majeure), il lui faudrait encore la cle privee SSH specifique du compte `bdvm` pour executer des commandes.

Ces deux mesures combinees respectent le principe de **Defense en Profondeur**. `mgmt01` n'est pas simplement protege par un mot de passe ; il est isole derriere une passerelle VPN elle-meme situee dans une zone demilitarisee et filtree.
