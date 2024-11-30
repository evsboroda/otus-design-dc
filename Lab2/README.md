## Underlay. OSPF
### Цель работы:
 - Настроить OSPF для Underlay сети.

Первым делом надо включить маршрутизацию.

```
ip routing
```
### Схема стенда
![Topology.jpg](/Lab2/Topology.jpg)

Базовая настройка OSPF на всех коммутаторах будет выглядеть одинаково. Отличаться будут только интерфейсы с включеным процессом OSPF.

Включение OSPF.

- Создаём процесс OSPF. В роли `router id` указываем адрес нашего `Loopback1`
- Переводим все интерфейсы в passive-режим, для уменьшения служебного траффика и защиты от нежелательных соседств.
- Включаем интерфейсы на которых будет установлено соседство - только на p2p интерфейсах.
- Включаем детальное логирование состояний соседства.

_Пример конфигурации OSPF для Leaf.1_
```
router ospf 10
   router-id 10.1.1.1
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   passive-interface Ethernet3
   max-lsa 12000
   log-adjacency-changes detail
```

Далее настраиваем интерфейсы участвующие в процессе OSPF.

- Включаем OSPF процесс на интерфейсе
- переводим интерфейс в режим сети point-to-point так как у нас все линки p2p, BR и BDR у нас выбираться не будут.
- Для безопасности и минимизации ошибок включаем аутентификацию MD5 на интерфейсах.
 
_пример конфигурации OSPF на интерфейсе eth1 для Leaf.1_
```
interface Ethernet1
   description Spine.1
   no switchport
   ip address 10.2.1.1/31
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 10 md5 7 DQRB1cPDQvk=
```
 
 После включения OSPF и настроек на интерфейсах на Leaf.1 и Spine.1 в логах видим сообщения об установлении соседства.

```
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <DOWN> to <DOWN>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <DOWN> to <INIT>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <INIT> to <2 WAYS>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <2 WAYS> to <EXCH START>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <EXCH START> to <EXCHANGE>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_STATE_CHANGE: NGB 10.0.1.1, intf 10.2.1.0 change from <EXCHANGE> to <FULL>
Nov 30 14:45:58 Leaf Rib: Instance 10: %OSPF-4-OSPF_ADJACENCY_ESTABLISHED: NGB 10.0.1.1, interface 10.2.1.1 adjacency established 
```

Для проверки смотрим команду `show ip ospf neighbor`
```
Leaf.1#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        10       default  0   FULL                   00:00:29    10.2.1.0        Ethernet1
```

После настроек OSPF на всех коммутаторах проверяем, что Spine установили соседство со всеми Leaf.

_Spine.1_
```
Spine.1(config)#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.1.1        10       default  0   FULL                   00:00:33    10.2.1.1        Ethernet1
10.1.2.1        10       default  0   FULL                   00:00:38    10.2.1.3        Ethernet2
10.1.3.1        10       default  0   FULL                   00:00:36    10.2.1.5        Ethernet3
```

_Spine.2_
```
Spine.2#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.1.1        10       default  0   FULL                   00:00:34    10.2.2.1        Ethernet1
10.1.2.1        10       default  0   FULL                   00:00:36    10.2.2.3        Ethernet2
10.1.3.1        10       default  0   FULL                   00:00:38    10.2.2.5        Ethernet3
```
<details>
<summary>Включаем OSPF на Loopback интерфейсах и на любом коммутаторе проверяем таблицу маршрутизации, что все адреса доступны.</summary> 

_Leaf.3_
```

 O        10.0.1.1/32 [110/20] via 10.2.1.4, Ethernet1 # Loopback1 Spine.1
 O        10.0.1.2/32 [110/20] via 10.2.1.4, Ethernet1 # Loopback2 Spine.1
 O        10.0.1.3/32 [110/20] via 10.2.1.4, Ethernet1 # Loopback3 Spine.1
 O        10.0.2.1/32 [110/20] via 10.2.2.4, Ethernet2 # Loopback1 Spine.2
 O        10.0.2.2/32 [110/20] via 10.2.2.4, Ethernet2 # Loopback2 Spine.2
 O        10.0.2.3/32 [110/20] via 10.2.2.4, Ethernet2 # Loopback3 Spine.2
 O        10.1.1.1/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback1 Leaf.1 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback1 Leaf.1 через Spine.2
 O        10.1.1.2/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback2 Leaf.1 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback2 Leaf.1 через Spine.2
 O        10.1.1.3/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback3 Leaf.1 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback3 Leaf.1 через Spine.2
 O        10.1.2.1/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback1 Leaf.2 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback1 Leaf.2 через Spine.2
 O        10.1.2.2/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback2 Leaf.2 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback2 Leaf.2 через Spine.2
 O        10.1.2.3/32 [110/30] via 10.2.1.4, Ethernet1 # Loopback3 Leaf.2 через Spine.1
                               via 10.2.2.4, Ethernet2 # Loopback3 Leaf.2 через Spine.2
 C        10.1.3.1/32 is directly connected, Loopback1
 C        10.1.3.2/32 is directly connected, Loopback2
 C        10.1.3.3/32 is directly connected, Loopback3
 O        10.2.1.0/31 [110/20] via 10.2.1.4, Ethernet1 # p2p сеть Leaf.1 - Spine.1
 O        10.2.1.2/31 [110/20] via 10.2.1.4, Ethernet1 # p2p сеть Leaf.2 - Spine.1
 C        10.2.1.4/31 is directly connected, Ethernet1
 O        10.2.2.0/31 [110/20] via 10.2.2.4, Ethernet2 # p2p сеть Leaf.1 - Spine.2
 O        10.2.2.2/31 [110/20] via 10.2.2.4, Ethernet2 # p2p сеть Leaf.2 - Spine.2
 C        10.2.2.4/31 is directly connected, Ethernet2
```
</details>

