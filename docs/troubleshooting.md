# Difficultés rencontrées

## 1. Injection de la configuration

### Une interface référencée ne peut pas devenir membre SD-WAN
- **Symptôme** : l'ajout de `port1`, `port2` et des tunnels comme membres SD-WAN échoue.
- **Diagnostic** : ces interfaces étaient encore utilisées par des policies de pare-feu et par les routes statiques par défaut.
- **Solution** : `purge` des policies (suivi de `y`) et `delete` des routes statiques par défaut **avant** `config members`. Les objets d'adresse utilisés par les règles SD-WAN (`srv1`, `192.168.0.0`) doivent aussi exister avant les services. Les routes statiques vers 1.1.0.0/16 et 2.2.0.0/16 sont recréées après le bloc SD-WAN.

## 2. Compatibilité FortiOS

### `tunnel-search nexthop`
- **Symptôme** : commande non acceptée.
- **Diagnostic** : la documentation Fortinet indique que l'option est supprimée à partir de FortiOS 7.0.
- **Solution** : retirer la ligne.

### `adv-additional-path`
- **Symptôme** : `command parse error`, code -61, sur le voisin BGP des spokes.
- **Diagnostic** : la documentation 7.0 liste pourtant la commande. **Cause non établie.** Hypothèse non vérifiée : la commande n'est disponible que lorsque `additional-path` est en `send` ou `both`, alors que les spokes sont en `receive`.
- **Solution** : retirer la ligne. Les spokes ne font que recevoir des chemins, donc l'impact sur le comportement est a priori nul.

## 4. Mesures et interprétation

### Traceroute trompeur
- **Symptôme** : après bascule du trafic sur VPN-ISP2, le traceroute affichait toujours 10.254.1.254 (hub, overlay 1) au saut 2.
- **Diagnostic** : ce saut est l'adresse source que le hub met dans sa propre réponse ICMP. Ce trafic généré localement ne suit pas les règles SD-WAN. Les temps de réponse (59 à 160 ms au saut 2, 20 ms pour le serveur) allaient dans ce sens. C'est une explication, pas une preuve directe.
- **Solution** : prouver le chemin avec une capture de paquets sur FG-LONDRES (`diagnose sniffer packet`). Les requêtes et les réponses passaient toutes par VPN-ISP2.

### Captures polluées par les sondes
- **Symptôme** : les captures vers 8.8.8.8 et 192.168.0.2 montraient des paquets qui n'étaient pas ceux du test.
- **Diagnostic** : les adresses sources étaient celles du FortiGate lui-même, donc c'était le trafic des sondes SLA.
- **Solution** : filtre `greater 150` dans le sniffer et pings de grande taille (`ping -s 200`).




