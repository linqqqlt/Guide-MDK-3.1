# ЭКЗАМЕН МДК 03.01 — ПОЛНОЕ ПОШАГОВОЕ РЕШЕНИЕ

**«Эксплуатация объектов сетевой инфраструктуры»**
**Специальность 09.02.06 «Сетевое и системное администрирование»**

---

## ТОПОЛОГИЯ СЕТИ (Рисунок 1)

```
      Left                                              Right
 ┌────────┐  ┌────────┐          ┌────────┐        ┌────────┐
 │ WEB-L  │  │  SRV   │          │  CLI   │        │ WEB-R  │
 │.100/24 │  │.200/24 │          │3.3.3.10│        │.100/24 │
 └───┬────┘  └───┬────┘          └───┬────┘        └───┬────┘
     │           │                   │                 │
     │  192.168.200.0/24             │ 3.3.3.0/24      │ 172.16.100.0/24
     └─────┬─────┘                   │                 │
     ┌─────┴─────┐            ┌──────┴─────┐     ┌────┴─────┐
     │   RTR-L   │            │    ISP     │     │  RTR-R   │
     │   .254/24 │            │  4.4.4.1   │     │  .254/24 │
     │4.4.4.100  ├────────────┤  5.5.5.1   ├─────┤5.5.5.100 │
     └───────────┘ 4.4.4.0/24 │  3.3.3.1   │     └──────────┘
                              └──────┬─────┘ 5.5.5.0/24
                                     │
                               ┌─────┴─────┐
                               │ Internet  │
                               └───────────┘
```

## ТАБЛИЦА АДРЕСАЦИИ

| Устройство | ОС | IP-адреса |
|---|---|---|
| ISP | Alt-Server | ens33: NAT, ens37: 4.4.4.1/24, ens38: 3.3.3.1/24, ens39: 5.5.5.1/24 |
| RTR-L | EcoRouter | ge0: 4.4.4.100/24 (к ISP), ge1: 192.168.200.254/24 (LAN) |
| RTR-R | EcoRouter | ge0: 5.5.5.100/24 (к ISP), ge1: 172.16.100.254/24 (LAN) |
| SRV | Alt-Server | 192.168.200.200/24, шлюз 192.168.200.254 |
| WEB-L | Alt-Server | 192.168.200.100/24, шлюз 192.168.200.254 |
| WEB-R | Alt-Server | 172.16.100.100/24, шлюз 172.16.100.254 |
| CLI | Alt-Workstation | 3.3.3.10/24, шлюз 3.3.3.1 |

## ТАБЛИЦА DNS-ЗАПИСЕЙ

**Зона ANDREW.ZLOY (на ISP):**

| Тип | Ключ | Значение |
|---|---|---|
| A | ISP | 3.3.3.1 |
| A | www | 4.4.4.100 |
| A | www | 5.5.5.100 |
| CNAME | Internet | ISP |
| NS | MDK | делегирование на SRV (через 4.4.4.100) |

**Зона MDK.ANDREW.ZLOY (на SRV):**

| Тип | Ключ | Значение |
|---|---|---|
| A | Web-l | 192.168.200.100 |
| A | Web-r | 172.16.100.100 |
| A | SRV | 192.168.200.200 |
| A | rtr-l | 192.168.200.254 |
| A | rtr-r | 172.16.100.254 |
| CNAME | ntp | ISP |
| CNAME | dns | SRV |

---

## ВАЖНЫЕ ЗАМЕЧАНИЯ (ГРАБЛИ)

1. **EcoRouter не видит первый адаптер** — нумерация сдвинута на 1. Если добавлено 3 адаптера, видно только ge0 и ge1. Решение: добавлять на 1 адаптер больше, чем нужно.
2. **NetworkManager конфликтует с etcnet** — на Alt Linux нужно отключить NM или поставить `NM_CONTROLLED=no` в options.
3. **Опечатки в IP** — внимательно проверяй! Ошибка `172.168` вместо `172.16` ломает связность.
4. **rndc.key содержит лишние блоки** — после `rndc-confgen` нужно удалить всё кроме `key "rndc-key" {...}`.

---

## ЗАДАНИЕ 1 — ИМЕНА УСТРОЙСТВ

### Alt Linux (ISP, SRV, WEB-L, WEB-R, CLI):

```bash
# ISP
hostnamectl set-hostname isp.andrew.zloy

# SRV
hostnamectl set-hostname srv-l.mdk.andrew.zloy

# WEB-L
hostnamectl set-hostname web-l.mdk.andrew.zloy

# WEB-R
hostnamectl set-hostname web-r.mdk.andrew.zloy

# CLI
hostnamectl set-hostname cli.andrew.zloy
```

