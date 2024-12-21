### Цель работы:
 - Настроите BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами с помошью протоколов iBGP или eBGP.

 Для лабороторной работы выберем протокол eBGP.

### Схема стенда
Схему для eBGP будем использовать без изменений, только поделим по AS.
![Topolpgy_eBGP.jpg](/Lab4/Topolpgy_eBGP.jpg)

Автономные системы будем использовать 2х байтные. 
Для Spine будем использовать общую AS, для Leaf'ов разные.

|Switch| AS |
|----|----|
|Spines|65001|
|Leaf.1|65101|
|Leaf.2|65102|
|Leaf.3|65103|

### Настройка протокола eBGP.

#### Настройки для Leaf'ов
- Включим процесс BGP и зададим AS `#router bgp 65101`
- Укажем `router-id` будем использовать адрес нашего _loopback1_ `10.1.1.1`
- Для упрощения конфигурации на Leaf у Arista есть возможность создавать _peer group_
  - Создадим `peer group Spines` и укажем нащих соседей. Соседство будем устанавливать на p2p линках.

  ```
  router bgp 65101
  router-id 10.1.1.1
  neighbor SPINES peer group
  neighbor SPINES remote-as 65001
  neighbor 10.2.1.0 peer group SPINES
  neighbor 10.2.1.0 description spine.1
  neighbor 10.2.2.0 peer group SPINES
  neighbor 10.2.2.0 description spine.2
  ```
- В  _address-family ipv4_ активируем нашу _peer group_ и укажем наши _Loopback_ для анонса.
```
neighbor SPINES activate
network 10.1.1.1/32
network 10.1.1.2/32
network 10.1.1.3/32
```
<details>
<summary>Проверим соседтво и увидем состаяние <b>idle</b> так как на <b>Spine</b>'ах ещё не запущен и не настроен BGP</summary>

```
Leaf.1#show ip bgp peer-group SPINES
BGP peer-group is SPINES, remote AS 65001, peer-group external
  BGP version 4
  Static peer-group members:
    VRF default:
      10.2.1.0, state: Idle
        Negotiated MP Capabilities:
            IPv4 Unicast: No
            IPv6 Unicast: No
            IPv4 SR-TE: No
            IPv6 SR-TE: No
      10.2.2.0, state: Idle
        Negotiated MP Capabilities:
            IPv4 Unicast: No
            IPv6 Unicast: No
            IPv4 SR-TE: No
            IPv6 SR-TE: No

```
</details>

Теперь настроим филтрацию маршрутов с помощью `route-map`, что бы принимать и анонсировать только те маршруты которые мы хотим. А хотим мы анонсировать _Loopback_ интерфейсы _Leaf_'ов, а принимать _Loopback_ интерфейсы _Leaf_'ов и _spine_'ов.
- создадим два `prefix-list` на экспорт и импорт. Укажем наши сети для _loopback_ интерфейсов из ip плана.
```
ip prefix-list EXPORT
   seq 10 permit 10.1.0.0/16 le 32
!
ip prefix-list IMPORT
   seq 10 permit 10.0.0.0/16 le 32
   seq 20 permit 10.1.0.0/16 le 32
```
- Далее настроим две _route-map_ на экспорт и импорт и заматчим туда наши _prefix-list_'ы.
```
route-map LF_EXPORT permit 10
   description Export Loopback from Leaf
   match ip address prefix-list EXPORT
!
route-map LF_IMPORT permit 10
   description Import Loopback from Spine and Leaf
   match ip address prefix-list IMPORT
```
- в _address-family ipv4_ для нашей _peer group_ настроим _route-map_ на импорт и экспорт.
```
neighbor SPINES route-map LF_IMPORT in
neighbor SPINES route-map LF_EXPORT out
```
- Настроим аутентификацию наших соседей что бы случайно не запирится с нежелательным соседом.
- включим BFD для увелечения сходимости сети.
- для балансировки исходящего трафика настроим _maximum-paths_ для двух _spine_
```
neighbor SPINES password 7 wMeqR2znTcw=
neighbor SPINES bfd
maximum-paths 2 ecmp 64
```
Итоговая конфигурация eBGP на _Leaf.1_:

