## Проектирование адресного пространства

### цель:

1.  Собрать схему CLOS
2.  Распределить адресное пространство для underlay сети

### При планировании, все адреса будем выдвать из подсети 10.0.0.0/8 и придерживаться следующего принципа:

- Подсеть каждого DC берётся из диапазона 10.0.0.0/13. Например:
    
    - DC1 - 10.0.0.0/13
    - DC2 - 10.8.0.0/13
    - и т.д
- Для loopback интерфейсов используем маску /32
    
- Для p2p линков используем подсети /31 и старший адрес в подсети назначаем на Leaf.
    
- Второй октет внутри подсети DC (10.**0**.0.0/13) используем для указания Spine, Leaf, p2p линков, резерва и сервисов. Пример:
    
    - 10.**0**.0.0/16 - Диапазон Loopback адресов для Spine.
    - 10.**1**.0.0/16 - Диапазон Loopback адресов для Leaf.
    - 10.**2**.0.0/16 - Диапазон для p2p линков.
    - 10.**3**.0.0/16 - Резерв.
    - 10.**4**.0.0/16 - Диапазон для для подключения различных систем: Firewall, Edge Router и т.д.
    - 10.**5**.0.0/16 - Диапазон для сервисов.
    - 10.**6**.0.0/16 - Диапазон для сервисов
    - 10.**7**.0.0/16 - Диапазон для сервисов
- Третий октет внутри подсети DC (10.0.**0**.0/13) используем для указания принадлежности Loopback и p2p адресов к конкретному Spine и Leaf. Например:
    
    - 10.0.1.0/24 - диапазон Loopback адресов для Spine_1 в DC1
    - 10.1.3.0/24 - диапазон Loopback адресов для Leaf_3 в DC1
    - 10.2.1.2/31 - подсеть для p2p между Spine_1 и Leaf_2

В Лабораторных будем использовать следующую топологию  
![Lab1.jpg](/Lab1/Lab1.jpg)

### Таблица Loopback адресов

Все Loopback адреса задаются с маской /32

| **Device** | **Loopback1** | **Loopback2** | **Loopback3** |
| --- | --- | --- | --- |
| **spine.1** | 10.0.1.1/32 | 10.0.1.2/32 | 10.0.1.3/32 |
| **spine.2** | 10.0.2.1/32 | 10.0.2.2/32 | 10.0.2.3/32 |
| **leaf.1** | 10.1.1.1/32 | 10.1.1.2/32 | 10.1.1.3/32 |
| **leaf.2** | 10.1.2.1/32 | 10.1.2.2/32 | 10.1.2.3/32 |
| **leaf.3** | 10.1.3.1/32 | 10.1.3.2/32 | 10.1.3.3/32 |

### Таблица подсетей для p2p линков

| **Device** | **leaf.1** | **leaf.2** | **Leaf.3** |
| --- | --- | --- | --- |
| **Spine.1** | 10.2.1.0/31 | 10.2.1.2/31 | 10.2.1.4/31 |
| **Spine.2** | 10.2.2.0/31 | 10.2.2.2/31 | 10.2.2.4/31 |

### Таблица адресов p2p на интерфейсах

| **Device** | **Порт** | **название** | **Адрес** | **Маска** |
| --- | --- | --- | --- | --- |
| **Spine.1** | Et1 | Leaf.1 | 10.2.1.0 | 255.255.255.254 |
|     | Et2 | Leaf.2 | 10.2.1.2 | 255.255.255.254 |
|     | Et3 | Leaf.3 | 10.2.1.4 | 255.255.255.254 |
| **Spine.2** | Et1 | Leaf.1 | 10.2.2.0 | 255.255.255.254 |
|     | Et2 | Leaf.2 | 10.2.2.2 | 255.255.255.254 |
|     | Et3 | Leaf.3 | 10.2.2.4 | 255.255.255.254 |
| **Leaf.1** | Et1 | Spine.1 | 10.2.1.1 | 255.255.255.254 |
|     | Et2 | Spine.2 | 10.2.2.1 | 255.255.255.254 |
| **Leaf.2** | Et1 | Spine.1 | 10.2.1.3 | 255.255.255.254 |
|     | Et2 | Spine.2 | 10.2.2.3 | 255.255.255.254 |
| **Leaf.3** | Et1 | Spine.1 | 10.2.1.5 | 255.255.255.254 |
|     | Et2 | Spine.2 | 10.2.2.5 | 255.255.255.254 |