<details>
<summary>С <b>Leaf.1</b> проверим что Loopback'и пингуются.</summary>

```
Leaf.1#ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=57.2 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=49.0 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=50.9 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=44.2 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=29.7 ms
--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 52ms
rtt min/avg/max/mdev = 29.723/46.255/57.246/9.253 ms, pipe 5, ipg/ewma 13.171/51.101 ms
Leaf.1#ping 10.0.2.1
PING 10.0.2.1 (10.0.2.1) 72(100) bytes of data.
80 bytes from 10.0.2.1: icmp_seq=1 ttl=64 time=78.6 ms
80 bytes from 10.0.2.1: icmp_seq=2 ttl=64 time=55.3 ms
80 bytes from 10.0.2.1: icmp_seq=3 ttl=64 time=47.1 ms
80 bytes from 10.0.2.1: icmp_seq=4 ttl=64 time=38.9 ms
80 bytes from 10.0.2.1: icmp_seq=5 ttl=64 time=31.1 ms
```

```
--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 59ms
rtt min/avg/max/mdev = 31.196/50.263/78.622/16.312 ms, pipe 5, ipg/ewma 14.754/63.399 ms
Leaf.1#ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 72(100) bytes of data.
80 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=3.44 ms
80 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.034 ms
80 bytes from 10.1.1.1: icmp_seq=3 ttl=64 time=0.036 ms
80 bytes from 10.1.1.1: icmp_seq=4 ttl=64 time=0.032 ms
80 bytes from 10.1.1.1: icmp_seq=5 ttl=64 time=0.034 ms
--- 10.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 0.032/0.716/3.447/1.365 ms, ipg/ewma 3.342/2.034 ms
Leaf.1#ping 10.1.2.1
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=63 time=101 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=63 time=92.1 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=63 time=84.9 ms
80 bytes from 10.1.2.1: icmp_seq=4 ttl=63 time=78.5 ms
80 bytes from 10.1.2.1: icmp_seq=5 ttl=63 time=71.5 ms
```
```
--- 10.1.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 71.514/85.716/101.413/10.414 ms, pipe 5, ipg/ewma 11.220/92.824 ms
Leaf.1#ping 10.1.3.1
PING 10.1.3.1 (10.1.3.1) 72(100) bytes of data.
80 bytes from 10.1.3.1: icmp_seq=1 ttl=63 time=47.5 ms
80 bytes from 10.1.3.1: icmp_seq=2 ttl=63 time=38.1 ms
80 bytes from 10.1.3.1: icmp_seq=3 ttl=63 time=36.7 ms
80 bytes from 10.1.3.1: icmp_seq=4 ttl=63 time=30.0 ms
80 bytes from 10.1.3.1: icmp_seq=5 ttl=63 time=29.3 ms
--- 10.1.3.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 29.370/36.381/47.543/6.589 ms, pipe 5, ipg/ewma 11.129/41.542 ms
```
</details>

Для уменьшения скорости сходимости OSPF включим BFD на всех p2p интерфейсах.

```
ip ospf neighbor bfd
```
<details>
<summary>Проверим на всех <b>Spine</b>, что со всеми <b>Leaf</b> сессии поднялись.</summary>

```
Spine.1#show bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.1  2600936089  1481346065        Ethernet1(13)  normal   11/30/24 19:14 
10.2.1.3  4019426997  1178722661        Ethernet2(14)  normal   11/30/24 19:14 
10.2.1.5  4125477046  2585000598        Ethernet3(15)  normal   11/30/24 19:14 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
Spine.2# show bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.2.2.1  3073013940  4057919696        Ethernet1(13)  normal   11/30/24 19:18 
10.2.2.3  1813504941  4015901872        Ethernet2(14)  normal   11/30/24 19:18 
10.2.2.5  2343508969  2082118508        Ethernet3(15)  normal   11/30/24 19:18 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
</details>

Итоговая конфигурация интерфейса на всех коммутаторах будет выглядеть одинаково. Пример Spine.2.

```
interface Ethernet3
   description Leaf.3
   no switchport
   ip address 10.2.2.4/31
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 10 md5 7 epwCMyhnEnI=
```

Итог: OSPF соседство установлено, все маршруты есть, bfd настоено.