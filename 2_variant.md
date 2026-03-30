1.Название машин
# Например для ISP:
hostnamectl set-hostname isp.andrew.zloy
#На RTR-L и RTR-R (EcoRouter)
hostname rtr-l.andrew.zloy
hostname rtr-r.andrew.zloy 

Окей, значит на ISP:

ens33 — NAT (выход в интернет)
ens37 — к RTR-L (4.4.4.1/24)
ens38 — к CLI (3.3.3.1/24)
ens39 — к RTR-R (5.5.5.1/24)

# ens37 — к RTR-L
mkdir -p /etc/net/ifaces/ens37
cat > /etc/net/ifaces/ens37/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "4.4.4.1/24" > /etc/net/ifaces/ens37/ipv4address

# ens38 — к CLI
mkdir -p /etc/net/ifaces/ens38
cat > /etc/net/ifaces/ens38/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "3.3.3.1/24" > /etc/net/ifaces/ens38/ipv4address

# ens39 — к RTR-R
mkdir -p /etc/net/ifaces/ens39
cat > /etc/net/ifaces/ens39/options << 'EOF'
TYPE=eth
BOOTPROTO=static
EOF
echo "5.5.5.1/24" > /etc/net/ifaces/ens39/ipv4address

# ens33 — NAT интерфейс, скорее всего уже настроен через DHCP
# Проверь его
cat /etc/net/ifaces/ens33/options

Затем:
bash# IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -p /etc/net/sysctl.conf

# NAT
iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Перезапуск сети
service network restart

# Проверка
ip a

RTR-L
port ge0
  service-instance to-isp
    encapsulation untagged
    exit

port ge1
  service-instance to-lan
    encapsulation untagged
    exit

interface to-isp
  connect port ge0 service-instance to-isp
  ip address 4.4.4.100/24
  exit

interface to-lan
  connect port ge1 service-instance to-lan
  ip address 192.168.200.254/24
  exit

ip route 0.0.0.0/0 4.4.4.1

Потом проверь:
do show ip route
do ping 4.4.4.1

**если слетает на клиенте не дает делать пинг
vim /etc/net/ifaces/ens33/options 
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
ONBOOT=yes
NM_CONTROLLED=no
DISABLED=no

** Задание 2 — Проброс SSH (DNAT на EcoRouter) **

*RTR-L:
conf t
ip nat source static tcp 192.168.200.100 22 4.4.4.100 1984
*RTR-R:
conf t
ip nat source static tcp 172.16.100.100 22 5.5.5.100 1945
*RTR-L:
ip nat pool NAT 192.168.200.1-192.168.200.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
  ip nat outside
interface to-lan
  ip nat inside
*RTR-R:
ip nat pool NAT 172.16.100.1-172.16.100.254
ip nat source dynamic inside-to-outside pool NAT overload interface to-isp

interface to-isp
  ip nat outside
interface to-lan
  ip nat inside

* WEB-L
vim /etc/openssh/sshd_config
Найди строку `PermitRootLogin` и поставь:
```
PermitRootLogin yes
systemctl restart sshd
ssh -p 1984 root@4.4.4.100
* WEB-R
vim /etc/openssh/sshd_config
Найди строку `PermitRootLogin` и поставь:
```
PermitRootLogin yes
systemctl restart sshd
ssh -p 1945 root@5.5.5.100

если что менять пароль это `passwd root`

**Задание 3 — GRE туннель**

*RTR-L:
conf t
interface tunnel.1
  ip address 10.0.0.1/30
  ip tunnel 4.4.4.100 5.5.5.100 mode gre
  exit
*RTR-R:
conf t
interface tunnel.1
  ip address 10.0.0.2/30
  ip tunnel 5.5.5.100 4.4.4.100 mode gre
  exit
Проверка: do ping 10.0.0.2 с RTR-L и do ping 10.0.0.1 с RTR-R.

**Задание 4 — OSPF с аутентификацией**

*RTR-L:
conf t
router ospf 1
  network 192.168.200.0/24 area 0
  network 10.0.0.0/30 area 0
  exit