### Соберем стенд

Зададим имена коммутаторам, назначим адреса интерфейсов и дадим описание описание.  
Что бы можно было назначить IP адрес на интерфейс их надо перевести в *routed* режим, командой `no switchport`

#### Spine.1
```
localhost(config)#hostname Spine.1
Spine.1(config)#interface ethernet 1
Spine.1(config-if-Et1)#description Leaf.1
Spine.1(config-if-Et1)#no switchport 
Spine.1(config-if-Et1)#ip address 10.2.1.0 255.255.255.254
Spine.1(config-if-Et1)#interface ethernet 2
Spine.1(config-if-Et1)#description Leaf.2
Spine.1(config-if-Et2)#no switchport 
Spine.1(config-if-Et2)#ip address 10.2.1.2 255.255.255.254
Spine.1(config-if-Et2)#interface ethernet 3
Spine.1(config-if-Et1)#description Leaf.3
Spine.1(config-if-Et3)#no switchport 
Spine.1(config-if-Et3)#ip address 10.2.1.4 255.255.255.254
```
#### Spine.2
```
localhost(config)#hostname Spine.2
Spine.2(config)#interface ethernet 1
Spine.2(config-if-Et1)#description Leaf.1
Spine.2(config-if-Et1)#no switchport 
Spine.2(config-if-Et1)#ip address 10.2.2.0 255.255.255.254
Spine.2(config-if-Et1)#interface ethernet 2
Spine.2(config-if-Et2)#description Leaf.2
Spine.2(config-if-Et2)#no switchport 
Spine.2(config-if-Et2)#ip address 10.2.2.2 255.255.255.254
Spine.2(config-if-Et2)#interface ethernet 3
Spine.2(config-if-Et3)#description Leaf.3
Spine.2(config-if-Et3)#no switchport 
Spine.2(config-if-Et3)#ip address 10.2.2.4 255.255.255.254
Spine.2(config-if-Et3)#interface 2
```
#### Leaf.1
```
localhost(config)#hostname Leaf.1
Leaf.1(config)#interface ethernet 1
Leaf.1(config-if-Et1)#description Spine.1
Leaf.1(config-if-Et1)#no switchport 
Leaf.1(config-if-Et1)#ip address 10.2.1.1 255.255.255.254
Leaf.1(config-if-Et1)#interface ethernet 2
Leaf.1(config-if-Et2)#description Spine.2
Leaf.1(config-if-Et2)#no switchport 
Leaf.1(config-if-Et2)#ip address 10.2.2.1 255.255.255.254
```
#### Leaf.2
```
localhost(config)#hostname Leaf.2
Leaf.2(config)#interface ethernet 1
Leaf.2(config-if-Et1)#description Spine.1
Leaf.2(config-if-Et1)#no switchport 
Leaf.2(config-if-Et1)#ip address 10.2.1.3 255.255.255.254
Leaf.2(config-if-Et1)#interface ethernet 2
Leaf.2(config-if-Et2)#description Spine.2
Leaf.2(config-if-Et2)#no switchport 
Leaf.2(config-if-Et2)#ip address 10.2.2.3 255.255.255.254
```
#### Leaf.3
```
localhost(config)#hostname Leaf.3
Leaf.3(config)#interface ethernet 1
Leaf.3(config-if-Et1)#description Spine.1
Leaf.3(config-if-Et1)#no switchport 
Leaf.3(config-if-Et1)#ip address 10.2.1.5 255.255.255.254
Leaf.3(config-if-Et1)#interface ethernet 2
Leaf.3(config-if-Et2)#description Spine.2
Leaf.3(config-if-Et2)#no switchport 
Leaf.3(config-if-Et2)#ip address 10.2.2.5 255.255.255.254
```
<details>
<summary>Проверим IP связанность на <b>Spine.1</b></summary>

