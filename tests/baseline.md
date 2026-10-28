# Test 01 : situation nominale

## Objectif

Vérifier que les overlays IPsec sont établis, que les sessions BGP sont ouvertes et que le hub connaît les LAN des agences avant d'introduire la moindre dégradation.

## 1. Tunnels IPsec sur le hub

Commande, sur FG-PARIS :

```
get vpn ipsec tunnel summary
```

Sortie relevée :

```
'VPN-ISP1_0' 1.1.1.2:0  selectors(total,up): 1/1  rx(pkt,err): 36171/0  tx(pkt,err): 30859/0
'VPN-ISP1_1' 1.1.2.2:0  selectors(total,up): 1/1  rx(pkt,err): 27399/0  tx(pkt,err): 22634/0
'VPN-ISP2_0' 2.2.1.2:0  selectors(total,up): 1/1  rx(pkt,err): 40220/0  tx(pkt,err): 25819/0
'VPN-ISP2_1' 2.2.2.2:0  selectors(total,up): 1/1  rx(pkt,err): 35416/0  tx(pkt,err): 22629/0
```

Lecture :
- Quatre tunnels sont établis : deux par spoke, un par ISP. Les adresses des pairs sont celles de London (1.1.1.2 et 2.2.1.2) et d'Atlanta (1.1.2.2 et 2.2.2.2).
- Chaque tunnel affiche `selectors 1/1` : le sélecteur de phase 2 est actif.
- Les compteurs ne montrent aucune erreur en réception ni en émission, et le trafic est en hausse (sondes SLA et BGP).
- Le suffixe `_0` ou `_1` correspond à l'ordre de connexion du spoke, pas à un site précis. Il peut changer après une reconnexion : c'est l'adresse du pair qui identifie le site.

## 2. Sessions BGP sur le hub

Commande, sur FG-PARIS :

```
get router info bgp summary
```

Sortie relevée :

```
VRF 0 BGP router identifier 200.0.0.2, local AS number 65000
BGP table version is 7

Neighbor   V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.254.1.1 4      65000    2280    2261        6    0    0 01:40:44        1
10.254.1.2 4      65000    2028    2033        6    0    0 01:30:06        1
10.254.2.1 4      65000    2273    2271        6    0    0 01:40:41        1
10.254.2.2 4      65000    2020    2036        6    0    0 01:30:07        1

Total number of neighbors 4
```

Lecture :
- Quatre voisins sont établis, deux par overlay, et chacun envoie un préfixe (le LAN de son site).
- Les deux voisins en `.2` sont apparus une dizaine de minutes après ceux en `.1`, ce qui correspond à l'ordre de connexion des spokes.
- Les files `InQ` et `OutQ` sont vides.

## 3. Routes du hub vers un LAN d'agence

Commande, sur FG-PARIS :

```
get router info routing-table details 192.168.1.2
```

Sortie relevée :

```
Routing table for VRF=0
Routing entry for 192.168.1.0/24
  Known via "bgp", distance 200, metric 0, best
  Last update 02:18:01 ago
  * 10.254.1.1 (recursive is directly connected, VPN-ISP1)
  * 10.254.2.1 (recursive is directly connected, VPN-ISP2)
```

Lecture :
- Le LAN de London (192.168.1.0/24) est appris en iBGP (distance 200) avec **deux chemins**, un par overlay.
- Les next-hops sont les adresses overlay du spoke (10.254.1.1 et 10.254.2.1), et non celles du hub : le route reflector conserve le next-hop d'origine.

## Conclusion

Avant toute dégradation, les quatre tunnels sont établis, les quatre sessions BGP sont ouvertes et le hub dispose de deux chemins par LAN d'agence. Ces relevés servent de référence aux tests suivants.

## Remarque

Les relevés des sections 1 et 2 viennent de deux captures prises à des moments différents. Les adresses overlay sont attribuées dynamiquement, donc elles ne sont pas forcément identiques d'un relevé à l'autre.