```
router bgp 65101
   router-id 10.1.1.1
   maximum-paths 2 ecmp 64
   neighbor SPINES peer group
   neighbor SPINES remote-as 65001
   neighbor SPINES bfd
   neighbor SPINES password 7 wMeqR2znTcw=
   neighbor 10.2.1.0 peer group SPINES
   neighbor 10.2.1.0 description spine.1
   neighbor 10.2.2.0 peer group SPINES
   neighbor 10.2.2.0 description spine.2
   !
   address-family ipv4
      neighbor SPINES activate
      neighbor SPINES route-map LF_IMPORT in
      neighbor SPINES route-map LF_EXPORT out
      network 10.1.1.1/32
      network 10.1.1.2/32
      network 10.1.1.3/32
```
Конфигурация BGP на _Leaf_'ах будет отличаться только `router-id` и адресами `neighbor`

#### Настройки для Spine'ов
- Включим процесс BGP и зададим AS `#router bgp 65001`
- Укажем `router-id` будем использовать адрес нашего _loopback1_ `10.0.1.1`
- Создадим `peer group LEAFS_DC1` и укажем нащих соседей. Соседство будем устанавливать на p2p линках.
```
router bgp 65001
   router-id 10.0.1.1
   neighbor LEAFS_DC1 peer group
   network 10.0.1.1/32
   network 10.0.1.2/32
   network 10.0.1.3/32
```
- Создадим `peer-filter LEAFS_BGP` в котором укажем атрибуты по которым будем принимать входящие запросы от пиров. Резрешим только AS наших _Leaf_'ов

```
peer-filter LEAFS_BGP
   10 match as-range 65101 result accept
   20 match as-range 65102 result accept
   30 match as-range 65103 result accept

```
- Для `peer-group LEAFS_DC1` укажем диапозон адресов с которых принимать запросы на соседство. В диапазоне укажем подсеть для p2p линков и ограничим `peer-filter LEAFS_BGP`.
```
router bgp 65001
   router-id 10.0.1.1
   bgp listen range 10.2.0.0/16 peer-group LEAFS_DC1 peer-filter LEAFS_BGP
   neighbor LEAFS_DC1 peer group
   network 10.0.1.1/32
   network 10.0.1.2/32
   network 10.0.1.3/32
```

<details>
<summary>Проверим соседтво и маршрутную информацию. Увидем, что состояние у нас установилось и в таблице маршрутизации BGP у нас появились маршруты до <i>Loopback</i> интерфейсов <i>Leaf.1</i>. Также посмотрим <code>update</code> пакеты с <i>Leaf.1</i> и <i>Spine.1</i></summary>

```
BGP peer-group is LEAFS_DC1
  BGP version 4
  Listen-range subnets:
    VRF default:
      10.2.0.0/16, peer filter LEAFS_BGP
  Dynamic peer-group members:
    VRF default:
      10.2.1.1, state: Established
        Negotiated MP Capabilities:
            IPv4 Unicast: Yes
            IPv6 Unicast: No
            IPv4 SR-TE: No
            IPv6 SR-TE: No
```
```
Spine.1#show ip bgp 
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Route status codes: * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     10.0.1.1/32            -                     0       0       -       i
 * >     10.0.1.2/32            -                     0       0       -       i
 * >     10.0.1.3/32            -                     0       0       -       i
 * >     10.1.1.1/32            10.2.1.1              0       100     0       65101 i
 * >     10.1.1.2/32            10.2.1.1              0       100     0       65101 i
 * >     10.1.1.3/32            10.2.1.1              0       100     0       65101 i
```
<b>Leaf.1</b>
![update_leaf1.jpg](/Lab4/update_leaf1.jpg)
<b>Spine.1</b>
![update_spine1.jpg](/Lab4/update_spine1.jpg)
</details>

