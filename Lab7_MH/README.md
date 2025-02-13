## VxLAN. Аналоги VPC

### Цель работы:
 - рассмотреть EVPN route-type 1 и 4;
 - особенности работы, основное применение.

Для лабороторной работы в Underlay будем использовать OSPF, в Overlay iBGP.

### Схема стенда
Схему для VxLAN/EVPN  будем использовать из [Проекта](https://github.com/evsboroda/otus-design-dc/tree/main/Project) так как там функционал EVPN Multihoming уже настроен.

![alt text](Project/Scheme/DC-1_DC-LZ_Scheme.png)


Для клиентов будем использовать следующие vlan и ip сети.

|Vlan| Network|
|----|----|
|vlan 10|10.182.10.0/24|
|vlan 20|10.182.20.0/24|

Таблица MAC и IP адресов АРМ.
|АРМ|Switch|MAC|IP|Port|
|---|-----|---|--|---|
|SRV10-20|DC-LZ-SW-1|00:50:00:00:0d:00|10.182.10.20|Eth1|
|SRV10-30|DC-LZ-SW-2|00:50:00:00:17:00|10.182.10.30|Eth1|


|АМР10-3|Leaf.3|00:50:79:66:68:08|192.168.10.30|Eth5|
|АМР20-1|Leaf.2|00:50:79:66:68:09|192.168.20.10|Eth6|
|АМР20-2|Leaf.3|00:50:01:00:06:00|192.168.20.20|Eth6|

### Underlay
Для Underlay настроен на OSPF.

### Overlay
Для Overlay настроен iBGP.

### EVPN Multihoming
Технология, которая позволяет подключать устройства к нескольким Leaf для повышения отказоустойчивости и возможности балансировки трафика.
Multi-Homed устройством могут выступать как  L3 устройства так и устройства подключённые по технологии LAG.
В лабороторной работе будем использовать подключение пары Leaf к коммутаторам с помощью LAG.

#### Терминалогия

Ethernet Segment (ES) - Группа линков соеденяющих PE устройства (Leaf) с Multi-Homed устройствами.
Ethernet Segment Identifier (ESI) - уникальный идентификатор Ethernet Segment'а.
Designated forwarder - узел который выбирается для каждого Ethernet Segment и ответсвенен за пересылку BUM трафика.
ES-Import RT - Дополнительное комьюнити для генерации и приёма маршрутов Route Type-4.

EVPN Multihoming использует два дополнительных типа маршрута: Route type-1 Ethernet Auto-discovery (A-D) и Route type-4 Ethernet Segment Route.
- Route type-1 - используются для автоматического обноружения PE-маршрутизаторов, анонсов ESI, анонсов массовой отмены изученных MAC адресов при выходе из стройя одного из PE устройств.
- Route type-2 - используются для выбора DF и для обнаружения VTEP подключённых к одному Ethernet Segment.

#### Приступим к настройкам VXLAN MH.

- Сначала настроим обычный _Port-Channel_ .
```
interface Port-Channel1
   description DC-LZ-SW
   switchport trunk allowed vlan 10,20,30,2001
   switchport mode trunk
```
Так же `route-target import` и `lacp system-id`. Но при желании в них можно закодировать различные идентификаторы, например: номер площадки, номер пары Leaf, номер интерфейса, вобщем всё что в голову/

- Далее интерфейсу зададим н6астройки для ESI LAG.
- Укажем одинаковый `Ethernet Segment Identifier` на всех Leaf где будет это ES. 
- Укажем одинаковый `route-target import` на всех Leaf где есть данный ES.
- укажем одинаковый `lacp system-id` на всех Leaf где подключаются клиенты к этому ES, чтобы они считали, что подключены к одному устройству. 
```
evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 00:00:00:00:00:01
   lacp system-id 01aa.bbbb.0001
```

ESI выбирался из логики:
>Port-channel 1 = 0000:0000:0000:0000:0001\
>Port-channel 2 = 0000:0000:0000:0000:0002


- Добавим интерфейс в `Port-Channel`
- Включим LACP

```
interface Ethernet7
   description PO1_DC-LZ-SW
   channel-group 1 mode active
```
На этом настройка EVPN MH (ESI LAG) закончена. Теперь конфигурацию надо повторить на тех Leaf на которых будет этот же ES. 

### Итоговая конфигурция EVPN MH (ESI LAG) на _DC-LZ-LF-1_:

```
interface Port-Channel1
   description DC-LZ-SW
   switchport trunk allowed vlan 10,20,30,2001
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 00:00:00:00:00:01
   lacp system-id 01aa.bbbb.0001
!
interface Ethernet7
   description PO1_DC-LZ-SW
   channel-group 1 mode active
```

Скопируем настройки на *DC-LZ-LF-2*.

Для наглядности настроим ещё на одной паре Leaf (_DC-LZ-LF-3_ и _DC-LZ-LF-4_) ESI LAG.

```
interface Port-Channel1
   description SRV10-3
   switchport access vlan 10
   switchport trunk allowed vlan 10
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0002
      route-target import 00:00:00:00:00:02
   lacp system-id 01aa.bbbb.0002
!
interface Port-Channel2
   description MGMT
   switchport access vlan 2001
   switchport trunk allowed vlan 2001
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0003
      route-target import 00:00:00:00:00:03
   lacp system-id 01aa.bbbb.0003
!
interface Ethernet5
   description PO1_SRV10-3
   channel-group 1 mode active
!
interface Ethernet6
   description PO2_MGMT
   channel-group 2 mode active
```

##### Настроим LAG на коммутаторах доступа к которым подключаются клиенты.

- DC-LZ-SW-1

```
nterface Port-Channel1
   description PO_LF1-LF2
   switchport trunk allowed vlan 10,20,30,2001
   switchport mode trunk
!
interface Ethernet7
   description Po1_DC-LZ-LF-1
   channel-group 1 mode active
!
interface Ethernet8
   description Po1_DC-LZ-LF-2
   channel-group 1 mode active
```

- DC-LZ-SW-2

```
interface Port-Channel1
   description PO_LF3-LF4
   switchport trunk allowed vlan 10
   switchport mode trunk
!
interface Ethernet7
   description Po1_DC-LZ-LF-3
   channel-group 1 mode active
!
interface Ethernet8
   description Po1_DC-LZ-LF-4
   channel-group 1 mode active
```

- DC-LZ-SW-3

```
interface Port-Channel1
   description PO_LF3-LF4
   switchport trunk allowed vlan 2001
   switchport mode trunk
!
interface Ethernet7
   description Po1_DC-LZ-LF-3
   channel-group 1 mode active
!
interface Ethernet8
   description Po1_DC-LZ-LF-4
   channel-group 1 mode active
```

### Проверки

На _DC-LZ-LF-1_, посмотрим Route type 4

```
DC-LZ-LF-1#show bgp evpn route-type ethernet-segment 
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64600
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.1.2:1 ethernet-segment 0000:0000:0000:0000:0001 10.1.1.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.2.2:1 ethernet-segment 0000:0000:0000:0000:0001 10.1.2.2
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.2:1 ethernet-segment 0000:0000:0000:0000:0001 10.1.2.2
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.2:1 ethernet-segment 0000:0000:0000:0000:0002 10.1.3.2
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.2:1 ethernet-segment 0000:0000:0000:0000:0002 10.1.3.2
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.2:1 ethernet-segment 0000:0000:0000:0000:0002 10.1.4.2
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.2:1 ethernet-segment 0000:0000:0000:0000:0002 10.1.4.2
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.2:1 ethernet-segment 0000:0000:0000:0000:0003 10.1.3.2
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.2:1 ethernet-segment 0000:0000:0000:0000:0003 10.1.3.2
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.2:1 ethernet-segment 0000:0000:0000:0000:0003 10.1.4.2
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.2:1 ethernet-segment 0000:0000:0000:0000:0003 10.1.4.2
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 

```
\
Видим маршруты для каждого ES через те Leaf на которых он настроен.

Посмотрим подробный вывод. Видм что в `Extended Community` присутствует `EvpnEsImportRt` которая позволяет VTEP'ам импортировать Route type-4 маршруты только для тех сегментов которыет которые на них настроены.

```
DC-LZ-LF-1#show bgp evpn route-type ethernet-segment detail 
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64600
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.1.2, Route Distinguisher: 10.1.1.2:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.2.2, Route Distinguisher: 10.1.2.2:1
 Paths: 2 available
  Local
    10.1.2.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.2.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
  Local
    10.1.2.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.2.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0002 10.1.3.2, Route Distinguisher: 10.1.3.2:1
 Paths: 2 available
  Local
    10.1.3.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
  Local
    10.1.3.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0002 10.1.4.2, Route Distinguisher: 10.1.4.2:1
 Paths: 2 available
  Local
    10.1.4.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
  Local
    10.1.4.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0003 10.1.3.2, Route Distinguisher: 10.1.3.2:1
 Paths: 2 available
  Local
    10.1.3.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
  Local
    10.1.3.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0003 10.1.4.2, Route Distinguisher: 10.1.4.2:1
 Paths: 2 available
  Local
    10.1.4.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
  Local
    10.1.4.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
```
\
На _DC-LZ-LF-1_, посмотрим Route type 1.\
Видим какой VTEP сгенирировал для каждого VNI который присутствует в ESI свой маршрут.


```
DC-LZ-LF-1#show bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64600
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.1.1:10 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 10.1.1.1:20 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 10.1.1.1:30 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 10.1.1.1:2001 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.2.1:10 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.1:10 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.2.1:20 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.1:20 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.2.1:30 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.1:30 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.2.1:2001 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.1:2001 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >      RD: 10.1.1.2:1 auto-discovery 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.2.2:1 auto-discovery 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.2.2:1 auto-discovery 0000:0000:0000:0000:0001
                                 10.1.2.2              -       100     0       i Or-ID: 10.1.2.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.1:10 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.1:10 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.1:10 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.1:10 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.2:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.2:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.2:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.2:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.1:2001 auto-discovery 0 0000:0000:0000:0000:0003
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.1:2001 auto-discovery 0 0000:0000:0000:0000:0003
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.1:2001 auto-discovery 0 0000:0000:0000:0000:0003
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.1:2001 auto-discovery 0 0000:0000:0000:0000:0003
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.3.2:1 auto-discovery 0000:0000:0000:0000:0003
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.3.2:1 auto-discovery 0000:0000:0000:0000:0003
                                 10.1.3.2              -       100     0       i Or-ID: 10.1.3.1 C-LST: 10.0.1.1 
 * >Ec    RD: 10.1.4.2:1 auto-discovery 0000:0000:0000:0000:0003
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
 *  ec    RD: 10.1.4.2:1 auto-discovery 0000:0000:0000:0000:0003
                                 10.1.4.2              -       100     0       i Or-ID: 10.1.4.1 C-LST: 10.0.1.1 
```
\
Подробный вывод Route type-1

```
DC-LZ-LF-1#show bgp evpn route-type ethernet-segment detail 
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64600
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.1.2, Route Distinguisher: 10.1.1.2:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.2.2, Route Distinguisher: 10.1.2.2:1
 Paths: 2 available
  Local
    10.1.2.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.2.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
  Local
    10.1.2.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.2.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0002 10.1.3.2, Route Distinguisher: 10.1.3.2:1
 Paths: 2 available
  Local
    10.1.3.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
  Local
    10.1.3.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0002 10.1.4.2, Route Distinguisher: 10.1.4.2:1
 Paths: 2 available
  Local
    10.1.4.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
  Local
    10.1.4.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:02
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0003 10.1.3.2, Route Distinguisher: 10.1.3.2:1
 Paths: 2 available
  Local
    10.1.3.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
  Local
    10.1.3.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.3.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0003 10.1.4.2, Route Distinguisher: 10.1.4.2:1
 Paths: 2 available
  Local
    10.1.4.2 from 10.0.2.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP head, ECMP, best, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
  Local
    10.1.4.2 from 10.0.1.1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, internal, ECMP, ECMP contributor
      Originator: 10.1.4.1, Cluster list: 10.0.1.1 
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:03
```


 
<details>
<summary>Посмотрим детальный вывод маршрутов _EVPN_ на _Leaf.1_ после пинга.</summary>

```
Leaf.1#show bgp evpn route-type mac-ip detail 
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65101
BGP routing table entry for mac-ip 0050.0100.0b00, Route Distinguisher: 10.1.1.1:10
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.0100.0b00 192.168.10.10, Route Distinguisher: 10.1.1.1:10
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:10:10010 Route-Target-AS:5000:5000 TunnelEncap:tunnelTypeVxlan
      VNI: 10 L3 VNI: 5000 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6809, Route Distinguisher: 10.1.2.1:20
 Paths: 2 available
  65001 65102
    10.1.2.1 from fe80::5201:ff:fe4b:6277%Et2 (10.0.2.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 20 ESI: 0000:0000:0000:0000:0000
  65001 65102
    10.1.2.1 from fe80::5201:ff:fee5:e36a%Et1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 20 ESI: 0000:0000:0000:0000:0000
BGP routing table entry for mac-ip 0050.7966.6809 192.168.20.10, Route Distinguisher: 10.1.2.1:20
 Paths: 2 available
  65001 65102
    10.1.2.1 from fe80::5201:ff:fe4b:6277%Et2 (10.0.2.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 Route-Target-AS:5000:5000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:01:00:be:ab:97
      VNI: 20 L3 VNI: 5000 ESI: 0000:0000:0000:0000:0000
  65001 65102
    10.1.2.1 from fe80::5201:ff:fee5:e36a%Et1 (10.0.1.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:20:10020 Route-Target-AS:5000:5000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:01:00:be:ab:97
      VNI: 20 L3 VNI: 5000 ESI: 0000:0000:0000:0000:0000
```
<summary> В маршрутах MAC/IP видим по два RT, и <code>EvpnRouterMac:50:01:00:be:ab:97</code> svi интерфейса vlan 20 на Leaf.2 </summary>
</details>

Посмотрим *DUMP*  **ICMP** между _Leaf.1_ и _Sine.2_

![alt text](ICMP_L3.jpg)

В заголовке вдим уже не изначальный _VNI_ 10, а L3VNI _5000_.

<details>
<summary>C Leaf.1 проверим доступность клиентов в другой сети.</summary>
ARM10-1 -> ARM20-1

```
 root@ARM10-1:~# ping 192.168.20.10
PING 192.168.20.10 (192.168.20.10) 56(84) bytes of data.
64 bytes from 192.168.20.10: icmp_seq=1 ttl=62 time=172 ms
64 bytes from 192.168.20.10: icmp_seq=2 ttl=62 time=67.8 ms
64 bytes from 192.168.20.10: icmp_seq=3 ttl=62 time=64.9 ms
64 bytes from 192.168.20.10: icmp_seq=4 ttl=62 time=50.4 ms
64 bytes from 192.168.20.10: icmp_seq=5 ttl=62 time=58.3 ms
^C
--- 192.168.20.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4007ms
rtt min/avg/max/mdev = 50.436/82.712/172.157/45.120 ms

```
ARM10-1 -> ARM20-2
```
PING 192.168.20.20 (192.168.20.20) 56(84) bytes of data.
64 bytes from 192.168.20.20: icmp_seq=1 ttl=62 time=234 ms
64 bytes from 192.168.20.20: icmp_seq=2 ttl=62 time=123 ms
64 bytes from 192.168.20.20: icmp_seq=3 ttl=62 time=66.3 ms
64 bytes from 192.168.20.20: icmp_seq=4 ttl=62 time=154 ms
64 bytes from 192.168.20.20: icmp_seq=5 ttl=62 time=64.4 ms
64 bytes from 192.168.20.20: icmp_seq=6 ttl=62 time=67.9 ms
64 bytes from 192.168.20.20: icmp_seq=7 ttl=62 time=60.2 ms
^C
--- 192.168.20.20 ping statistics ---
7 packets transmitted, 7 received, 0% packet loss, time 6008ms
rtt min/avg/max/mdev = 60.199/109.889/234.353/60.675 ms
```
</details>

<details>
<summary>Проверим таблицу маршрутизации для VRF1. У нас есть как и ip-mac /32 маршруты полученные из Route-type 2, так и /24 полученные из Route-type 5</summary>

```
VRF: VRF1
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

 B E      192.168.10.20/32 [200/0] via VTEP 10.1.2.1 VNI 5000 router-mac 50:01:00:be:ab:97 local-interface Vxlan1
 B E      192.168.10.30/32 [200/0] via VTEP 10.1.3.1 VNI 5000 router-mac 50:01:00:27:03:91 local-interface Vxlan1
 C        192.168.10.0/24 is directly connected, Vlan10
 B E      192.168.20.10/32 [200/0] via VTEP 10.1.2.1 VNI 5000 router-mac 50:01:00:be:ab:97 local-interface Vxlan1
 B E      192.168.20.20/32 [200/0] via VTEP 10.1.3.1 VNI 5000 router-mac 50:01:00:27:03:91 local-interface Vxlan1
 B E      192.168.20.0/24 [200/0] via VTEP 10.1.2.1 VNI 5000 router-mac 50:01:00:be:ab:97 local-interface Vxlan1
                                  via VTEP 10.1.3.1 VNI 5000 router-mac 50:01:00:27:03:91 local-interface Vxlan1
```
</details>

Итог: Настроили L3 связанности между клиентами с использованием EVPN. С помощью *Route-type 5* решили проблему "молчаливых клиентов"