### EcoRouter (RTR-L, RTR-R):

```
en
conf t
hostname rtr-l.andrew.zloy

hostname rtr-r.andrew.zloy
```

---

## БАЗОВАЯ НАСТРОЙКА СЕТИ — ISP

### Настройка интерфейсов:

```bash
# ens37 — к RTR-L (4.4.4.1/24)
mkdir -p /etc/net/ifaces/ens37
cat > /etc/net/ifaces/ens37/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "4.4.4.1/24" > /etc/net/ifaces/ens37/ipv4address

# ens38 — к CLI (3.3.3.1/24)
mkdir -p /etc/net/ifaces/ens38
cat > /etc/net/ifaces/ens38/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "3.3.3.1/24" > /etc/net/ifaces/ens38/ipv4address

# ens39 — к RTR-R (5.5.5.1/24)
mkdir -p /etc/net/ifaces/ens39
cat > /etc/net/ifaces/ens39/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "5.5.5.1/24" > /etc/net/ifaces/ens39/ipv4address
```

### IP Forwarding и NAT:

```bash
# Включить маршрутизацию
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -p /etc/net/sysctl.conf

# NAT через NAT-адаптер (ens33)
iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Перезапуск сети и проверка
service network restart
ip a
```

---

## БАЗОВАЯ НАСТРОЙКА СЕТИ — RTR-L (EcoRouter)

```
en
conf t

port ge0
  service-instance to-isp
    encapsulation untagged
    exit

port ge1
  service-instance to-lan
    encapsulation untagged
    exit

interface to-isp
  ip mtu 1500
  connect port ge0 service-instance to-isp
  ip address 4.4.4.100/24
  exit

interface to-lan
  ip mtu 1500
  connect port ge1 service-instance to-lan
  ip address 192.168.200.254/24
  exit

ip route 0.0.0.0/0 4.4.4.1
```

**Проверка:**
```
do show ip route
do ping 4.4.4.1
```

---

## БАЗОВАЯ НАСТРОЙКА СЕТИ — RTR-R (EcoRouter)

```
en
conf t

port ge0
  service-instance to-isp
    encapsulation untagged
    exit

port ge1
  service-instance to-lan
    encapsulation untagged
    exit

interface to-isp
  ip mtu 1500
  connect port ge0 service-instance to-isp
  ip address 5.5.5.100/24
  exit

interface to-lan
  ip mtu 1500
  connect port ge1 service-instance to-lan
  ip address 172.16.100.254/24
  exit

ip route 0.0.0.0/0 5.5.5.1
```

**Проверка:**
```
do show ip route
do ping 5.5.5.1
```

---

## БАЗОВАЯ НАСТРОЙКА СЕТИ — SRV, WEB-L, WEB-R, CLI

### SRV (192.168.200.200/24):

```bash
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << 'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
ONBOOT=yes
NM_CONTROLLED=no
DISABLED=no
EOF
echo "192.168.200.200/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 192.168.200.254" > /etc/net/ifaces/ens33/ipv4route
service network restart
ping 192.168.200.254
```

### WEB-L (192.168.200.100/24):

```bash
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << 'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
ONBOOT=yes
NM_CONTROLLED=no
DISABLED=no
EOF
echo "192.168.200.100/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 192.168.200.254" > /etc/net/ifaces/ens33/ipv4route
service network restart
ping 192.168.200.254
```

### WEB-R (172.16.100.100/24):

```bash
mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << 'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
ONBOOT=yes
NM_CONTROLLED=no
DISABLED=no
EOF
echo "172.16.100.100/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 172.16.100.254" > /etc/net/ifaces/ens33/ipv4route
service network restart
ping 172.16.100.254
```

### CLI (3.3.3.10/24):

**ВАЖНО:** На CLI обязательно отключить NetworkManager!

```bash
systemctl stop NetworkManager
systemctl disable NetworkManager

mkdir -p /etc/net/ifaces/ens33
cat > /etc/net/ifaces/ens33/options << 'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
ONBOOT=yes
NM_CONTROLLED=no
DISABLED=no
EOF
echo "3.3.3.10/24" > /etc/net/ifaces/ens33/ipv4address
echo "default via 3.3.3.1" > /etc/net/ifaces/ens33/ipv4route
service network restart
ping 3.3.3.1
```

---

## ЗАДАНИЕ 2 — ПРОБРОС SSH (NAT)

### RTR-L (внешний порт 1984 → WEB-L:22):

