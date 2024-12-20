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

Настройку начнём с Leaf.1
- Включим процесс BGP и зададим AS `#router bgp 65101`
- Для упрощения конфигурации у Arista есть возможность создавать _peer group_
  - Создадим _peer group Spines_ и укажем нащих соседей. Соседство будем устанавливать на p2p линках.

  ```
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