Настроим филтрацию маршрутов по примеру _Leaf_'ов. Анонсировать будем _Loopback_ интерфейсы _Spine_'ов, а принимать _Loopback_ интерфейсы _Leaf_'ов.
```
ip prefix-list EXPORT
   seq 10 permit 10.0.0.0/16 le 32
!
ip prefix-list IMPORT
   seq 10 permit 10.1.0.0/16 le 32
!
route-map SP_EXPORT permit 10
   description Export Loopback from Spine
   match ip address prefix-list EXPORT
!
route-map SP_IMPORT permit 10
   description Import Loopback from Leaf
   match ip address prefix-list IMPORT

```
- Включим `bfd` и аутентификацию
Итоговая конфигурация eBGP на _Spine.1_:
```
outer bgp 65001
   router-id 10.0.1.1
   bgp listen range 10.2.0.0/16 peer-group LEAFS_DC1 peer-filter LEAFS_BGP
   neighbor LEAFS_DC1 peer group
   neighbor LEAFS_DC1 bfd
   neighbor LEAFS_DC1 password 7 ZyukOQqAtng=
   network 10.0.1.1/32
   network 10.0.1.2/32
   network 10.0.1.3/32
   !
   address-family ipv4
      neighbor LEAFS_DC1 route-map SP_IMPORT in
      neighbor LEAFS_DC1 route-map SP_EXPORT out

```
Конфигурация BGP на _Leaf_'ах будет отличаться только `router-id`. 

Распространим настройки BGP на все коммутаторы и проверим таблицы маршрутизации и доступность _Loopback_ интерфейсов.

смотреть будем на _Leaf.1_

<details>
<summary>Проверим итоговую таблицу маршрутизации. Видим что `ecmp` работает, мы имеем маршруты до каждого <i>Loopback</i> интерфейса <i>Leaf</i> через каждый <i>Spine</i></summary>

```
Leaf.1#show ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65101
Route status codes: * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     10.0.1.1/32            10.2.1.0              0       100     0       65001 i
 * >     10.0.1.2/32            10.2.1.0              0       100     0       65001 i
 * >     10.0.1.3/32            10.2.1.0              0       100     0       65001 i
 * >     10.0.2.1/32            10.2.2.0              0       100     0       65001 i
 * >     10.0.2.2/32            10.2.2.0              0       100     0       65001 i
 * >     10.0.2.3/32            10.2.2.0              0       100     0       65001 i
 * >     10.1.1.1/32            -                     0       0       -       i
 * >     10.1.1.2/32            -                     0       0       -       i
 * >     10.1.1.3/32            -                     0       0       -       i
 * >Ec   10.1.2.1/32            10.2.2.0              0       100     0       65001 65102 i
 *  ec   10.1.2.1/32            10.2.1.0              0       100     0       65001 65102 i
 * >Ec   10.1.2.2/32            10.2.2.0              0       100     0       65001 65102 i
 *  ec   10.1.2.2/32            10.2.1.0              0       100     0       65001 65102 i
 * >Ec   10.1.2.3/32            10.2.2.0              0       100     0       65001 65102 i
 *  ec   10.1.2.3/32            10.2.1.0              0       100     0       65001 65102 i
 * >Ec   10.1.3.1/32            10.2.2.0              0       100     0       65001 65103 i
 *  ec   10.1.3.1/32            10.2.1.0              0       100     0       65001 65103 i
 * >Ec   10.1.3.2/32            10.2.2.0              0       100     0       65001 65103 i
 *  ec   10.1.3.2/32            10.2.1.0              0       100     0       65001 65103 i
 * >Ec   10.1.3.3/32            10.2.2.0              0       100     0       65001 65103 i
 *  ec   10.1.3.3/32            10.2.1.0              0       100     0       65001 65103 i

```
</details>

#### Проверим с _Leaf.1_ что все Loopback интерфейсы пингуются

<details>
<summary>Leaf.2</summary>

