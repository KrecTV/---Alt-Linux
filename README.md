
---

# Строим корпоративную сеть на ALT Linux: от VLAN до OSPF и DNS

В этой статье мы детально разберём практическую настройку сегмента сети предприятия на базе ALT Linux. Будет всё: создание VLAN, настройка межсетевого экрана и NAT, поднятие GRE-туннеля между офисами, запуск OSPF для динамической маршрутизации, установка DHCP- и DNS-серверов. Материал построен как пошаговое руководство, которое можно использовать для повторения на виртуальном стенде или реальном оборудовании.

Все команды вводятся от суперпользователя (root). Имена интерфейсов, IP-адреса, названия хостов заменены на обобщённые, чтобы вы могли легко адаптировать их под свою среду. Например, `isp.example.com` вместо реального домена, `10.10.1.1/30` вместо конкретных подсетей.

---

## 1. Настройка маршрутизатора ISP (пограничный узел)

### 1.1. Назначение имени хоста
Первым делом зададим системе понятное доменное имя. Это упрощает администрирование и идентификацию устройства.

```bash
hostnamectl hostname isp.example.com
exec bash
```
Команда `hostnamectl` изменяет статическое имя хоста в systemd, а `exec bash` перезапускает оболочку, чтобы применить изменения сразу.

### 1.2. Создание конфигурации для внутренних интерфейсов
У провайдера три сетевых адаптера: один смотрит в интернет (адрес по DHCP), два других — в локальные офисы. Для статических интерфейсов создадим отдельные каталоги в `/etc/net/ifaces` – стандартном месте хранения сетевых настроек в ALT Linux.

```bash
cd /etc/net/ifaces
mkdir enp0s3 enp0s4
```

Теперь зададим опции для каждого интерфейса. Флаги `TYPE=eth`, `BOOTPROTO=static`, `ONBOOT=yes` означают, что это проводной Ethernet, адрес назначается статически, и интерфейс должен подниматься при загрузке.

```bash
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s3/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s4/options
```

Прописываем IP-адреса и префиксы маски. Формат `X.X.X.X/YY` задаёт адрес и длину префикса подсети (CIDR).

```bash
echo 10.10.1.1/30 > enp0s3/ipv4address
echo 10.10.2.1/30 > enp0s4/ipv4address
```

Перезапускаем сетевую службу, чтобы применить созданные файлы.

```bash
systemctl restart network
```

### 1.3. Включение NAT и IP-форвардинга
Чтобы офисные сети имели доступ в интернет, маршрутизатор должен передавать пакеты между интерфейсами и подменять локальные адреса на свой внешний.

Устанавливаем необходимые пакеты: `nano` для редактирования конфигов, `iptables` для управления фильтрацией и сетевым адресом перевода.

```bash
apt-get update
apt-get install -y nano iptables
```

Активируем пересылку пакетов на уровне ядра. Параметр `net.ipv4.ip_forward` должен быть равен 1.

```bash
nano /etc/net/sysctl.conf
# находим или добавляем строку:
net.ipv4.ip_forward = 1

sysctl -p /etc/net/sysctl.conf
```

Теперь настраиваем маскарадинг на внешнем интерфейсе. Весь трафик, уходящий через `enp0s2` (DHCP-интерфейс к провайдеру), будет скрыт за его адресом.

