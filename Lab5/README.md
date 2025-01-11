## VxLAN. EVPN L2 

### Цель работы:
 - рассмотреть route-type 2 и 3;
 - построение базовой overlay сети с помощью EVPN.

 Для лабороторной работы в Underlay будем использовать OSPF, в Overlay iBGP.

### Схема стенда
Схему для VxLAN/EVPN L2 будем использовать без изменений, только добавим клиентов.
![Topology_VxLan_EVPNL2.jpg](/Lab5/Topology_VxLan_EVPNL2.jpg)

Для клиентов будем использовать следующие vlan и ip сети.

|Vlan| Network|
|----|----|
|vlan 10|192.168.10.0/24|
|vlan 20|192.158.20.0/24|

### Underlay
Для Underlay OSPF будем использовать настройки из [Лабороторной 2](https://github.com/evsboroda/otus-design-dc/tree/main/Lab2)

### Overlay
В Overlay для Control-plane будем использовать EVPN (Ethernet VPN) который является стандартом [RFC 7432](https://datatracker.ietf.org/doc/html/rfc7432) и использует отдельную address family в Multi Protocol BGP. AFI 25 (l2vpn), SAFI 70 (evpn). Для этого в Overlay будем использовать iBGP.

EVPN позволяет передавать MAC-адрес так же, как и маршруты в анонсах BGP. EVPN использует 5ть типов маршрутов

*Route Type-1* - Ethernet Auto-Discovery Route. Для объявления Ethernet Segment Identifier (ESI) (конвергенция, балансировка).

*Route Type-2* - Host Advertisement Route. Для анонса информации о подключенных хостах.

*Route Type-3* - Inclusive Multicast Ethernet Tag Route (IMET). Для работы с BUM (Ingress Replication).

*Route Type-4* - Ethernet Segment Route. Для выбора DF (кто управляет BUM)

*Route Type-5* - IP-prefix route advertisement. Для анонса внешних маршрутов в фабрику.

Для настройки L2 связанности между клиентами с использованием EVPN будем использовать маршруты Type-2 и Type-3.

### Настройки iBGP на Leaf
- Включим процесс BGP и зададим AS `#router bgp 65101`
- Укажем `router-id` будем использовать адрес нашего _loopback1_.
- Создадим `peer group LEAFS` и укажем нащих соседей. Соседство будем устанавливать на интерфейсе Loopback1.
- Включим `send-community extended` так как EVPN их использует для распространения маршрутной информации.
- Включим `address-family evpn` и активируем соседей.
- Выключим `address-family ipv4` так как её мы не используем.
```
router bgp 65001
   router-id 10.1.1.1
   neighbor SPINES peer group
   neighbor SPINES remote-as 65001
   neighbor SPINES update-source Loopback1
   neighbor SPINES send-community extended
   neighbor 10.0.1.1 peer group SPINES
   neighbor 10.0.1.1 description Spine.1
   neighbor 10.0.2.1 peer group SPINES
   neighbor 10.0.2.1 description Spine.2
   !
   address-family evpn
      neighbor SPINES activate
   address-family ipv4
      no neighbor SPINES activate
```
### Настройки iBGP на Spine
- Включим процесс BGP и зададим AS `#router bgp 65101`
- Укажем `router-id` будем использовать адрес нашего _loopback1_.
- Создадим `peer group Spines` и укажем нащих соседей. Соседство будем устанавливать на интерфейсе Loopback1.
- Включим `send-community extended` так как EVPN их использует для распространения маршрутной информации.
- Включим `address-family evpn` и активируем соседей.
- Выключим `address-family ipv4` так как её мы не используем.
```
router bgp 65001
   router-id 10.0.1.1
   neighbor LEAFS peer group
   neighbor LEAFS remote-as 65001
   neighbor LEAFS update-source Loopback1
   neighbor LEAFS route-reflector-client
   neighbor LEAFS send-community extended
   neighbor 10.1.1.1 peer group LEAFS
   neighbor 10.1.2.1 peer group LEAFS
   neighbor 10.1.3.1 peer group LEAFS
   !
   address-family evpn
      neighbor LEAFS activate
   !
   address-family ipv4
      no neighbor LEAFS activate
```


```

Настроим VTI интерфейс.
- Настроим интерфейс Vxlan1
- насроим интефейс который будет использоваться при обмене VXLAN кадрами.
- Будем использовать VLAN-Based модель маппинга EVI к L2-домену. `vxlan vlan 10 vni 10`
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10
```
!
   vlan 10
      rd auto
      route-target both 10:1010
      redistribute learned
```