### Цель работы:
 - Настроить IS-IS для Underlay сети.

### Схема стенда
Схему будем использовать без изменений.
![Topology.jpg](/Lab3_ISIS/Topology.jpg)


Как и в OSPF в IS-IS есть индефикатор области AREA называемый NET (Network Entity Title) который имеет следущий формат:
| AFI | Area ID | SysID | SEL |
| --- | --- | --- | --- |
| 49 | 0001 | 0100.0100.1001 | 00 |
- AFI - Равен 49, так как относится к классу локальных адресов.
- Area ID - поле переменной длины, обозначает область к которой принадлежит маршрутизатор. В нашем случае область 1 (0001) 
- SysID - поле переменной длины, индификатор маршрутизатора. В нашем случае будем использовать адреса интерфейсов Loopback 1. 
Добовляем нули: Leaf.1 10.1.1.1 -> 010.001.001.001 -> 0100.0100.1001
Так-как ISIS исполузует TLV Type 137 (Dynamic Name), то запоминать SysID нет особой необходимости. Мы будем в ISIS везде пользоваться Hostname наших коммутаторов.
- SEL - в IP сетях выставляем нули (00).

#### Настройка протокола IS-IS.

- Включаем IS-IS и называем имя процесса по номеру нашей проектируемой площадки, "dc1"
- Зададим NET для коммутатора
- Включим address-family для протокола ipv4
- Везде будем использовать соседство Type L2, что бы иметь одну базу LSDB L2 и домен L2.
_Пример конфигурации IS-IS для Leaf.1_
```
router isis dc1
   net 49.0001.0100.0100.1001.00
   is-type level-2
   !
   address-family ipv4 unicast
```

Далее переходим к настройкам интерфейсов.
- Включаем IS-IS процесс на интерфейсе
- переводим интерфейс в режим сети point-to-point так как у нас все линки p2p.
- Для безопасности и минимизации ошибок включаем аутентификацию MD5 на интерфейсах.
- на интерфейсах не участвующих в установлении соседства включим пассивный режим `#isis passive` 
_пример конфигурации IS-IS на интерфейсе eth1 для Leaf.1_
```
nterface Ethernet1
   description Spine.1
   no switchport
   ip address 10.2.1.1/31
   isis enable dc1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 9q99ta8yYgo=
```
После настроек ISIS на всех коммутаторах проверяем, что Spine установили соседство со всеми Leaf.

_Spine.1_
```
Spine.1#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
dc1       default  Leaf.1           L2   Ethernet1          P2P               UP    22          0D                  
dc1       default  Leaf.2           L2   Ethernet2          P2P               UP    30          0D                  
dc1       default  Leaf.3           L2   Ethernet3          P2P               UP    23          0D      
```
_Spine.2_
```
Spine.2#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
dc1       default  Leaf.1           L2   Ethernet1          P2P               UP    26          0E                  
dc1       default  Leaf.2           L2   Ethernet2          P2P               UP    27          0E                  
dc1       default  Leaf.3           L2   Ethernet3          P2P               UP    23          0E         
```
<details>
<summary>Включаем ISIS на Loopback интерфейсах и на любом коммутаторе проверяем таблицу маршрутизации, что все адреса доступны.</summary> 

_Leaf.2_
```
Leaf.2#show ip route 

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 I L2     10.0.1.1/32 [115/20] via 10.2.1.2, Ethernet1 # Loopback1 Spine.1
 I L2     10.0.1.2/32 [115/20] via 10.2.1.2, Ethernet1 # Loopback2 Spine.1
 I L2     10.0.1.3/32 [115/20] via 10.2.1.2, Ethernet1 # Loopback3 Spine.1
 I L2     10.0.2.1/32 [115/20] via 10.2.2.2, Ethernet2 # Loopback1 Spine.2
 I L2     10.0.2.2/32 [115/20] via 10.2.2.2, Ethernet2 # Loopback2 Spine.2
 I L2     10.0.2.3/32 [115/20] via 10.2.2.2, Ethernet2 # Loopback3 Spine.2
 I L2     10.1.1.1/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback1 Leaf.1 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback1 Leaf.1 через Spine.2
 I L2     10.1.1.2/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback2 Leaf.1 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback2 Leaf.1 через Spine.2
 I L2     10.1.1.3/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback3 Leaf.1 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback3 Leaf.1 через Spine.2
 C        10.1.2.1/32 is directly connected, Loopback1
 C        10.1.2.2/32 is directly connected, Loopback2
 C        10.1.2.3/32 is directly connected, Loopback3
 I L2     10.1.3.1/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback1 Leaf.3 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback1 Leaf.3 через Spine.2
 I L2     10.1.3.2/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback2 Leaf.3 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback2 Leaf.3 через Spine.2
 I L2     10.1.3.3/32 [115/30] via 10.2.1.2, Ethernet1 # Loopback3 Leaf.3 через Spine.1
                               via 10.2.2.2, Ethernet2 # Loopback3 Leaf.3 через Spine.2
 I L2     10.2.1.0/31 [115/20] via 10.2.1.2, Ethernet1 # p2p сеть Leaf.1 - Spine.1
 C        10.2.1.2/31 is directly connected, Ethernet1
 I L2     10.2.1.4/31 [115/20] via 10.2.1.2, Ethernet1 # p2p сеть Leaf.3 - Spine.1
 I L2     10.2.2.0/31 [115/20] via 10.2.2.2, Ethernet2 # p2p сеть Leaf.1 - Spine.2
 C        10.2.2.2/31 is directly connected, Ethernet2
 I L2     10.2.2.4/31 [115/20] via 10.2.2.2, Ethernet2 # p2p сеть Leaf.3 - Spine.2
 ```