```
conf t

ip nat source static tcp 192.168.200.100 22 4.4.4.100 1984

ip nat pool NAT 192.168.200.1-192.168.200.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
  ip nat outside
  exit

interface to-lan
  ip nat inside
  exit
```

### RTR-R (внешний порт 1945 → WEB-R:22):

```
conf t

ip nat source static tcp 172.16.100.100 22 5.5.5.100 1945

ip nat pool NAT 172.16.100.1-172.16.100.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
  ip nat outside
  exit

interface to-lan
  ip nat inside
  exit
```

### Разрешить вход root по SSH на WEB-L и WEB-R:

```bash
# На WEB-L и WEB-R:
vim /etc/openssh/sshd_config
# Найти и изменить:
# PermitRootLogin yes
systemctl restart sshd
```

Если забыл пароль root:
```bash
passwd root
```

### Проверка (с CLI):

```bash
ssh -p 1984 root@4.4.4.100
ssh -p 1945 root@5.5.5.100
```

---

## ЗАДАНИЕ 3 — GRE ТУННЕЛЬ

### RTR-L:

```
conf t
interface tunnel.1
  ip address 10.0.0.1/30
  ip tunnel 4.4.4.100 5.5.5.100 mode gre
  exit
```

### RTR-R:

```
conf t
interface tunnel.1
  ip address 10.0.0.2/30
  ip tunnel 5.5.5.100 4.4.4.100 mode gre
  exit
```

### Проверка:

```
# С RTR-L:
do ping 10.0.0.2

# С RTR-R:
do ping 10.0.0.1
```

---

## ЗАДАНИЕ 4 — OSPF С АУТЕНТИФИКАЦИЕЙ

### RTR-L:

```
conf t
router ospf 1
  network 192.168.200.0/24 area 0
  network 10.0.0.0/30 area 0
  exit

interface tunnel.1
  ip ospf authentication
  ip ospf authentication-key P@ssw0rd
  exit
```

### RTR-R:

```
conf t
router ospf 1
  network 172.16.100.0/24 area 0
  network 10.0.0.0/30 area 0
  exit

interface tunnel.1
  ip ospf authentication
  ip ospf authentication-key P@ssw0rd
  exit
```

### Проверка:

```
do show ip ospf neighbor
```

Должен появиться сосед. После этого WEB-R может пинговать SRV (192.168.200.200) через туннель:

```bash
# С WEB-R:
ping 192.168.200.200

# С WEB-L:
ping 172.16.100.100
```

---

## ЗАДАНИЕ 5 — DNS ПЕРВОГО УРОВНЯ НА ISP (зона ANDREW.ZLOY)

### Установка BIND:

```bash
apt-get update
apt-get install bind bind-utils -y
```

### Настройка options.conf:

```bash
vim /etc/bind/options.conf
```

Внутри блока `options { ... }` добавить/изменить:
```
listen-on { any; };
allow-query { any; };
allow-recursion { any; };
forwarders { 77.88.8.8; };
```

### Добавить зону в rfc1912.conf:

```bash
cat >> /etc/bind/rfc1912.conf << 'EOF'

zone "ANDREW.ZLOY" {
    type master;
    file "ANDREW.ZLOY";
    allow-update { none; };
};
EOF
```

### Создать файл зоны:

```bash
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/ANDREW.ZLOY
cat > /var/lib/bind/etc/zone/ANDREW.ZLOY << 'EOF'
$TTL 86400
@       IN  SOA ANDREW.ZLOY. root.ANDREW.ZLOY. (
            2025010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   isp.ANDREW.ZLOY.
isp     IN  A    3.3.3.1
www     IN  A    4.4.4.100
www     IN  A    5.5.5.100
Internet IN CNAME isp.ANDREW.ZLOY.
MDK     IN  NS   srv.MDK.ANDREW.ZLOY.
srv.MDK IN  A    4.4.4.100
EOF
```

### Проброс DNS на RTR-L (SRV во внутренней сети):

```
# На RTR-L:
conf t
ip nat source static tcp 192.168.200.200 53 4.4.4.100 53
ip nat source static udp 192.168.200.200 53 4.4.4.100 53
```

### Генерация rndc.key и запуск:

```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
```

### Проверка и запуск:

```bash
named-checkconf /etc/bind/named.conf
named-checkzone ANDREW.ZLOY /var/lib/bind/etc/zone/ANDREW.ZLOY
systemctl enable --now bind
systemctl restart bind
systemctl status bind
```

### Если ошибка "'options' redefined":

```bash
vim /var/lib/bind/etc/rndc.key
```

Оставить ТОЛЬКО:
```
key "rndc-key" {
    algorithm hmac-sha256;
    secret "сгенерированный_ключ";
};
```