```
Spine.1#ping 10.2.1.1
PING 10.2.1.1 (10.2.1.1) 72(100) bytes of data.
80 bytes from 10.2.1.1: icmp_seq=1 ttl=64 time=212 ms
80 bytes from 10.2.1.1: icmp_seq=2 ttl=64 time=208 ms
80 bytes from 10.2.1.1: icmp_seq=3 ttl=64 time=205 ms
80 bytes from 10.2.1.1: icmp_seq=4 ttl=64 time=201 ms
80 bytes from 10.2.1.1: icmp_seq=5 ttl=64 time=241 ms

--- 10.2.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 201.504/214.037/241.647/14.280 ms, pipe 5, ipg/ewma 12.478/214.052 ms
Spine.1#ping 10.2.1.3
PING 10.2.1.3 (10.2.1.3) 72(100) bytes of data.
80 bytes from 10.2.1.3: icmp_seq=1 ttl=64 time=145 ms
80 bytes from 10.2.1.3: icmp_seq=2 ttl=64 time=137 ms
80 bytes from 10.2.1.3: icmp_seq=3 ttl=64 time=132 ms
80 bytes from 10.2.1.3: icmp_seq=4 ttl=64 time=127 ms
80 bytes from 10.2.1.3: icmp_seq=5 ttl=64 time=136 ms

--- 10.2.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 127.591/136.001/145.087/5.761 ms, pipe 5, ipg/ewma 11.316/140.352 ms
Spine.1#ping 10.2.1.5
PING 10.2.1.5 (10.2.1.5) 72(100) bytes of data.
80 bytes from 10.2.1.5: icmp_seq=1 ttl=64 time=68.2 ms
80 bytes from 10.2.1.5: icmp_seq=2 ttl=64 time=63.6 ms
80 bytes from 10.2.1.5: icmp_seq=3 ttl=64 time=59.7 ms
80 bytes from 10.2.1.5: icmp_seq=4 ttl=64 time=56.0 ms
80 bytes from 10.2.1.5: icmp_seq=5 ttl=64 time=54.3 ms

--- 10.2.1.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 54.355/60.401/68.240/5.074 ms, pipe 5, ipg/ewma 11.148/63.969 ms

```
</details>

<details>
<summary>Проверим IP связанность на <b>Spine.2</b></summary>

```
Spine.2#ping 10.2.2.1
PING 10.2.2.1 (10.2.2.1) 72(100) bytes of data.
80 bytes from 10.2.2.1: icmp_seq=1 ttl=64 time=148 ms
80 bytes from 10.2.2.1: icmp_seq=2 ttl=64 time=130 ms
80 bytes from 10.2.2.1: icmp_seq=3 ttl=64 time=121 ms
80 bytes from 10.2.2.1: icmp_seq=4 ttl=64 time=114 ms
80 bytes from 10.2.2.1: icmp_seq=5 ttl=64 time=108 ms

--- 10.2.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 73ms
rtt min/avg/max/mdev = 108.943/124.640/148.143/13.705 ms, pipe 5, ipg/ewma 18.289/135.506 ms
Spine.2#ping 10.2.2.3
PING 10.2.2.3 (10.2.2.3) 72(100) bytes of data.
80 bytes from 10.2.2.3: icmp_seq=1 ttl=64 time=230 ms
80 bytes from 10.2.2.3: icmp_seq=2 ttl=64 time=223 ms
80 bytes from 10.2.2.3: icmp_seq=3 ttl=64 time=222 ms
80 bytes from 10.2.2.3: icmp_seq=4 ttl=64 time=214 ms
80 bytes from 10.2.2.3: icmp_seq=5 ttl=64 time=207 ms

--- 10.2.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 207.966/219.808/230.596/7.774 ms, pipe 5, ipg/ewma 11.165/224.641 ms
Spine.2#ping 10.2.2.5
PING 10.2.2.5 (10.2.2.5) 72(100) bytes of data.
80 bytes from 10.2.2.5: icmp_seq=1 ttl=64 time=151 ms
80 bytes from 10.2.2.5: icmp_seq=2 ttl=64 time=141 ms
80 bytes from 10.2.2.5: icmp_seq=3 ttl=64 time=136 ms
80 bytes from 10.2.2.5: icmp_seq=4 ttl=64 time=131 ms
80 bytes from 10.2.2.5: icmp_seq=5 ttl=64 time=125 ms

--- 10.2.2.5 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 125.886/137.380/151.403/8.698 ms, pipe 5, ipg/ewma 12.455/143.796 ms

```
</details>