</details>

#### Проверим Проверим LSDB L1 и L2. 
Так как мы везде используем маршрутизаторы Type L2 то LSDB L1 пустая.
```
Leaf.2#show isis database level-1
Leaf.2#
```
<details>
<summary>Проверим LSDB L2</summary>

```
Leaf.2#show isis database level-2 detail 

IS-IS Instance: dc1 VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Spine.1.00-00               689  23148  1084    173 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: Spine.1
      Area addresses: 49.0001
      Interface address: 10.0.1.3
      Interface address: 10.0.1.2
      Interface address: 10.0.1.1
      Interface address: 10.2.1.4
      Interface address: 10.2.1.2
      Interface address: 10.2.1.0
      IS Neighbor          : Leaf.3.00           Metric: 10
      IS Neighbor          : Leaf.2.00           Metric: 10
      IS Neighbor          : Leaf.1.00           Metric: 10
      Reachability         : 10.0.1.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.1.1/32 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.0/31 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.0.1.3 Flags: []
        Area leader priority: 250 algorithm: 0
    Spine.2.00-00               687  57035   596    173 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: Spine.2
      Area addresses: 49.0001
      Interface address: 10.0.2.3
      Interface address: 10.0.2.2
      Interface address: 10.0.2.1
      Interface address: 10.2.2.4
      Interface address: 10.2.2.2
      Interface address: 10.2.2.0
      IS Neighbor          : Leaf.3.00           Metric: 10
      IS Neighbor          : Leaf.2.00           Metric: 10
      IS Neighbor          : Leaf.1.00           Metric: 10
      Reachability         : 10.0.2.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.2.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.0.2.1/32 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.0/31 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.0.2.3 Flags: []
        Area leader priority: 250 algorithm: 0
    Leaf.1.00-00                703   2514   994    148 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: Leaf.1
      Area addresses: 49.0001
      Interface address: 10.1.1.3
      Interface address: 10.1.1.2
      Interface address: 10.1.1.1
      Interface address: 10.2.2.1
      Interface address: 10.2.1.1
      IS Neighbor          : Spine.2.00          Metric: 10
      IS Neighbor          : Spine.1.00          Metric: 10
      Reachability         : 10.1.1.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.1.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.1.1/32 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.0/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.0/31 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.1.3 Flags: []
        Area leader priority: 250 algorithm: 0
    Leaf.2.00-00                680  46110  1124    148 L2 <>
      LSP generation remaining wait time: 0 ms
      Time remaining until refresh: 824 s
      NLPID: 0xCC(IPv4)
      Hostname: Leaf.2
      Area addresses: 49.0001
      Interface address: 10.1.2.3
      Interface address: 10.1.2.2
      Interface address: 10.1.2.1
      Interface address: 10.2.2.3
      Interface address: 10.2.1.3
      IS Neighbor          : Spine.2.00          Metric: 10
      IS Neighbor          : Spine.1.00          Metric: 10
      Reachability         : 10.1.2.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.1/32 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.2/31 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.2.3 Flags: []
        Area leader priority: 250 algorithm: 0
    Leaf.3.00-00                680  58063   695    148 L2 <>
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: Leaf.3
      Area addresses: 49.0001
      Interface address: 10.1.3.3
      Interface address: 10.1.3.2
      Interface address: 10.1.3.1
      Interface address: 10.2.2.5
      Interface address: 10.2.1.5
      IS Neighbor          : Spine.1.00          Metric: 10
      IS Neighbor          : Spine.2.00          Metric: 10
      Reachability         : 10.1.3.3/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.3.2/32 Metric: 10 Type: 1 Up
      Reachability         : 10.1.3.1/32 Metric: 10 Type: 1 Up
      Reachability         : 10.2.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.2.1.4/31 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.3.3 Flags: []
        Area leader priority: 250 algorithm: 0
```
</details>