Удалить все блоки `options`, `server` из этого файла.

### Настройка DNS на CLI:

```bash
echo "nameserver 3.3.3.1" > /etc/resolv.conf
```

---

## ЗАДАНИЕ 6 — DNS ВТОРОГО УРОВНЯ НА SRV (зона MDK.ANDREW.ZLOY)

### Установка BIND:

```bash
apt-get update
apt-get install bind bind-utils -y
```

### Добавить зону:

```bash
vim /etc/bind/rfc1912.conf
```

Добавить в конец:
```
zone "MDK.ANDREW.ZLOY" {
    type master;
    file "MDK.ANDREW.ZLOY";
    allow-update { none; };
};
```

### Создать файл зоны:

```bash
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/MDK.ANDREW.ZLOY
cat > /var/lib/bind/etc/zone/MDK.ANDREW.ZLOY << 'EOF'
$TTL 86400
@       IN  SOA MDK.ANDREW.ZLOY. root.MDK.ANDREW.ZLOY. (
            2025010101  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400       ; Minimum TTL
)
        IN  NS   srv.MDK.ANDREW.ZLOY.
srv     IN  A    192.168.200.200
Web-l   IN  A    192.168.200.100
Web-r   IN  A    172.16.100.100
rtr-l   IN  A    192.168.200.254
rtr-r   IN  A    172.16.100.254
ntp     IN  CNAME isp.ANDREW.ZLOY.
dns     IN  CNAME srv.MDK.ANDREW.ZLOY.
EOF
```

### Настройка options.conf на SRV:

```bash
vim /etc/bind/options.conf
```

Внутри `options { ... }`:
```
listen-on { any; };
allow-query { any; };
allow-recursion { any; };
```

### Запуск:

```bash
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
systemctl enable --now bind
systemctl restart bind
```

### Настройка resolv.conf на SRV:

```bash
echo "nameserver 192.168.200.200" > /etc/resolv.conf
```

### Проверка (с CLI):

```bash
nslookup isp.ANDREW.ZLOY 3.3.3.1
nslookup Web-l.MDK.ANDREW.ZLOY 3.3.3.1
```

---

## ЗАДАНИЕ 7 — NTP (СИНХРОНИЗАЦИЯ ВРЕМЕНИ)

### ISP — NTP-сервер (chrony, stratum=3):

```bash
apt-get update
apt-get install chrony -y

cat > /etc/chrony.conf << 'EOF'
local stratum 3
allow 0.0.0.0/0
driftfile /var/lib/chrony/drift
EOF

systemctl enable --now chronyd
systemctl restart chronyd
```

### Все клиенты (SRV, WEB-L, WEB-R, CLI) — NTP-клиенты:

```bash
apt-get install chrony -y

cat > /etc/chrony.conf << 'EOF'
server 3.3.3.1 iburst
driftfile /var/lib/chrony/drift
EOF

systemctl enable --now chronyd
systemctl restart chronyd
```

**Примечание:** На SRV и WEB-L сервер NTP через шлюз RTR-L (4.4.4.1 недоступен напрямую), поэтому можно указать IP ISP в их сети, но проще оставить `3.3.3.1` — пакеты пойдут через NAT на RTR-L.

### На роутерах EcoRouter:

```
# RTR-L:
ntp timezone utc+3
ntp server 4.4.4.1

# RTR-R:
ntp timezone utc+3
ntp server 5.5.5.1
```

### Проверка (на клиентах):

```bash
chronyc sources
```

---

## ЗАДАНИЕ 8 — ЯНДЕКС.БРАУЗЕР НА CLI

```bash
apt-get update
apt-get install yandex-browser-stable -y
```

Если пакета нет в репозитории — нужен доступ к интернету через ISP NAT, либо установка из RPM.

---

## ЗАДАНИЕ 9 — SMB-СЕРВЕР НА SRV С RAID 1

### Шаг 1: Добавить 2 диска в VMware

В настройках VM SRV: Add → Hard Disk → SCSI → Create new → 1 GB. Повторить для второго диска.

### Шаг 2: Определить диски

```bash
lsblk
```

**ВНИМАНИЕ:** Системный диск (30 ГБ) — НЕ ТРОГАТЬ! Использовать только новые диски по 1 ГБ.

Пример вывода:
```
sda   30G   ← СИСТЕМНЫЙ, НЕ ТРОГАЙ
sdb    1G   ← новый диск ✓
sdc    1G   ← новый диск ✓
```

### Шаг 3: Создать RAID 1 (зеркало)

```bash
mdadm --create /dev/md0 -l 1 -n 2 /dev/sdb /dev/sdc
# На вопрос про bitmap ответить: y
```