interface tunnel.1
  ip ospf authentication
  ip ospf authentication-key P@ssw0rd
  exit
*RTR-R:
conf t
router ospf 1
  network 172.16.100.0/24 area 0
  network 10.0.0.0/30 area 0
  exit

interface tunnel.1
  ip ospf authentication
  ip ospf authentication-key P@ssw0rd
  exit
Проверка: do show ip ospf neighbor — должен появиться сосед. 
После этого ping 192.168.200.100 с WEB-R должен ходить через туннель.

**Задание 5 — DNS первого уровня на ISP (зона ANDREW.ZLOY)**

apt-get install bind bind-utils -y
Файл /etc/bind/options.conf — найди секцию options и добавь/измени:
bashvim /etc/bind/options.conf
```
Внутри `options { ... }` убедись что есть:
```
listen-on { any; };
allow-query { any; };
allow-recursion { any; };
forwarders { 77.88.8.8; };
Файл /etc/bind/rfc1912.conf — добавь зону:
bashcat >> /etc/bind/rfc1912.conf << 'EOF'

zone "ANDREW.ZLOY" {
    type master;
    file "ANDREW.ZLOY";
    allow-update { none; };
};
EOF
Создай файл зоны:
bashcp /etc/bind/zone/empty /var/lib/bind/etc/zone/ANDREW.ZLOY
cat > /var/lib/bind/etc/zone/ANDREW.ZLOY << 'EOF'
$TTL 86400
@       IN  SOA ANDREW.ZLOY. root.ANDREW.ZLOY. (
            2025010101
            3600
            1800
            604800
            86400
)
        IN  NS   isp.ANDREW.ZLOY.
isp     IN  A    3.3.3.1
www     IN  A    4.4.4.100
www     IN  A    5.5.5.100
Internet IN CNAME isp.ANDREW.ZLOY.
@       IN  NS   isp.ANDREW.ZLOY.
MDK     IN  NS   srv.MDK.ANDREW.ZLOY.
srv.MDK IN  A    4.4.4.100
EOF
```

Делегирование MDK.ANDREW.ZLOY идёт на адрес 4.4.4.100 (внешний адрес RTR-L), потому что SRV во внутренней сети. Поэтому **RTR-L должен пробрасывать DNS**:

**На RTR-L:**

ip nat source static tcp 192.168.200.200 53 4.4.4.100 53
ip nat source static udp 192.168.200.200 53 4.4.4.100 53
Настрой resolv.conf на CLI для использования DNS ISP:
bash# На CLI:
echo "nameserver 3.3.3.1" > /etc/resolv.conf
Запусти BIND на ISP:
bashrndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
systemctl enable --now bind
systemctl restart bind

Если ошибка: 
named-checkconf /etc/bind/named.conf
named-checkzone ANDREW.ZLOY /var/lib/bind/etc/zone/ANDREW.ZLOY
vim /var/lib/bind/etc/rndc.key
```

Оставь только:
```
key "rndc-key" {
    algorithm hmac-sha256;
    secret "...тут_ключ...";
};
Всё остальное (options, server) удали из этого файла.
 Проверка зоны — правильный синтаксис:
bashnamed-checkzone ANDREW.ZLOY /var/lib/bind/etc/zone/ANDREW.ZLOY
После исправления rndc.key:
bashnamed-checkconf /etc/bind/named.conf
systemctl restart bind
systemctl status bind

**Задание 6 — DNS второго уровня на SRV (зона MDK.ANDREW.ZLOY)**

apt-get update
apt-get install bind bind-utils -y
vim /etc/bind/rfc1912.conf:

zone "MDK.ANDREW.ZLOY" {
    type master;
    file "MDK.ANDREW.ZLOY";
    allow-update { none; };
};

Создай файл зоны:
cp /etc/bind/zone/empty /var/lib/bind/etc/zone/MDK.ANDREW.ZLOY
vim /var/lib/bind/etc/zone/MDK.ANDREW.ZLOY 
$TTL 86400
@       IN  SOA MDK.ANDREW.ZLOY. root.MDK.ANDREW.ZLOY. (
            2025010101
            3600
            1800
            604800
            86400
)
        IN  NS   srv.MDK.ANDREW.ZLOY.
srv     IN  A    192.168.200.200
Web-l   IN  A    192.168.200.100
Web-r   IN  A    172.16.100.100
rtr-l   IN  A    192.168.200.254
rtr-r   IN  A    172.16.100.254
ntp     IN  CNAME isp.ANDREW.ZLOY.
dns     IN  CNAME srv.MDK.ANDREW.ZLOY.


В vim /etc/bind/options.conf на SRV добавь:
```
listen-on { any; };
allow-query { any; };
allow-recursion { any; };