```bash
iptables -t nat -A POSTROUTING -o enp0s2 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

### 1.4. Синхронизация времени
Для корректной работы сервисов и протоколов важно, чтобы на всех устройствах было одинаковое время. Установим часовой пояс на примере московского времени.

```bash
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status
```

---

## 2. Маршрутизатор головного офиса (HQ-RTR)

### 2.1. Имя хоста
```bash
hostnamectl hostname hq.example.com
exec bash
```

### 2.2. Подготовка интерфейсов и VLAN
Маршрутизатор головного офиса имеет один физический порт в сторону ISP и один внутренний порт, который будет разделён на несколько VLAN для разных сегментов сети.

```bash
cd /etc/net/ifaces
mkdir enp0s3 enp0s4 enp0s4.100 enp0s4.200 enp0s4.999
```

Настраиваем физические интерфейсы как Ethernet с статическим IP.

```bash
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s3/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s4/options
```

Для VLAN-интерфейсов помимо типа указываем родительский интерфейс (`HOST`) и идентификатор VLAN (`VID`).

```bash
echo -e "TYPE=vlan\nHOST=enp0s4\nVID=100\nBOOTPROTO=static\nONBOOT=yes" > enp0s4.100/options
echo -e "TYPE=vlan\nHOST=enp0s4\nVID=200\nBOOTPROTO=static\nONBOOT=yes" > enp0s4.200/options
echo -e "TYPE=vlan\nHOST=enp0s4\nVID=999\nBOOTPROTO=static\nONBOOT=yes" > enp0s4.999/options
```

Назначаем адреса каждому интерфейсу. Внешний порт получает адрес из общей сети с ISP, а внутренние VLAN — частные адреса.

```bash
echo 10.10.1.2/30 > enp0s3/ipv4address
echo 192.168.10.1/29 > enp0s4.100/ipv4address
echo 192.168.20.1/24 > enp0s4.200/ipv4address
echo 192.168.30.1/29 > enp0s4.999/ipv4address
```

Указываем маршрут по умолчанию через ISP.

```bash
echo default via 10.10.1.1 > enp0s3/ipv4route
```

### 2.3. Временная настройка DNS
Пока не поднят собственный DNS-сервер, прописываем публичные серверы имён. Это позволит системе обновлять пакеты и выходить в интернет.

```bash
echo nameserver 8.8.8.8 > enp0s3/resolv.conf
echo nameserver 8.8.8.8 > enp0s4.100/resolv.conf
echo nameserver 8.8.8.8 > enp0s4.200/resolv.conf
echo nameserver 8.8.8.8 > enp0s4.999/resolv.conf
systemctl restart network
```

### 2.4. Форвардинг и NAT на маршрутизаторе
Повторяем настройку, аналогичную ISP, чтобы устройства в локальных сетях могли выходить в интернет через внешний интерфейс.

```bash
apt-get update
apt-get install -y nano iptables

nano /etc/net/sysctl.conf
# net.ipv4.ip_forward = 1

sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

### 2.5. Создание административной учётной записи
Пользователь `net_admin` будет использоваться для удалённого управления маршрутизатором. Ему предоставляется полный доступ через sudo без ввода пароля.

```bash
useradd -m net_admin
passwd net_admin
usermod -aG wheel net_admin
echo "net_admin ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```

### 2.6. Настройка DHCP-сервера
Клиентская подсеть (VLAN 200) должна автоматически получать сетевые параметры. Устанавливаем и конфигурируем ISC DHCP-сервер.

```bash
apt-get install -y dhcp-server
cd /etc/dhcp
cp dhcpd.conf.sample dhcpd.conf

nano dhcpd.conf
```

Внутри файла задаём пул адресов, шлюз, DNS-суффикс и адрес DNS-сервера (позже им станет наш сервер HQ-SRV). В данном примере пул ограничен одним адресом, чтобы гарантировать соответствие DNS-записи.

```ini
ddns-update-style none;

subnet 192.168.20.0 netmask 255.255.255.0 {
        option routers                  192.168.20.1;
        option subnet-mask              255.255.255.0;
        option nis-domain               "example.com";
        option domain-name              "example.com";
        option domain-name-servers      192.168.10.2;
        range dynamic-bootp 192.168.20.2 192.168.20.2;
        default-lease-time 21600;
        max-lease-time 43200;
}
```

Запускаем сервис:

```bash
systemctl enable --now dhcpd
```

### 2.7. Часовой пояс
```bash
apt-get install -y tzdata
timedatectl set-timezone Europe/Moscow
timedatectl status
```

### 2.8. GRE-туннель до филиала
Для связи двух офисов напрямую через интернет создаём GRE-туннель. Он инкапсулирует IP-пакеты и позволяет маршрутизаторам “видеть” друг друга, как в общей сети.

```bash
cd /etc/net/ifaces
mkdir gre1
echo -e "TYPE=iptun\nTUNTYPE=gre\nTUNLOCAL=10.10.1.2\nTUNREMOTE=10.10.2.2\nTUNTTL=64\nTUNOPTIONS='ttl 64'" > gre1/options
echo 172.16.0.1/30 > gre1/ipv4address
systemctl restart network
```
Параметры: `TUNLOCAL` — локальный IP ISP-интерфейса, `TUNREMOTE` — удалённый IP филиала. Туннелю присваивается адрес из отдельной подсети.

