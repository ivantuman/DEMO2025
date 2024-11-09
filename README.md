# DEMO2025

# Задание 1.

**Топология сети**

![image](https://github.com/user-attachments/assets/9abe5702-ce9a-49d7-a528-b0c3d0721308)

**Таблица устройств:**

|Машина          |RAM, Гб       | CPU  |HDD/SSD, Гб     |Шаблон                 |
|  ------------- | ------------ | -----| -------------  |  ---------------------|
|ISP             |1             |1     |10              |Vesr       |
|HQ-RTR          |1             |1     |10              |EcoRouter              |                 
|BR-RTR          |1             |1     |10              |Vesr              |                
|HQ-SRV          |2             |1     |10              |ALT Linux Server       |                
|BR-SRV          |2             |1     |10              |ALT Linux Server       |                
|HQ-CLI          |3             |2     |15              |ALT Linux Workstation  |                
|Итого           |10            |7     |65              |-                      |        


**Таблица IP-адресов**
| Имя устройства | Интерфейс | IP          | Маска           | Шлюз        |
| -------------- | --------- | ----------  | --------------- | ----------- |
| ISP            | gi1/0/1   | DHCP        |                 |             |
|                | gi1/0/2   | 172.16.5.1  | 255.255.255.240 |             |
|                | gi1/0/3   | 172.16.4.1  | 255.255.255.240 |             |
| HQ-RTR         | ge0       | 172.16.4.2  | 255.255.255.240 | 172.16.4.1  |      
|                | ge1.100   | 192.168.0.1 | 255.255.255.192 |             |      
|                | ge1.200   | 192.168.0.65| 255.255.255.240 |             |      
|                | ge1.999   | 192.168.0.81| 255.255.255.248 |             |      
|                | tunnel.1  | 172.16.1.1  | 255.255.255.252 |             |      
| BR-RTR         | gi1/0/1   | 172.16.5.2  | 255.255.255.240 | 172.16.5.1  |      
|                | gi1/0/2   | 192.168.1.1 | 255.255.255.224 |             |
|                | gre1      | 172.16.1.2  | 255.255.255.252 |             |
| HQ-SRV         | ens192    | 192.168.0.2 | 255.255.255.192 | 192.168.0.1 |      
| HQ-CLI         | ens192    | DHCP        | 255.255.255.240 | 192.168.0.65|      
| BR-SRV         | ens192    | 192.168.1.2 | 255.255.255.224 | 192.168.1.1 |  

# 1. Назначаем IP-адреса
## ISP
```
configure

int gi1/0/3
ip address 172.16.4.1/28
ip firewall disable
no shutdown

int gi1/0/2
ip address 172.16.5.1/28
ip firewall disable
no shutdown

commit
confirm
```
## BR-RTR

```
configure

interface gigabitethernet 1/0/1
  ip firewall disable
  ip address 172.16.5.2/28
  no shutdown
exit
interface gigabitethernet 1/0/2
  ip firewall disable
  ip address 192.168.1.1/27
  no shutdown
exit
tunnel gre 1
  ttl 16
  mtu 1400
  ip firewall disable
  local address 172.16.5.2
  remote address 172.16.4.2
  ip address 172.16.1.2/30
  enable
exit
ip route 0.0.0.0/0 172.16.5.1

commit
confirm
```
## BR-SRV

```
echo "TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes" > /etc/net/ifaces/ens192/options
```

```
echo 192.168.1.2/26 > /etc/net/ifaces/ens192/ipv4address
```

```
echo default via 192.168.1.1 > /etc/net/ifaces/ens192/ipv4route
```

```
echo nameserver 192.168.0.2 > /etc/net/ifaces/ens192/resolv.conf
```

```
systemctl restart network
```

```
ip address
```
## HQ-SRV

```
echo "TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes" > /etc/net/ifaces/ens192/options
```

```
echo 192.168.0.2/26 > /etc/net/ifaces/ens192/ipv4address
```

```
echo default via 192.168.0.1 > /etc/net/ifaces/ens192/ipv4route
```

```
systemctl restart network
```
## HQ-RTR - EcoRouter


```
int TO-ISP
ip address 172.16.4.2/28
no shutdown
```
```
port ge0
service-instance SI-ISP
encapsulation untagged
connect ip interface TO-ISP
```
```
interface HQ-SRV
 ip mtu 1500
 ip address 192.168.0.1/26
!
interface HQ-CLI
 ip mtu 1500
 ip address 192.168.0.65/28
!
interface HQ-MGMT
 ip mtu 1500
 ip address 192.168.0.81/29
!
```
```
port ge1
 mtu 9234
 service-instance ge1/vlan100
  encapsulation dot1q 100
  rewrite pop 1
  connect ip interface HQ-SRV
 service-instance ge1/vlan200
  encapsulation dot1q 200
  rewrite pop 1
  connect ip interface HQ-CLI
 service-instance ge1/vlan999
  encapsulation dot1q 999
  rewrite pop 1
  connect ip interface HQ-MGMT
```

Создаем GRE туннель

```
interface tunnel.1
 ip mtu 1400
 ip address 172.16.1.1/30
 ip tunnel 172.16.4.2 172.16.5.2 mode gre
```

Задаем маршрут по умолчанию в сторону ISP

```
ip route 0.0.0.0/0 172.16.4.1
```
## HQ-CLI получает IP-адрес по DHCP

# 2. Настройка OSPF

## HQ-RTR

```
router ospf 1
 network 172.16.1.0 0.0.0.3 area 0.0.0.0
 network 192.168.0.0 0.0.0.255 area 0.0.0.0
!
interface tunnel.1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Demo2025
 ip ospf network point-to-point
```
## Проверка

```
sh ip ospf neighbor 
sh ip ospf interface brief
sh ip route
```
![image](https://github.com/user-attachments/assets/ba4ecfd8-acbc-4275-9823-cbd55def9f48)
![image](https://github.com/user-attachments/assets/3e6be689-d63b-42c1-8262-0120926d8b06)
![image](https://github.com/user-attachments/assets/64490d8f-fbd9-44ab-8f2c-e74934d5e26d)

## BR-RTR

```
key-chain ospfkey
  key 1
    key-string ascii-text Demo2025
  exit
exit

router ospf 1
  area 0.0.0.0
    enable
  exit
  enable
exit

interface gigabitethernet 1/0/2
  ip ospf instance 1
  ip ospf
exit
tunnel gre 1
  ip ospf instance 1
  ip ospf network point-to-point
  ip ospf authentication key-chain ospfkey
  ip ospf authentication algorithm md5
  ip ospf
exit

commit
confirm
```
## Проверка

```
sh ip ospf neighbor
sh ip route
```

```
sh ip ospf interface
```
![image](https://github.com/user-attachments/assets/2c009f34-c896-4b44-9f7d-e2e7f8ed0199)
![image](https://github.com/user-attachments/assets/020e4e5b-6cf1-43c2-b06f-97746a7e16ed)
![image](https://github.com/user-attachments/assets/a62a82ab-28c4-4812-9234-742a7461a3d1)

# 3. Настройка DHCP на HQ-RTR

```
ip pool HQ-NET200 1
 range 192.168.0.66-192.168.0.70
!
dhcp-server 1
 lease 86400
 mask 255.255.255.0
 pool HQ-NET200 1
  dns 192.168.0.2
  domain-name au-team.irpo
  gateway 192.168.0.65
  mask 255.255.255.240
!
interface HQ-CLI
 dhcp-server 1
!
```
## Проверка
![image](https://github.com/user-attachments/assets/760212ce-63dc-4575-96da-998f2bc805d6)

# 4. Настройка NAT

## ISP

```
object-group network LOCAL_NET
  ip address-range 172.16.5.1-172.16.5.2
  ip address-range 172.16.4.1-172.16.4.2
exit

nat source
  ruleset SNAT
    to interface gigabitethernet 1/0/1
    rule 1
      match source-address LOCAL_NET
      action source-nat interface
      enable
    exit
  exit
exit
```
## HQ-RTR

```
interface TO-ISP
 ip nat outside
!
interface HQ-SRV
 ip nat inside
!
interface HQ-CLI
 ip nat inside
!
interface HQ-MGMT
 ip nat inside
!
ip nat pool NAT_POOL 192.168.0.1-192.168.0.254
!
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface TO-ISP
```

## BR-RTR

```
object-group network LOCAL_NET
  ip address-range 192.168.1.1-192.168.1.30
exit

nat source
  ruleset SNAT
    to interface gigabitethernet 1/0/1
    rule 1
      match source-address LOCAL_NET
      action source-nat interface
      enable
    exit
  exit
exit
```

## Проверка

![image](https://github.com/user-attachments/assets/97f07183-4403-4b17-8f68-2402e0ee85e3)
![image](https://github.com/user-attachments/assets/ee2460a2-8891-4c5a-bb3d-c6b883ddc615)

# 5. Настройка пользователей
## HQ-SRV, BR-SRV
```
useradd -m -u 1010 sshuser
passwd sshuser
```

```
nano /etc/sudoers.d/sshuser
```

```
sshuser ALL=(ALL) NOPASSWD:ALL
```

### Проверка:

```
su - sshuser
sudo whoami
```
## HQ-RTR (EcoRouter)

```
conf t
username net_admin
password P@ssw0rd
role admin
activate
```

## BR-RTR (Eltex - vESR)

```
configure
username net_admin
password P@ssw0rd
privilege 15
end
!
commit
confirm
!
```
# 6. Настройка DNS

## HQ-SRV
Устанавливаем bind:

```
apt-get install bind bind-utils
```

Редактируем конфиг:

```
nano  /var/lib/bind/etc/options.conf
```
Меняем следующее
# 1.
![image](https://github.com/user-attachments/assets/913868cf-8e4c-44cd-899e-e9b0e8bd4a5e)

![image](https://github.com/user-attachments/assets/d2557ec8-ce5f-4433-9ee2-af2b7ba7afe9)

![image](https://github.com/user-attachments/assets/4febb05a-f431-47ac-a33e-27631df09699)

![image](https://github.com/user-attachments/assets/ebe53426-1787-40ee-8200-6ac8162f0943)

# 2.
![image](https://github.com/user-attachments/assets/07dd0e06-6fc2-410c-b511-4d871469eaaa)

![image](https://github.com/user-attachments/assets/9c40c1c1-74d9-4f96-9d1a-43c9cd36e60a)

![image](https://github.com/user-attachments/assets/924ae861-5548-4e93-b978-189f5eea34c0)

Проверяем на ошибки

```
named-checkconf
```
Если появилась ошибка, нужно скофигураровать ключи с помощью `rndc-confgen`. Редактируем файл `var/lib/bind/etc/rndc.key `
```
nano /var/lib/bind/etc/rndc.key 
```

Встравляем в него ключ, который получили при помощи `rndc-confgen` и проверяем на ошибки. 
```
named-checkconf
```
Если ошибок нет, то запускаем `bind`

```
systemctl enable --now bind
```
Проверяем что `bind` работает

```
systemctl status bind
```
Редактируем `resolv.conf`

```
nano /etc/net/ifaces/ens192/resolv.conf 
```

```
search au-team.irpo
nameserver 127.0.0.1
nameserver 192.168.0.2
nameserver 77.88.8.8
```

Перезагружаем сеть

```
systemctl restart network
```

Проверяем

```
dig ya.ru
```

### Создаем зону прямого просмотра

```
nano  /var/lib/bind/etc/local.conf
```

```
zone "au-team.irpo" {
        type master;
        file "au-team.irpo.db";
};
```
![image](https://github.com/user-attachments/assets/cc43f220-d708-4275-a7d4-318ba1c60649)

Создаем копию файла-шаблона прямой зоны
```
# cp /var/lib/bind/etc/zone/localdomain  /var/lib/bind/etc/zone/au-team.irpo.db
```
Задаём права на файл
```
chown named. /var/lib/bind/etc/zone/au-team.irpo.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo.db
```
Открываем файл и прописываем в нём следующее:
```
nano /var/lib/bind/etc/zone/au-team.irpo.db
```
```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
        IN      A       192.168.0.2
hq-rtr  IN      A       192.168.0.1
br-rtr  IN      A       192.168.1.1
hq-srv  IN      A       192.168.0.2
hq-cli  IN      A       192.168.0.66
br-srv  IN      A       192.168.1.2
moodle  IN      CNAME   hq-rtr
wiki    IN      CNAME   hq-rtr
```
Проверяем правильность настройки зоны и перезагружаем bind
```
named-checkconf -z
```
![image](https://github.com/user-attachments/assets/4fa4189a-6144-4f83-bb44-51a46407ecde)
```
systemctl restart bind
```
И делаем проверку
```
dig hq-srv.au-team.irpo
```
![image](https://github.com/user-attachments/assets/0ad7ba95-0fc8-4cce-82a8-4c217ff5d532)

### Создаем зону обратного просмотра
Заходим в конфигурационный файл `/var/lib/bind/etc/local.conf` и вписываем следующее:

```
zone "0.168.192.in-addr.arpa" {
        type master;
        file "au-team.irpo_rev.db";
};
```
![image](https://github.com/user-attachments/assets/b1c37d3f-9248-4c44-a7f8-928315f3d014)

Копируем шаблон файла и задаём права 
```
cp /var/lib/bind/etc/zone/{127.in-addr.arpa,au-team.irpo_rev.db}
```
```
chown named. /var/lib/bind/etc/zone/au-team.irpo_rev.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo_rev.db
```
Открываем `/var/lib/bind/etc/zone/au-team.irpo_rev.db` и вписываем в него следующее
```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
66      IN      PTR     hq-cli.au-team.irpo.
```
Делаем проверку
```
named-checkconf -z
```
![image](https://github.com/user-attachments/assets/18ff2d5c-31bc-4037-8110-bc58fde5a899)

Перезагружаем bind и проверяем
```
dig -x 192.168.0.2
```
![image](https://github.com/user-attachments/assets/cd06fc93-9009-4fa6-b0f2-18f382b6fb09)
## Производим полную проверку с HQ-CLI
![image](https://github.com/user-attachments/assets/f6c75cd5-786d-4b6a-84b0-c66631bc3895)

# Настройка часовых поясов 
На HQ-SRV, HQ-CLI, BR-SRV проверяем часовой пояс
```
timedatectl status
```
И меняем 
```
timedatectl set-timezone Asia/Yekaterinburg
```
На HQ-RTR 
```
conf t
ntp timezone utc+5
show ntp timezone
```
На BR-RTR
```
configure
clock timezone gmt +5
end
commit
confirm
```
И проверяем
```
show date
```