#### Проверим с _Leaf.1_ что все Loopback интерфейсы пингуются

<details>
<summary>Leaf.2</summary>

```
Leaf.1#ping 10.1.2.1
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=63 time=44.6 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=63 time=35.7 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=63 time=30.8 ms
80 bytes from 10.1.2.1: icmp_seq=4 ttl=63 time=26.9 ms
80 bytes from 10.1.2.1: icmp_seq=5 ttl=63 time=20.3 ms

--- 10.1.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 77ms
rtt min/avg/max/mdev = 20.391/31.716/44.614/8.177 ms, pipe 4, ipg/ewma 19.495/37.597 ms
Leaf.1#ping 10.1.2.2
PING 10.1.2.2 (10.1.2.2) 72(100) bytes of data.
80 bytes from 10.1.2.2: icmp_seq=1 ttl=63 time=38.8 ms
80 bytes from 10.1.2.2: icmp_seq=2 ttl=63 time=32.0 ms
80 bytes from 10.1.2.2: icmp_seq=3 ttl=63 time=46.4 ms
80 bytes from 10.1.2.2: icmp_seq=4 ttl=63 time=41.2 ms
80 bytes from 10.1.2.2: icmp_seq=5 ttl=63 time=21.1 ms

--- 10.1.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 75ms
rtt min/avg/max/mdev = 21.167/35.944/46.468/8.730 ms, pipe 4, ipg/ewma 18.951/37.043 ms
Leaf.1#ping 10.1.2.3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=63 time=41.8 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=63 time=37.6 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=63 time=33.4 ms
80 bytes from 10.1.2.3: icmp_seq=4 ttl=63 time=40.6 ms
80 bytes from 10.1.2.3: icmp_seq=5 ttl=63 time=24.4 ms

--- 10.1.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 72ms
rtt min/avg/max/mdev = 24.497/35.615/41.869/6.263 ms, pipe 4, ipg/ewma 18.211/38.401 ms
```
</details>

<details>
<summary>Leaf.3</summary>

```
Leaf.1#ping 10.1.3.1
PING 10.1.3.1 (10.1.3.1) 72(100) bytes of data.
80 bytes from 10.1.3.1: icmp_seq=1 ttl=63 time=29.6 ms
80 bytes from 10.1.3.1: icmp_seq=2 ttl=63 time=21.9 ms
80 bytes from 10.1.3.1: icmp_seq=3 ttl=63 time=17.8 ms
80 bytes from 10.1.3.1: icmp_seq=4 ttl=63 time=23.4 ms
80 bytes from 10.1.3.1: icmp_seq=5 ttl=63 time=19.7 ms

--- 10.1.3.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 98ms
rtt min/avg/max/mdev = 17.858/22.535/29.684/4.053 ms, pipe 2, ipg/ewma 24.639/25.978 ms
Leaf.1#ping 10.1.3.2
PING 10.1.3.2 (10.1.3.2) 72(100) bytes of data.
80 bytes from 10.1.3.2: icmp_seq=1 ttl=63 time=37.4 ms
80 bytes from 10.1.3.2: icmp_seq=2 ttl=63 time=29.2 ms
80 bytes from 10.1.3.2: icmp_seq=3 ttl=63 time=24.8 ms
80 bytes from 10.1.3.2: icmp_seq=4 ttl=63 time=34.0 ms
80 bytes from 10.1.3.2: icmp_seq=5 ttl=63 time=21.9 ms

--- 10.1.3.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 71ms
rtt min/avg/max/mdev = 21.974/29.526/37.457/5.698 ms, pipe 4, ipg/ewma 17.762/33.260 ms
Leaf.1#ping 10.1.3.3
PING 10.1.3.3 (10.1.3.3) 72(100) bytes of data.
80 bytes from 10.1.3.3: icmp_seq=1 ttl=63 time=92.7 ms
80 bytes from 10.1.3.3: icmp_seq=2 ttl=63 time=87.0 ms
80 bytes from 10.1.3.3: icmp_seq=3 ttl=63 time=82.1 ms
80 bytes from 10.1.3.3: icmp_seq=4 ttl=63 time=78.2 ms
80 bytes from 10.1.3.3: icmp_seq=5 ttl=63 time=73.0 ms

--- 10.1.3.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 42ms
rtt min/avg/max/mdev = 73.084/82.633/92.747/6.826 ms, pipe 5, ipg/ewma 10.719/87.201 ms
Leaf.1#ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=24.4 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=22.9 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=10.7 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=9.97 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=8.91 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 83ms
rtt min/avg/max/mdev = 8.919/15.390/24.452/6.808 ms, pipe 2, ipg/ewma 20.942/19.481 ms
```
</details>

<details>
<summary>Spine.1</summary>