### 2.9. Запуск OSPF для динамической маршрутизации
Протокол OSPF (link-state) будет анонсировать локальные сети через туннель, чтобы филиал знал о сетях главного офиса и наоборот. Используем пакет FRR.

```bash
apt-get install -y frr
nano /etc/frr/daemons
# проверяем, что строка ospfd=yes раскомментирована

systemctl enable --now frr.service
```

Конфигурируем OSPF через встроенную оболочку `vtysh`. Включим пассивный режим по умолчанию для безопасности и разрешим обмен только на GRE-интерфейсе с аутентификацией.

```bash
vtysh <<EOF
conf t
router ospf
passive-interface default
network 192.168.10.0/29 area 0
network 192.168.20.0/24 area 0
network 192.168.30.0/29 area 0
network 172.16.0.0/30 area 0
exit
interface gre1
no ip ospf passive
ip ospf authentication-key 1234
end
wr mem
EOF
exit
```

### 2.10. Переключение на локальный DNS-сервер
Когда сервер HQ-SRV с dnsmasq будет готов, заменим публичные DNS на его адрес. Это обеспечит разрешение внутренних имён.

```bash
cd /etc/net/ifaces
echo nameserver 192.168.10.2 > enp0s3/resolv.conf
echo nameserver 192.168.10.2 > enp0s4.100/resolv.conf
echo nameserver 192.168.10.2 > enp0s4.200/resolv.conf
echo nameserver 192.168.10.2 > enp0s4.999/resolv.conf
systemctl restart network
```

---

## 3. Сервер главного офиса (HQ-SRV)

### 3.1. Имя хоста и VLAN-интерфейс
Сервер находится в VLAN 100, поэтому для основного адаптера создаём тегированный подинтерфейс.

```bash
hostnamectl hostname srv-hq.example.com
exec bash

cd /etc/net/ifaces
mkdir enp0s3 enp0s3.100
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s3/options
echo -e "TYPE=vlan\nHOST=enp0s3\nVID=100\nBOOTPROTO=static\nONBOOT=yes" > enp0s3.100/options

echo 192.168.10.2/29 > enp0s3.100/ipv4address
echo default via 192.168.10.1 > enp0s3.100/ipv4route
echo nameserver 8.8.8.8 > enp0s3.100/resolv.conf
systemctl restart network
```

### 3.2. Установка и настройка dnsmasq
Dnsmasq — лёгкий DNS-сервер, который будет обслуживать внутренние запросы и пересылать внешние на указанный публичный сервер.

```bash
apt-get update
apt-get install -y nano dnsmasq

nano /etc/dnsmasq.conf
```

Добавляем следующие строки:
```
listen-address=192.168.10.2
server=8.8.8.8
address=/hq.example.com/10.10.1.2
ptr-record=2.1.10.10.in-addr.arpa,hq.example.com
address=/br.example.com/10.10.2.2
address=/srv-hq.example.com/192.168.10.2
ptr-record=2.10.168.192.in-addr.arpa,srv-hq.example.com
address=/cli-hq.example.com/192.168.20.2
ptr-record=2.20.168.192.in-addr.arpa,cli-hq.example.com
address=/srv-br.example.com/192.168.3.2
address=/docker.example.com/10.10.1.1
address=/web.example.com/10.10.2.1
```

Запускаем сервис:

```bash
systemctl enable --now dnsmasq
```

### 3.3. Пользователь для удалённого доступа
Создаём учётную запись `sshuser` с заданным UID (2026) и правом на sudo без пароля.

```bash
useradd -u 2026 -m sshuser
passwd sshuser
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers
```

### 3.4. Настройка SSH-сервера
Для повышения безопасности переносим SSH на нестандартный порт 2026, ограничиваем доступ только пользователю `sshuser`, уменьшаем количество попыток аутентификации и добавляем предупредительный баннер.

```bash
apt-get install -y openssh-server

nano /etc/openssh/sshd_config
# изменяем/добавляем:
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner.txt

echo "Authorized access only" > /etc/openssh/banner.txt
systemctl enable --now sshd
```