Запусти:
rndc-confgen > /var/lib/bind/etc/rndc.key
sed -i '6,$d' /var/lib/bind/etc/rndc.key
systemctl enable --now bind
systemctl restart bind
vim /etc/resolv.conf "nameserver 192.168.200.200"

**Задание 7 — NTP на ISP (chrony, stratum=3)**

На ISP:
upt-det update
apt-get install chrony -y

vim /etc/chrony.conf

local stratum 3
allow 0.0.0.0/0

driftfile /var/lib/chrony/drift

systemctl enable --now chronyd
systemctl restart chronyd

На всех клиентах (SRV, WEB-L, WEB-R, CLI):
apt-get update
apt-get install chrony -y
vim /etc/chrony.conf 

server 3.3.3.1 iburst
driftfile /var/lib/chrony/drift

systemctl enable --now chronyd
systemctl restart chronyd

**На роутерах:**

ntp timezone utc+3
ntp server 4.4.4.1
(RTR-R: ntp server 5.5.5.1)

**Задание 8 — Яндекс.Браузер на CLI**
apt-get install yandex-browser-stable -y

**Задание 9 — SMB-сервер на SRV с RAID 1 (зеркало)**

Судя по lsblk:

sda = 30 ГБ — СИСТЕМНЫЙ, НЕ ТРОГАЙ
sdb = 1 ГБ — новый диск ✓
sdc = 1 ГБ — новый диск ✓

Ответь y на вопрос если команда ещё не запустилась, но если она уже пошла с /dev/sda — нажми Ctrl+C немедленно!
Правильная команда:
bashmdadm --create /dev/md0 -l 1 -n 2 /dev/sdb /dev/sdc
После создания:
bash# Форматируй
mkfs --type ext4 /dev/md0

# Создай точку монтирования
mkdir -p /mnt/storage

# Добавь в fstab
echo "/dev/md0  /mnt/storage  ext4  defaults  0  0" >> /etc/fstab

# Монтируй
mount -a

# Проверь
lsblk
Потом Samba:
bash# Установи
apt-get install samba -y

ip route

# Проверь DNS
cat /etc/resolv.conf

# Проверь пинг до шлюза
ping 192.168.200.254

# Проверь пинг до ISP
ping 4.4.4.1

echo "nameserver 77.88.8.8" > /etc/resolv.conf

# Попробуй снова
apt-get install samba -y


# Создай папку для шары
mkdir -p /mnt/storage/share
chmod 777 /mnt/storage/share

# Конфиг
vim /etc/samba/smb.conf 
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


# Запусти
systemctl enable --now smb
systemctl restart smb

**Задание 10 — CIFS монтирование на WEB-L и WEB-R**

* WEB-L:
apt-get install cifs-utils -y
mkdir -p /opt/share
mount -t cifs //192.168.200.200/share /opt/share -o guest
echo "//192.168.200.200/share  /opt/share  cifs  guest  0  0" >> /etc/fstab
*Проверь:
# Создай файл
touch /opt/share/test_from_webl
ls /opt/share/
* WEB-R:
bashapt-get install cifs-utils -y
mkdir -p /opt/share
mount -t cifs //192.168.200.200/share /opt/share -o guest
echo "//192.168.200.200/share  /opt/share  cifs  guest  0  0" >> /etc/fstab
Проверь:
ls /opt/share/
Если на WEB-R видно файл test_webl — всё работает. 
Добавь автомонтирование на обоих:
echo "//192.168.200.200/share  /opt/share  cifs  guest  0  0" >> /etc/fstab
