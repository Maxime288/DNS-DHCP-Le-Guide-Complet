# 🌐 DNS & DHCP — Le Guide Complet

> **Comprendre les fondations invisibles du réseau** — De la résolution DNS au VLAN, tout ce que vous devez savoir, enfin bien expliqué.

---

## 📚 Table des matières

1. [Résolution DNS](#-résolution-dns)
2. [Recursive Resolver](#-recursive-resolver)
3. [Cache Poisoning](#-cache-poisoning)
4. [DNS Spoofing](#-dns-spoofing)
5. [DHCP](#-dhcp)
6. [Adressage IP](#-adressage-ip)
7. [NAT](#-nat-network-address-translation)
8. [VLAN](#-vlan-virtual-local-area-network)

---

## 🔍 Résolution DNS

Le **DNS (Domain Name System)** est l'annuaire d'Internet. Il traduit un nom de domaine lisible par un humain en adresse IP compréhensible par une machine.

### Comment ça marche ?

```
Vous tapez : www.example.com
             ↓
Le navigateur interroge un résolveur DNS
             ↓
Le résolveur remonte la hiérarchie DNS
             ↓
Réponse : 93.184.216.34
```

### La hiérarchie DNS

```
                        . (Root)
                       / \
                    .com  .fr  .org  ...
                    /
               example.com
                    |
              www.example.com
```

| Niveau | Rôle | Exemple |
|--------|------|---------|
| **Root Servers** | Sommet de la hiérarchie (13 clusters mondiaux) | `a.root-servers.net` |
| **TLD Servers** | Gèrent les extensions (`.com`, `.fr`...) | `a.gtld-servers.net` |
| **Authoritative NS** | Répondent pour un domaine spécifique | `ns1.example.com` |

### Les types d'enregistrements DNS

| Type | Utilité | Exemple |
|------|---------|---------|
| `A` | IPv4 | `example.com → 93.184.216.34` |
| `AAAA` | IPv6 | `example.com → 2606:2800::1` |
| `CNAME` | Alias vers un autre nom | `www → example.com` |
| `MX` | Serveur mail | `mail.example.com` |
| `TXT` | Texte libre (SPF, DKIM...) | `"v=spf1 include:..."` |
| `NS` | Serveurs de noms | `ns1.example.com` |
| `PTR` | Résolution inverse (IP → nom) | `34.216.184.93.in-addr.arpa` |

---

## 🔄 Recursive Resolver

Le **résolveur récursif** est l'intermédiaire entre votre machine et la hiérarchie DNS. Il fait le travail à votre place.

### Étapes détaillées d'une résolution

```
Client                Resolver            Root NS         TLD NS         Auth NS
  |                      |                   |               |               |
  |-- "www.example.com?" →|                   |               |               |
  |                      |-- "Qui gère .com?" →|               |               |
  |                      |←-- "Parle à TLD NS" --|               |               |
  |                      |                       |               |               |
  |                      |-- "Qui gère example.com?" ----------→|               |
  |                      |←-- "Parle à Auth NS" ----------------|               |
  |                      |                                                       |
  |                      |-- "Quelle IP pour www.example.com?" ---------------→|
  |                      |←-- "93.184.216.34, TTL: 3600s" --------------------|
  |                      |                   |               |               |
  |←-- 93.184.216.34 ----|
```

### Cache du résolveur

Après une première résolution, la réponse est **mise en cache** selon le TTL (Time To Live) :

```bash
# Vérifier le TTL d'un enregistrement
dig www.example.com | grep -E "ANSWER|[0-9]+\s+IN"

# Exemple de réponse :
;; ANSWER SECTION:
www.example.com.   3600   IN   A   93.184.216.34
#                  ^^^^
#                  TTL en secondes (1 heure)
```

> **⚠️ Implication sécurité** : Un TTL élevé = propagation lente des changements. Un TTL bas = plus de requêtes, moins de cache exploitable.

---

## ☠️ Cache Poisoning

Le **Cache Poisoning** (ou empoisonnement du cache DNS) consiste à injecter de fausses réponses DNS dans le cache d'un résolveur, pour rediriger les utilisateurs vers des serveurs malveillants.

### Principe de l'attaque

```
Attaquant                   Résolveur victime           Serveur légitime
    |                              |                           |
    |    Le résolveur interroge le vrai serveur                |
    |                              |--- "example.com?" -----→|
    |                              |                           |
    |  L'attaquant flood avec de fausses réponses              |
    |-- "93.X.X.X (FAUX !) ID: 4242" --→|                   |
    |-- "93.X.X.X (FAUX !) ID: 4243" --→|                   |
    |-- "93.X.X.X (FAUX !) ID: 4244" --→|                   |
    |                              |←--- Vraie réponse (trop tard ?)
    |                              |
    |          Si l'attaquant gagne la "course" :              |
    |          Le cache empoisonné redirige TOUS               |
    |          les utilisateurs vers le faux serveur           |
```

### La faille de Kaminsky (2008)

Dan Kaminsky a découvert qu'un attaquant pouvait démultiplier ses chances en demandant des sous-domaines aléatoires :

```
attaque.example.com ?  → force le résolveur à re-interroger
attaque2.example.com ? → nouvelle tentative d'empoisonnement
attaque3.example.com ? → encore...
```

### Contre-mesures

| Protection | Mécanisme |
|-----------|-----------|
| **DNSSEC** | Signatures cryptographiques des réponses DNS |
| **0x20 encoding** | Randomisation de la casse (`ExAmPlE.CoM`) pour vérification |
| **Port source aléatoire** | Augmente l'entropie nécessaire à l'attaquant |
| **Query ID aléatoire** | 16 bits d'entropie (insuffisant seul) |

```bash
# Vérifier si DNSSEC est actif sur un domaine
dig +dnssec example.com
# Chercher les flags "ad" (Authenticated Data) dans la réponse
```

---

## 🎭 DNS Spoofing

Le **DNS Spoofing** est un terme plus large qui englobe toute technique permettant de fournir de fausses réponses DNS. Le cache poisoning en est une forme, mais ce n'est pas la seule.

### Vecteurs d'attaque

```
1. Cache Poisoning       → Empoisonnement du résolveur (vu ci-dessus)
2. Man-in-the-Middle     → Interception sur le réseau local
3. Rogue DNS Server      → Faux serveur DNS sur un réseau compromis
4. BGP Hijacking         → Détournement des routes vers les vrais NS
5. Compromission du registrar → Modification des NS légitimes
```

### MITM DNS Spoofing (réseau local)

```bash
# Exemple pédagogique avec arpspoof (à des fins de test uniquement)
# Sur votre propre réseau de lab

# 1. Activer le forwarding IP
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. Empoisonner l'ARP cache de la victime
arpspoof -i eth0 -t <IP_VICTIME> <IP_GATEWAY>

# 3. Intercepter et modifier les réponses DNS (dnsspoof)
dnsspoof -i eth0 -f hosts.txt
# hosts.txt : "1.2.3.4 example.com"
```

### Détection et défense

```
✅ Utiliser DNS over HTTPS (DoH) ou DNS over TLS (DoT)
✅ DNSSEC pour valider l'authenticité des réponses
✅ Surveiller les changements d'enregistrements (monitoring)
✅ Valider les certificats TLS (HTTPS) même si le DNS est compromis
```

> **💡 Pourquoi HTTPS reste important même avec DNS spoofing** : même si l'attaquant redirige vers son serveur, il ne peut pas présenter un certificat TLS valide pour `example.com` sans le compromettre — le navigateur affichera une erreur.

---

## 📡 DHCP

Le **DHCP (Dynamic Host Configuration Protocol)** attribue automatiquement une configuration réseau aux équipements qui rejoignent un réseau.

### Ce que DHCP distribue

```
┌─────────────────────────────────┐
│         Bail DHCP               │
├─────────────────────────────────┤
│  Adresse IP   : 192.168.1.42    │
│  Masque       : 255.255.255.0   │
│  Passerelle   : 192.168.1.1     │
│  DNS primaire : 1.1.1.1         │
│  DNS secondaire: 8.8.8.8        │
│  Durée du bail: 24h             │
└─────────────────────────────────┘
```

### Le processus DORA

```
Client                          Serveur DHCP
  |                                  |
  |--- DISCOVER (broadcast) ------→  |  "Y a-t-il un serveur DHCP ?"
  |                                  |
  |←-- OFFER ----------------------- |  "Oui ! Je t'offre 192.168.1.42"
  |                                  |
  |--- REQUEST (broadcast) ------→   |  "J'accepte l'offre !"
  |                                  |
  |←-- ACK ------------------------- |  "C'est confirmé, bail de 24h"
  |                                  |
```

> **Pourquoi broadcast ?** Le client n'a pas encore d'IP, il ne peut pas envoyer en unicast. Le broadcast (`255.255.255.255`) atteint tous les équipements du segment.

### DHCP Snooping

Mécanisme de sécurité sur les switches managés qui bloque les serveurs DHCP non autorisés :

```
Port TRUNK (trusted)  ←→ Serveur DHCP légitime ✅
Port ACCESS (untrusted) ← Les OFFER/ACK sont bloqués ici ❌
                          (protection contre Rogue DHCP)
```

### Attaques DHCP

| Attaque | Principe | Impact |
|---------|----------|--------|
| **DHCP Starvation** | Épuiser le pool d'IPs avec de fausses requêtes | DoS : plus d'IP disponible |
| **Rogue DHCP Server** | Faux serveur distribuant de faux DNS/gateway | MITM total |

---

## 🔢 Adressage IP

### IPv4

Une adresse IPv4 = 32 bits = 4 octets en notation décimale pointée.

```
192    .    168    .    1    .    42
11000000  10101000  00000001  00101010
```

### Classes et RFC 1918 (adresses privées)

| Plage | Masque | Usages |
|-------|--------|--------|
| `10.0.0.0/8` | `/8` | Grandes entreprises |
| `172.16.0.0/12` | `/12` | Moyennes structures |
| `192.168.0.0/16` | `/16` | Réseaux domestiques |
| `127.0.0.0/8` | `/8` | Loopback (localhost) |
| `169.254.0.0/16` | `/16` | APIPA (pas de DHCP) |

### Notation CIDR

```
192.168.1.0/24

Adresse réseau : 192.168.1.0
Masque         : 255.255.255.0  (24 bits à 1)
Broadcast      : 192.168.1.255
Hosts dispo    : 192.168.1.1 → 192.168.1.254  (254 hôtes)
```

### Calcul rapide

```
/24 → 256 adresses  (254 utilisables)
/25 → 128 adresses  (126 utilisables)
/26 → 64 adresses   (62 utilisables)
/27 → 32 adresses   (30 utilisables)
/28 → 16 adresses   (14 utilisables)
/30 → 4 adresses    (2 utilisables) ← liens point-à-point
/32 → 1 adresse     ← host route
```

### IPv6

128 bits, notation hexadécimale groupée par blocs de 16 bits :

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
         ↓ simplification
2001:db8:85a3::8a2e:370:7334
```

---

## 🔀 NAT (Network Address Translation)

Le NAT permet à plusieurs équipements d'un réseau privé de partager une seule adresse IP publique.

### Types de NAT

**NAT Statique (1:1)**
```
192.168.1.10  ←→  203.0.113.10  (mapping fixe)
```

**NAT Dynamique (pool)**
```
192.168.1.10  ←→  203.0.113.10  (tiré d'un pool)
192.168.1.11  ←→  203.0.113.11
...
```

**PAT / NAT Overload (le plus courant)**
```
192.168.1.10:54321  ←→  203.0.113.1:10001
192.168.1.11:43210  ←→  203.0.113.1:10002
192.168.1.12:65432  ←→  203.0.113.1:10003
         ↑                        ↑
   IP privée + port          1 seule IP publique
                             + ports différents
```

### Table de translation (NAT Table)

```
┌──────────────────┬──────────────────┬─────────────┐
│ Source interne   │ Source externe   │  Destination │
├──────────────────┼──────────────────┼─────────────┤
│ 192.168.1.10:80  │ 203.0.113.1:1024 │ 8.8.8.8:443 │
│ 192.168.1.11:443 │ 203.0.113.1:1025 │ 1.1.1.1:53  │
└──────────────────┴──────────────────┴─────────────┘
```

### Limitations du NAT

```
❌ Complique les connexions entrantes (nécessite port forwarding)
❌ Casse certains protocoles (FTP actif, SIP, IPsec...)
❌ Introduit de la latence (lookup dans la table)
✅ Conserve les IPs IPv4
✅ "Masque" le réseau interne (sécurité par obscurité)
```

---

## 🏷️ VLAN (Virtual Local Area Network)

Un **VLAN** permet de segmenter logiquement un réseau physique en plusieurs réseaux isolés, sans câblage supplémentaire.

### Sans VLAN vs Avec VLAN

```
SANS VLAN :
Switch ──┬── PC Marketing
         ├── PC RH
         ├── PC Compta
         └── Serveur
(Tout le monde voit tout)

AVEC VLANs :
Switch ──┬── VLAN 10 : Marketing  (192.168.10.0/24)
         ├── VLAN 20 : RH         (192.168.20.0/24)
         ├── VLAN 30 : Compta     (192.168.30.0/24)
         └── VLAN 99 : Serveurs   (10.0.99.0/24)
(Isolation totale entre VLANs)
```

### Trunk vs Access

| Mode | Description | Usage |
|------|-------------|-------|
| **Access** | Un seul VLAN, la trame est envoyée sans tag | Port vers un PC/serveur |
| **Trunk** | Plusieurs VLANs, trames taggées (802.1Q) | Port entre switches, vers routeur |

### L'en-tête 802.1Q

```
Trame Ethernet normale :
[ Dst MAC | Src MAC | EtherType | Payload ]

Trame taggée 802.1Q :
[ Dst MAC | Src MAC | 0x8100 | PCP | DEI | VLAN ID (12 bits) | EtherType | Payload ]
                                                 ↑
                                          4096 VLANs possibles (0-4095)
```

### Routage inter-VLAN

Les VLANs sont isolés par défaut. Pour communiquer entre eux, il faut un **routeur** (ou un switch L3) :

```
Option 1 — Router-on-a-stick :
          Routeur
         /  (sous-interfaces)
        /
    Switch (trunk)
   /    \
VLAN10  VLAN20

Option 2 — Switch Layer 3 :
Switch L3 avec SVI (Switch Virtual Interface)
interface Vlan10 → IP 192.168.10.1/24
interface Vlan20 → IP 192.168.20.1/24
(routage matériel, plus performant)
```

### Configuration basique (Cisco IOS)

```bash
# Créer un VLAN
Switch(config)# vlan 10
Switch(config-vlan)# name Marketing

# Port en mode access
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

# Port en mode trunk
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30,99
```

---

## 🧪 Outils pratiques

```bash
# DNS
dig example.com                    # Requête DNS basique
dig +trace example.com             # Résolution complète pas à pas
dig -x 93.184.216.34               # Résolution inverse
nslookup example.com               # Alternative à dig
host example.com                   # Encore plus simple

# Réseau
ip addr show                       # Interfaces et IPs (Linux)
ip route show                      # Table de routage
arp -n                             # Table ARP
ss -tuln                           # Ports en écoute

# DHCP
dhclient -v eth0                   # Demander une IP (Linux)
ipconfig /release && ipconfig /renew  # Windows

# Capture et analyse
tcpdump -i eth0 port 53            # Capturer le trafic DNS
tcpdump -i eth0 port 67 or 68     # Capturer le trafic DHCP
wireshark                          # Analyse graphique
```

---

## 📖 Pour aller plus loin

| Sujet | RFC de référence |
|-------|-----------------|
| DNS | RFC 1034, RFC 1035 |
| DNSSEC | RFC 4033–4035 |
| DHCP | RFC 2131 |
| NAT | RFC 3022 |
| VLAN 802.1Q | IEEE 802.1Q |
| IPv6 | RFC 8200 |
| DNS over HTTPS | RFC 8484 |
| DNS over TLS | RFC 7858 |

---

<div align="center">

**Made with ❤️ — N'hésitez pas à ouvrir une issue ou une PR pour contribuer !**

</div>