```
Leaf.1#ping 10.1.2.1 source 10.1.1.1
PING 10.1.2.1 (10.1.2.1) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=63 time=34.2 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=63 time=26.2 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=63 time=18.7 ms
80 bytes from 10.1.2.1: icmp_seq=4 ttl=63 time=22.4 ms
80 bytes from 10.1.2.1: icmp_seq=5 ttl=63 time=22.3 ms

--- 10.1.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 110ms
rtt min/avg/max/mdev = 18.782/24.805/34.204/5.259 ms, pipe 2, ipg/ewma 27.508/29.293 ms
Leaf.1#ping 10.1.2.2 source 10.1.1.1
PING 10.1.2.2 (10.1.2.2) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.2.2: icmp_seq=1 ttl=63 time=29.6 ms
80 bytes from 10.1.2.2: icmp_seq=2 ttl=63 time=24.5 ms
80 bytes from 10.1.2.2: icmp_seq=3 ttl=63 time=21.6 ms
80 bytes from 10.1.2.2: icmp_seq=4 ttl=63 time=22.1 ms
80 bytes from 10.1.2.2: icmp_seq=5 ttl=63 time=22.0 ms

--- 10.1.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 84ms
rtt min/avg/max/mdev = 21.668/24.024/29.645/3.000 ms, pipe 3, ipg/ewma 21.033/26.691 ms
Leaf.1#ping 10.1.2.3 source 10.1.1.1
PING 10.1.2.3 (10.1.2.3) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=63 time=21.8 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=63 time=14.2 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=63 time=15.2 ms
80 bytes from 10.1.2.3: icmp_seq=4 ttl=63 time=22.2 ms
80 bytes from 10.1.2.3: icmp_seq=5 ttl=63 time=28.1 ms

--- 10.1.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 62ms
rtt min/avg/max/mdev = 14.213/20.339/28.129/5.108 ms, pipe 3, ipg/ewma 15.522/21.403 ms
```
</details>

<details>
<summary>Leaf.3</summary>

```
Leaf.1#ping 10.1.3.1 source 10.1.1.1
PING 10.1.3.1 (10.1.3.1) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.3.1: icmp_seq=1 ttl=63 time=34.8 ms
80 bytes from 10.1.3.1: icmp_seq=2 ttl=63 time=23.1 ms
80 bytes from 10.1.3.1: icmp_seq=3 ttl=63 time=22.1 ms
80 bytes from 10.1.3.1: icmp_seq=4 ttl=63 time=31.9 ms
80 bytes from 10.1.3.1: icmp_seq=5 ttl=63 time=19.1 ms

--- 10.1.3.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 94ms
rtt min/avg/max/mdev = 19.120/26.253/34.825/6.057 ms, pipe 3, ipg/ewma 23.619/30.363 ms
Leaf.1#ping 10.1.3.2 source 10.1.1.1
PING 10.1.3.2 (10.1.3.2) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.3.2: icmp_seq=1 ttl=63 time=38.7 ms
80 bytes from 10.1.3.2: icmp_seq=2 ttl=63 time=30.9 ms
80 bytes from 10.1.3.2: icmp_seq=3 ttl=63 time=23.3 ms
80 bytes from 10.1.3.2: icmp_seq=4 ttl=63 time=29.7 ms
80 bytes from 10.1.3.2: icmp_seq=5 ttl=63 time=17.5 ms

--- 10.1.3.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 67ms
rtt min/avg/max/mdev = 17.593/28.083/38.792/7.183 ms, pipe 4, ipg/ewma 16.954/33.016 ms
Leaf.1#ping 10.1.3.3 source 10.1.1.1
PING 10.1.3.3 (10.1.3.3) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.1.3.3: icmp_seq=1 ttl=63 time=51.0 ms
80 bytes from 10.1.3.3: icmp_seq=2 ttl=63 time=46.0 ms
80 bytes from 10.1.3.3: icmp_seq=3 ttl=63 time=158 ms
80 bytes from 10.1.3.3: icmp_seq=4 ttl=63 time=152 ms

--- 10.1.3.3 ping statistics ---
5 packets transmitted, 4 received, 20% packet loss, time 89ms
rtt min/avg/max/mdev = 46.012/102.050/158.383/53.569 ms, pipe 4, ipg/ewma 22.334/75.042 ms
```
</details>

<details>
<summary>Spine.1</summary>

