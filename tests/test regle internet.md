# Test 05 : sortie Internet (règle `internet`)

## Objectif

Déterminer par quelle interface sort le trafic de PC-LONDRES vers Internet, et comprendre comment la règle SD-WAN `internet` répartit les sessions.

## 1. Configuration de la règle

Commande, sur FG-LONDRES :

```
diagnose sys sdwan service 5
```

Sortie relevée :

```
Service(5): Address Mode(IPV4) flags=0x200 use-shortcut-sla
  Gen(15), TOS(0x0/0x0), Protocol(0: 1->65535), Mode(load-balance  hash-mode=round-robin)
  Service role: standalone
  Members(4):
    1: Seq_num(4 port2), alive, sla(0x1), gid(2), num of pass(1), selected
    2: Seq_num(3 port1), alive, sla(0x1), gid(2), num of pass(1), selected
    3: Seq_num(2 VPN-ISP2), alive, sla(0x1), gid(2), num of pass(1), selected
    4: Seq_num(1 VPN-ISP1), alive, sla(0x1), gid(2), num of pass(1), selected
  Dst address(1):
        0.0.0.0-255.255.255.255
```

Lecture :
- La règle est en `load-balance` avec un hash en **round-robin** : les sessions sont réparties tour à tour sur les membres conforme au SLA de la sonde `Default_Google Search`.
- Les quatre membres (port2, port1, VPN-ISP2, VPN-ISP1) sont `alive`, respectent le SLA (`sla(0x1)`) et sont tous `selected`.


## 2. Trafic de PC-LONDRES vers 8.8.8.8

Commande, sur FG-LONDRES :

```
diagnose sniffer packet any 'icmp and host 8.8.8.8 and greater 150' 4 12
```

Sortie relevée :

```
4.795064 port2 out 2.2.1.2 -> 8.8.8.8: icmp: echo request
4.796797 port2 in 8.8.8.8 -> 2.2.1.2: icmp: echo reply
4.796828 port10 out 8.8.8.8 -> 192.168.1.2: icmp: echo reply
6.392619 port10 in 192.168.1.2 -> 8.8.8.8: icmp: echo request
6.392711 port1 out 1.1.1.2 -> 8.8.8.8: icmp: echo request
6.395421 port1 in 8.8.8.8 -> 1.1.1.2: icmp: echo reply
6.395457 port10 out 8.8.8.8 -> 192.168.1.2: icmp: echo reply
7.782037 port10 in 192.168.1.2 -> 8.8.8.8: icmp: echo request
7.782142 VPN-ISP2 out 10.254.2.2 -> 8.8.8.8: icmp: echo request
7.790488 VPN-ISP2 in 8.8.8.8 -> 10.254.2.2: icmp: echo reply
7.790524 port10 out 8.8.8.8 -> 192.168.1.2: icmp: echo reply
8.990642 port10 in 192.168.1.2 -> 8.8.8.8: icmp: echo request
8.990699 VPN-ISP1 out 10.254.1.1 -> 8.8.8.8: icmp: echo request
8.996054 VPN-ISP1 in 8.8.8.8 -> 10.254.1.1: icmp: echo reply
```

| Session | Interface de sortie | Adresse source après NAT |
|---|---|---|
| 1 | port2 | 2.2.1.2 |
| 2 | port1 | 1.1.1.2 |
| 3 | VPN-ISP2 | 10.254.2.2 |
| 4 | VPN-ISP1 | 10.254.1.2 |

Lecture :
- Quatre  sessions successives sont parties par quatre interfaces différentes, ce qui correspond au round-robin de la règle.
- Dans chaque cas, FG-LONDRES applique son NAT : la source devient l'adresse de l'interface de sortie.

NB: Les captures réelles de trafic sont renseignées dans evidence

## Conclusion

Il n'existe pas d'interface de sortie unique pour Internet : la règle `internet` répartit les sessions en round-robin entre les quatre membres conformes au SLA. 