### Шаг 4: Форматирование и монтирование

```bash
mkfs --type ext4 /dev/md0
mkdir -p /mnt/storage
echo "/dev/md0  /mnt/storage  ext4  defaults  0  0" >> /etc/fstab
mount -a
lsblk
```

### Шаг 5: Установка и настройка Samba

```bash
# Если нет интернета — временно сменить DNS:
echo "nameserver 77.88.8.8" > /etc/resolv.conf

apt-get update
apt-get install samba -y

# Создать директорию для шары
mkdir -p /mnt/storage/share
chmod 777 /mnt/storage/share
```

### Шаг 6: Конфигурация Samba

```bash
cat > /etc/samba/smb.conf << 'EOF'
[global]
workgroup = WORKGROUP
server string = SRV
map to guest = Bad User

[share]
path = /mnt/storage/share
browseable = yes
writable = yes
guest ok = yes
create mask = 0777
directory mask = 0777
EOF
```

### Шаг 7: Запуск

```bash
systemctl enable --now smb
systemctl restart smb
systemctl status smb
```

---

## ЗАДАНИЕ 10 — CIFS МОНТИРОВАНИЕ НА WEB-L И WEB-R

### WEB-L:

```bash
apt-get install cifs-utils -y
mkdir -p /opt/share
mount -t cifs //192.168.200.200/share /opt/share -o guest
echo "//192.168.200.200/share  /opt/share  cifs  guest  0  0" >> /etc/fstab
```

### WEB-R:

**ВАЖНО:** Для WEB-R нужен работающий GRE+OSPF (задания 3-4), иначе WEB-R не видит SRV (192.168.200.200)!

```bash
apt-get install cifs-utils -y
mkdir -p /opt/share
mount -t cifs //192.168.200.200/share /opt/share -o guest
echo "//192.168.200.200/share  /opt/share  cifs  guest  0  0" >> /etc/fstab
```

### Проверка:

```bash
# На WEB-L — создать файл:
touch /opt/share/test_from_webl

# На WEB-R — проверить что файл виден:
ls /opt/share/
# Должен быть виден test_from_webl
```

---

## ПОРЯДОК ПРОВЕРКИ ПОСЛЕ ПЕРЕЗАГРУЗКИ

1. `ip a` на всех машинах — все интерфейсы UP с правильными IP
2. `ping 4.4.4.1` с RTR-L, `ping 5.5.5.1` с RTR-R — связь с ISP
3. `ping 3.3.3.1` с CLI — связь CLI с ISP
4. `ping 10.0.0.2` с RTR-L — GRE туннель работает
5. `do show ip ospf neighbor` на роутерах — OSPF сосед виден
6. `ping 192.168.200.200` с WEB-R — маршрутизация через туннель
7. `ssh -p 1984 root@4.4.4.100` с CLI — проброс SSH на WEB-L
8. `ssh -p 1945 root@5.5.5.100` с CLI — проброс SSH на WEB-R
9. `nslookup isp.ANDREW.ZLOY 3.3.3.1` с CLI — DNS первого уровня
10. `nslookup Web-l.MDK.ANDREW.ZLOY 3.3.3.1` с CLI — DNS второго уровня (делегирование)
11. `chronyc sources` на клиентах — NTP синхронизация
12. `df -h` на WEB-L и WEB-R — CIFS шара смонтирована в /opt/share
13. `ls /opt/share/` на WEB-R — файлы от WEB-L видны

---

## ТИПИЧНЫЕ ПРОБЛЕМЫ И РЕШЕНИЯ

| Проблема | Причина | Решение |
|---|---|---|
| Интерфейс DOWN на CLI | NetworkManager конфликтует с etcnet | `systemctl disable NetworkManager` + `NM_CONTROLLED=no` в options |
| EcoRouter видит меньше портов | Баг: первый адаптер невидим | Добавить лишний адаптер, нумерация сдвинута на 1 |
| Пинг с роутера не идёт | Нет маршрута по умолчанию | `ip route 0.0.0.0/0 <шлюз>` |
| BIND не запускается | Дублирование блока options в rndc.key | Удалить лишние блоки из rndc.key |
| WEB-R не монтирует шару с SRV | Нет маршрута между подсетями | Сначала настроить GRE + OSPF |
| SSH Permission denied | PermitRootLogin запрещён | Изменить на `yes` в sshd_config, перезапустить sshd |
| apt-get не работает (resolving) | Нет DNS или нет NAT | Проверить resolv.conf и NAT на роутере |
| Опечатка 172.168 вместо 172.16 | Невнимательность | Тщательно проверять IP-адреса |