```
Leaf.1#ping 10.0.1.1 source 10.1.1.1
PING 10.0.1.1 (10.0.1.1) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=23.7 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=14.6 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=12.8 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=6.62 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=8.07 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 64ms
rtt min/avg/max/mdev = 6.627/13.198/23.748/6.050 ms, pipe 3, ipg/ewma 16.165/18.115 ms
Leaf.1#ping 10.0.1.2 source 10.1.1.1
PING 10.0.1.2 (10.0.1.2) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=16.8 ms
80 bytes from 10.0.1.2: icmp_seq=2 ttl=64 time=14.8 ms
80 bytes from 10.0.1.2: icmp_seq=3 ttl=64 time=11.9 ms
80 bytes from 10.0.1.2: icmp_seq=4 ttl=64 time=9.40 ms
80 bytes from 10.0.1.2: icmp_seq=5 ttl=64 time=8.15 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 62ms
rtt min/avg/max/mdev = 8.152/12.243/16.878/3.261 ms, pipe 2, ipg/ewma 15.724/14.327 ms
Leaf.1#ping 10.0.1.3 source 10.1.1.1
PING 10.0.1.3 (10.0.1.3) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.1.3: icmp_seq=1 ttl=64 time=22.8 ms
80 bytes from 10.0.1.3: icmp_seq=2 ttl=64 time=31.8 ms
80 bytes from 10.0.1.3: icmp_seq=3 ttl=64 time=40.5 ms
80 bytes from 10.0.1.3: icmp_seq=4 ttl=64 time=14.5 ms
80 bytes from 10.0.1.3: icmp_seq=5 ttl=64 time=9.68 ms

--- 10.0.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 80ms
rtt min/avg/max/mdev = 9.685/23.887/40.551/11.248 ms, pipe 3, ipg/ewma 20.069/22.725 ms
```
</details>

<details>
<summary>Spine.2 </summary>

```
Leaf.1#ping 10.0.2.1 source 10.1.1.1
PING 10.0.2.1 (10.0.2.1) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.2.1: icmp_seq=1 ttl=64 time=17.0 ms
80 bytes from 10.0.2.1: icmp_seq=2 ttl=64 time=19.1 ms
80 bytes from 10.0.2.1: icmp_seq=3 ttl=64 time=11.0 ms
80 bytes from 10.0.2.1: icmp_seq=4 ttl=64 time=10.1 ms
80 bytes from 10.0.2.1: icmp_seq=5 ttl=64 time=8.88 ms

--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 64ms
rtt min/avg/max/mdev = 8.886/13.261/19.129/4.067 ms, pipe 2, ipg/ewma 16.207/14.897 ms
Leaf.1#ping 10.0.2.2 source 10.1.1.1
PING 10.0.2.2 (10.0.2.2) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=16.5 ms
80 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=9.50 ms
80 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=10.7 ms
80 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=10.1 ms
80 bytes from 10.0.2.2: icmp_seq=5 ttl=64 time=10.6 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 56ms
rtt min/avg/max/mdev = 9.505/11.522/16.510/2.534 ms, pipe 2, ipg/ewma 14.029/13.950 ms
Leaf.1#ping 10.0.2.3 source 10.1.1.1
PING 10.0.2.3 (10.0.2.3) from 10.1.1.1 : 72(100) bytes of data.
80 bytes from 10.0.2.3: icmp_seq=1 ttl=64 time=23.0 ms
80 bytes from 10.0.2.3: icmp_seq=2 ttl=64 time=18.0 ms
80 bytes from 10.0.2.3: icmp_seq=3 ttl=64 time=20.8 ms
80 bytes from 10.0.2.3: icmp_seq=4 ttl=64 time=13.1 ms
80 bytes from 10.0.2.3: icmp_seq=5 ttl=64 time=9.02 ms

--- 10.0.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 65ms
rtt min/avg/max/mdev = 9.026/16.819/23.090/5.121 ms, pipe 3, ipg/ewma 16.347/19.602 ms
```
</details>

<details>
<summary>Проверим на всех <b>Spine</b>, что со всеми <b>Leaf</b> сессии поднялись.</summary>

```
Spine.1#show bfd peers 
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.2.1.1   642278866  1722446562        Ethernet1(13)  normal   12/21/24 13:41 
10.2.1.3   959512758  3382286518        Ethernet2(14)  normal   12/21/24 15:17 
10.2.1.5  2082088043  2469894083        Ethernet3(15)  normal   12/21/24 15:17 

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
10.2.2.1   496370478  3277325486        Ethernet1(13)  normal   12/21/24 15:51 
10.2.2.3   886698390  2589197142        Ethernet2(14)  normal   12/21/24 15:51 
10.2.2.5  3995632917   212054400        Ethernet3(15)  normal   12/21/24 15:51 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
</details>

Итог: eBGP соседство установлено, все маршруты распространены, bfd настроено.