```
Leaf.1#ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=24.4 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=22.9 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=10.7 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=9.97 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=8.91 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 83ms
rtt min/avg/max/mdev = 8.919/15.390/24.452/6.808 ms, pipe 2, ipg/ewma 20.942/19.481 ms
Leaf.1#ping 10.0.1.2
PING 10.0.1.2 (10.0.1.2) 72(100) bytes of data.
80 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=17.9 ms
80 bytes from 10.0.1.2: icmp_seq=2 ttl=64 time=18.2 ms
80 bytes from 10.0.1.2: icmp_seq=3 ttl=64 time=12.2 ms
80 bytes from 10.0.1.2: icmp_seq=4 ttl=64 time=9.23 ms
80 bytes from 10.0.1.2: icmp_seq=5 ttl=64 time=9.81 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 64ms
rtt min/avg/max/mdev = 9.236/13.507/18.291/3.898 ms, pipe 2, ipg/ewma 16.193/15.457 ms
Leaf.1#ping 10.0.1.3
PING 10.0.1.3 (10.0.1.3) 72(100) bytes of data.
80 bytes from 10.0.1.3: icmp_seq=1 ttl=64 time=17.4 ms
80 bytes from 10.0.1.3: icmp_seq=2 ttl=64 time=15.7 ms
80 bytes from 10.0.1.3: icmp_seq=3 ttl=64 time=11.1 ms
80 bytes from 10.0.1.3: icmp_seq=4 ttl=64 time=9.12 ms
80 bytes from 10.0.1.3: icmp_seq=5 ttl=64 time=10.1 ms

--- 10.0.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 63ms
rtt min/avg/max/mdev = 9.127/12.708/17.414/3.274 ms, pipe 2, ipg/ewma 15.752/14.855 ms
```
</details>

<details>
<summary>Spine.2 </summary>

```
Leaf.1#ping 10.0.2.1
PING 10.0.2.1 (10.0.2.1) 72(100) bytes of data.
80 bytes from 10.0.2.1: icmp_seq=1 ttl=64 time=30.1 ms
80 bytes from 10.0.2.1: icmp_seq=2 ttl=64 time=18.5 ms
80 bytes from 10.0.2.1: icmp_seq=3 ttl=64 time=10.5 ms
80 bytes from 10.0.2.1: icmp_seq=4 ttl=64 time=9.34 ms
80 bytes from 10.0.2.1: icmp_seq=5 ttl=64 time=11.7 ms

--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 95ms
rtt min/avg/max/mdev = 9.348/16.069/30.117/7.718 ms, pipe 2, ipg/ewma 23.903/22.710 ms
Leaf.1#ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 72(100) bytes of data.
80 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=28.3 ms
80 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=17.8 ms
80 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=20.5 ms
80 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=11.7 ms
80 bytes from 10.0.2.2: icmp_seq=5 ttl=64 time=13.3 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 76ms
rtt min/avg/max/mdev = 11.765/18.370/28.366/5.890 ms, pipe 3, ipg/ewma 19.230/23.042 ms
Leaf.1#ping 10.0.2.3
PING 10.0.2.3 (10.0.2.3) 72(100) bytes of data.
80 bytes from 10.0.2.3: icmp_seq=1 ttl=64 time=24.6 ms
80 bytes from 10.0.2.3: icmp_seq=2 ttl=64 time=31.7 ms
80 bytes from 10.0.2.3: icmp_seq=3 ttl=64 time=24.4 ms
80 bytes from 10.0.2.3: icmp_seq=4 ttl=64 time=9.87 ms
80 bytes from 10.0.2.3: icmp_seq=5 ttl=64 time=10.7 ms

--- 10.0.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 70ms
rtt min/avg/max/mdev = 9.875/20.297/31.769/8.589 ms, pipe 3, ipg/ewma 17.552/21.868 ms
```
</details>

Для уменьшения скорости сходимости ISIS включим BFD на всех p2p интерфейсах
```
#isis bfd
```

<details>
<summary>Проверим на всех <b>Spine</b>, что со всеми <b>Leaf</b> сессии поднялись.</summary>

```
Spine.1#show bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.1  2713730390    21420236        Ethernet1(13)  normal   12/14/24 19:58 
10.2.1.3  1897969859  3699434593        Ethernet2(14)  normal   12/14/24 19:58 
10.2.1.5  4230091444  3313479353        Ethernet3(15)  normal   12/14/24 19:58 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

```
Spine.2#show bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.2.2.1   714393955  2465194990        Ethernet1(13)  normal   12/14/24 19:58 
10.2.2.3   538390384  3331822893        Ethernet2(14)  normal   12/14/24 19:58 
10.2.2.5  1770922390  3424751825        Ethernet3(15)  normal   12/14/24 19:58 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
</details>

 Итог: ISIS соседство установлено, все маршруты распространены, bfd настроено.

