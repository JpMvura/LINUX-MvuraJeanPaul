# B1 Linux - TP2

## I. Exploration locale en solo

### 1. Affichage d'informations sur la pile TCP/IP locale

### a) Infos des cartes réseau (WiFi + Ethernet) 
`ip addr show`

**![alt text](image.png)**

Résultats (extraits) :

    Interface WiFi

        Nom : wlp1s0

        MAC : 18:56:80:18:f3:26

        IP : 192.168.1.175/24

    Interface Ethernet

        Nom : enp0s31f6

        MAC : e4:e7:49:1d:45:73

        IP : aucune (interface DOWN)


### b)Adresse de réseau + broadcast

`ipcalc 192.168.1.175/24`

![alt text](image-1.png)

Résultats :

    WiFi (wlp1s0)

        Adresse de réseau : 192.168.1.0/24

        Adresse de broadcast : 192.168.1.255

    Ethernet (enp0s31f6)

        Non applicable car pas d’IP configurée (interface DOWN)


### c)Afficher la gateway (passerelle)

`ip route show default`

![alt text](image-2.png)

Gateway WiFi : 192.168.1.1 (default via 192.168.1.1 dev wlp1s0) [web:7]

### En graphique (GUI)

![alt text](image-3.png)

nfos relevées (interface WiFi) :

    IP : 192.168.1.175

    MAC : 18:56:80:18:f3:26

    Gateway : 192.168.1.1

# Question : À quoi sert la gateway dans le réseau d'Ingésup ?

La gateway (passerelle par défaut) est le routeur du réseau local (ex: 192.168.1.1) vers lequel le PC envoie tout trafic destiné à une adresse **hors** de son sous-réseau (par exemple, Internet).  
Elle sert donc de “sortie” du réseau local Ingésup vers d’autres réseaux et permet l’accès à Internet (souvent via du NAT).


## 2. Modifications des informations

### A. Modification d'adresse IP - Pt. 1

**Calcul première/dernière IP disponible :**
- Réseau : `192.168.1.0/24` 
- **Première IP disponible : `192.168.1.1`** (adresse réseau .0 réservée)
- **Dernière IP disponible : `192.168.1.254`** (broadcast .255 réservé)
- IPs utilisables : **254 adresses** (1 à 254)

**Changement via interface graphique Ubuntu :**

![alt text](image-4.png)

**Vérification CLI après changement :**

![alt text](image-5.png)

### B. nmap - Scanner réseau pour IP libre

**Scan réseau WiFi (Ping Scan) :**

![alt text](image-7.png)

Résultats du scan :

    Hôtes actifs détectés : [liste des IPs trouvées, ex: 192.168.1.1, .175, etc.]

    IP libre trouvée : 192.168.1.XXX (absent du scan, donc disponible)

Explication nmap :

    -sn = Ping Scan (détecte hôtes vivants sans port scan)

    192.168.1.0/24 = cible tout le réseau (256 adresses)

    Permet de voir quelles IPs sont occupées pour choisir une libre

### C .Modification d'adresse IP - Pt. 2

Changement vers IP libre + gateway incorrecte :

![alt text](image-8.png)

Test connectivité Internet :

![alt text](image-9.png)

Explication du problème :

    IP libre OK → communication locale possible

    Gateway fausse → impossible d'atteindre Internet (pas de route vers 8.8.8.8)

    Même principe que DHCP automatique qui configure tout correctement

Retour configuration DHCP automatique :

![alt text](image-10.png)



## **III. Manipulations d'autres outils/protocoles côté client**

### 1. DHCP

#### a) Afficher l’adresse IP du serveur DHCP du réseau WiFi

**Commande bash :**
nmcli con show "SFR_D89F" | grep -i


Explication :

    dhcp_server_identifier = 192.168.1.1 → c’est l’adresse IP du serveur DHCP de mon réseau WiFi.

    ip_address = 192.168.1.175 → IP attribuée à ma carte WiFi.

    routers = 192.168.1.1 → gateway reçue via DHCP.

    subnet_mask = 255.255.255.0 et domain_name_servers = 192.168.1.1 → masque et DNS fournis automatiquement. [web:8][conversation_history:2]

#### b) Trouver la date d’expiration du bail DHCP (DHCP lease)

Commande bash :
nmcli con show "SFR_D89F" | grep -i 


Explication :

    dhcp_lease_time = 86400 → le bail dure 86400 secondes, soit 24 heures.

    expiry = 1769538262 → timestamp UNIX indiquant la date/heure exacte d’expiration du bail.

    Le serveur DHCP loue donc l’adresse 192.168.1.175 pour 24h, puis le client devra la renouveler (ou en demander une nouvelle).

Commande complémentaire (journal NetworkManager) :
journalctl -u NetworkManager | grep -i "dhcp4 (wlp1s0)" | tail -5

[Screenshot DHCP-3 : journalctl_dhcp_new_lease.png]

Explication :

    On voit des lignes du type :
    state changed new lease, address=192.168.1.175 → NetworkManager confirme qu’un nouveau bail DHCP a été obtenu pour cette IP.

c) Fonctionnement de DHCP (grandes lignes)

Résumé (pas de commande, partie théorique) :

DHCP (Dynamic Host Configuration Protocol) est un protocole qui permet à un client d’obtenir automatiquement ses paramètres réseau (adresse IP, masque, gateway, DNS) auprès d’un serveur, sans configuration manuelle. [web:8]
Le dialogue classique se fait en quatre grandes étapes : Discover → Offer → Request → Ack (DORA), ce qui aboutit à l’attribution d’un bail (lease) limité dans le temps que le client renouvellera avant expiration. [web:8][web:54]
d) Demander une nouvelle adresse IP (en ligne de commande)

Commande bash (renew via NetworkManager) :
nmcli con down "SFR_D89F"
sleep 3
nmcli con up "SFR_D89F"
ip addr show wlp1s0

[Screenshot DHCP-4 : nmcli_renew_and_ip_addr.png]

Explication :

    nmcli con down "SFR_D89F" coupe la connexion WiFi.

    nmcli con up "SFR_D89F" relance la connexion et déclenche une nouvelle négociation DHCP avec le serveur. [web:63]

    ip addr show wlp1s0 permet de vérifier l’adresse IP obtenue après renouvellement (dans mon cas, l’IP reste 192.168.1.175/24, mais elle aurait pu changer selon la configuration du serveur DHCP). [conversation_history:2]


## 2. DNS
### a) Trouver l'adresse IP du serveur DNS utilisé

But : trouver quels serveurs DNS votre machine utilise (souvent reçu via DHCP). 

Commande (NetworkManager) :
nmcli dev show wlp1s0 | grep -i "IP4.DNS"

[Screenshot III-DNS-1 : nmcli_dns.png]

### b) Faire des lookups (dig)

Installer l’outil:
sudo apt update
sudo apt install dnsutils -y

[Screenshot III-DNS-2 : install_dnsutils.png]

Lookup (nom → IP) :
dig google.com +short
dig ynov.com +short

[Screenshot III-DNS-3 : dig_forward.png]

Interprétation (2–3 lignes) :

    dig google.com retourne une ou plusieurs IP : ce sont les adresses des serveurs/points d’entrée associés au nom de domaine.

    Plusieurs IP possibles = répartition de charge / infrastructure distribuée.

Reverse lookup (IP → nom, si un PTR existe) :
dig -x 78.78.21.21 +short
dig -x 92.16.54.88 +short

[Screenshot III-DNS-4 : dig_reverse.png]