### 3.5. Часовой пояс
```bash
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status
```

---

## 4. Маршрутизатор филиала (BR-RTR)

### 4.1. Имя хоста и интерфейсы
Настройка аналогична головному маршрутизатору, но без VLAN – филиал использует обычную плоскую сеть для сервера.

```bash
hostnamectl hostname br.example.com
exec bash

cd /etc/net/ifaces
mkdir enp0s3 enp0s4
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s3/options
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s4/options

echo 10.10.2.2/30 > enp0s3/ipv4address
echo 192.168.3.1/29 > enp0s4/ipv4address

echo default via 10.10.2.1 > enp0s3/ipv4route

echo nameserver 8.8.8.8 > enp0s3/resolv.conf
echo nameserver 8.8.8.8 > enp0s4/resolv.conf
systemctl restart network
```

### 4.2. NAT, пользователь, время
Повторяем стандартные действия: форвардинг, маскарадинг на внешнем интерфейсе, создание `net_admin`, установка часового пояса. (Для краткости команды те же, что и в пункте 2.4–2.5).

### 4.3. GRE-туннель
Создаём парный туннель со стороны филиала.

```bash
cd /etc/net/ifaces
mkdir gre1
echo -e "TYPE=iptun\nTUNTYPE=gre\nTUNLOCAL=10.10.2.2\nTUNREMOTE=10.10.1.2\nTUNTTL=64\nTUNOPTIONS='ttl 64'" > gre1/options
echo 172.16.0.2/30 > gre1/ipv4address
systemctl restart network
```

### 4.4. OSPF
Запускаем FRR и настраиваем анонсирование сети филиала и туннельного стыка.

```bash
apt-get install -y frr
nano /etc/frr/daemons
systemctl enable --now frr.service

vtysh <<EOF
conf t
router ospf
passive-interface default
network 192.168.3.0/29 area 0
network 172.16.0.0/30 area 0
exit
interface gre1
no ip ospf passive
ip ospf authentication-key 1234
end
wr mem
EOF
exit
```

### 4.5. Переключение DNS на центральный сервер
```bash
cd /etc/net/ifaces
echo nameserver 192.168.10.2 > enp0s3/resolv.conf
echo nameserver 192.168.10.2 > enp0s4/resolv.conf
systemctl restart network
```

---

## 5. Сервер филиала (BR-SRV)

### 5.1. Адаптация имени хоста и сети
```bash
hostnamectl hostname srv-br.example.com
exec bash

cd /etc/net/ifaces
mkdir enp0s3
echo -e "TYPE=eth\nBOOTPROTO=static\nONBOOT=yes" > enp0s3/options
echo 192.168.3.2/29 > enp0s3/ipv4address
echo default via 192.168.3.1 > enp0s3/ipv4route
echo nameserver 8.8.8.8 > enp0s3/resolv.conf
systemctl restart network
```

### 5.2. Пользователь, SSH, время и финальный DNS
Создаём `sshuser` (uid 2026), настраиваем sshd аналогично серверу HQ. В конце меняем DNS на 192.168.10.2.

---

## 6. Клиентская машина (HQ-CLI)

### 6.1. Минимальная настройка
Клиент работает в VLAN 200 и получает IP по DHCP. В ALT Linux Workstation VLAN можно добавить через графический интерфейс (NetworkManager), указав родительский интерфейс, идентификатор 200 и автоматическое получение адреса.

```bash
hostnamectl hostname cli-hq.example.com
exec bash
```

После настройки подключения выполняем:

```bash
apt-get install tzdata -y
timedatectl set-timezone Europe/Moscow
timedatectl status
```

---

## Заключение
Мы последовательно сконфигурировали все узлы небольшой распределённой сети. Теперь главный офис и филиал объединены защищённым туннелем, обмениваются маршрутами через OSPF, а клиентские машины автоматически получают сетевые реквизиты. Центральный DNS-сервер разрешает как внутренние, так и внешние имена.

Статья специально написана так, чтобы её можно было использовать как шпаргалку: каждая команда предваряется пояснением, а все специфические адреса заменены на обобщённые. Это позволяет легко адаптировать материал под требования конкретной лабораторной работы или экзаменационного задания, не нарушая правил конфиденциальности. Успешной настройки!
