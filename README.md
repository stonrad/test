# SD-WAN multi-sites avec FortiGate (GNS3)

Dans ce lab, nous réalisons une solution SD-WAN reliant un datacenter (hub) et deux agences (spokes) sur deux liens Internet simulés, avec des FortiGate virtuels sous GNS3.

## Topologie

Topologie ![`topologie.png`](docs/images/topologie.png).

| Site | Rôle | LAN | ISP1 | ISP2 |
|---|---|---|---|---|
| Paris | Hub, datacenter | 192.168.0.0/24 | 1.1.0.2/24 | 2.2.0.2/24 |
| London | Spoke | 192.168.1.0/24 | 1.1.1.2/24 | 2.2.1.2/24 |
| Atlanta | Spoke | 192.168.2.0/24 | 1.1.2.2/24 | 2.2.2.2/24 |

Chaque site dispose de deux accès Internet ISP1 et ISP2 (underlay)
Deux overlays IPsec (un par ISP) relient chaque spoke au hub. Les adresses overlay (10.254.1.0/24 et 10.254.2.0/24) sont distribuées dynamiquement par le hub (`mode-cfg`).

| Overlay | Réseau | IP PARIS | IP LONDRES | IP ATLANTA |
|---|---|---|---|---|
| ISP1 (VPN-ISP1) | 10.254.1.0/24 | 10.254.1.254 | 10.254.1.1 | 10.254.1.2 |
| ISP2 (VPN-ISP2) | 10.254.2.0/24 | 10.254.2.254 | 10.254.2.1 | 10.254.2.2 |

## Technologies

FortiGate VM (FortiOS : *`v7.0.15`*), GNS3, IPsec IKEv2, BGP (iBGP, route reflector, add-path), ADVPN, SD-WAN (règles, SLA, health-checks).

## Ce que fait la solution

- **Overlays IPsec** : deux tunnels par spoke, un par ISP, établis vers le hub.
- **Routage dynamique** : le hub est route reflector BGP. Un nouveau site est connu de tous sans reconfigurer les autres.
- **Choix du lien par SLA** : chaque spoke mesure latence, gigue et perte par overlay et oriente son trafic en conséquence.
- **Adhérence du trafic retour (traffic steering)** : le spoke annonce son LAN avec une community différente selon l'état du SLA (65000:1, 65000:2 ou 65000:11). Le hub la convertit en route-tag et l'utilise dans ses règles SD-WAN, pour que le retour suive le même lien que l'aller.
- **Shortcuts ADVPN** : le trafic entre deux agences déclenche un tunnel direct, sans repasser par le hub.

## Résultats observés

- **Dégradation de latence sur ISP1/ISP2** : le trafic vers le datacenter bascule sur VPN-ISP2/VPN-ISP1 repectivement. Une capture sur FG-LONDRES montre les requêtes et les réponses ICMP sur ce lien.
- 
- **Shortcut spoke à spoke** : le tunnel direct apparaît dans la table de routage de London (`VPN-ISP1_0`), et le next-hop BGP, conservé par le route reflector, est résolu par ce tunnel.

## Structure du dépôt

```
configs/      Configurations CLI des trois FortiGate
docs/         Architecture et journal des difficultés
tests/        Un fichier par scénario (commandes, sorties, conclusion)
evidence/     Captures de sorties de commandes
```

## Difficultés rencontrées

Détail dans [`docs/troubleshooting.md`](docs/troubleshooting.md). Les principales :

- ordre d'injection de la configuration (une interface référencée par une policy ou une route ne peut pas devenir membre SD-WAN) ;
- commandes non acceptées selon la version de FortiOS (`tunnel-search`, `adv-additional-path`) ;
- NAT appliqué aux flux inter-sites, qui masquait les adresses sources réelles ;
- `traceroute` trompeur : il affiche l'adresse source de la réponse du hub, pas le chemin des données. Le chemin réel a été prouvé par capture de paquets.

## Limites et pistes d'amélioration

- Propositions de chiffrement faibles (`des-md5`, `des-sha1`), acceptables en lab uniquement.
- Clé pré-partagée unique pour tous les sites. En production : certificats ou clés distinctes.
- Les secrets des fichiers de configuration sont remplacés par des placeholders (`<PSK>`).
