### Домашнее задание к занятию "3.8. Компьютерные сети, лекция 3"  

#### 1. Подключитесь к публичному маршрутизатору в интернет. Найдите маршрут к вашему публичному IP
```shell
telnet route-views.routeviews.org
Username: rviews
show ip route x.x.x.x/32
show bgp x.x.x.x/32
```
```shell
route-views>show ip route 185.170.xxx.xxx
Routing entry for 185.170.xxx.0/22
  Known via "bgp 6447", distance 20, metric 0
  Tag 8283, type external
  Last update from 94.142.247.3 3w2d ago
  Routing Descriptor Blocks:
  * 94.142.247.3, from 94.142.247.3, 3w2d ago
      Route metric is 0, traffic share count is 1
      AS Hops 4
      Route tag 8283
      MPLS label: none
```
```shell
route-views>show bgp 185.170.xxx.xxx
BGP routing table entry for 185.170.xxx.0/22, version 1295019693
Paths: (24 available, best #10, table default)
  Not advertised to any peer
  Refresh Epoch 1
  4901 6079 1299 5504 206912
    162.250.137.254 from 162.250.137.254 (162.250.137.254)
      Origin IGP, localpref 100, valid, external
      Community: 65000:10100 65000:10300 65000:10400
      path 7FE0DF658448 RPKI State not found
      rx pathid: 0, tx pathid: 0
..........
  Refresh Epoch 1
  19214 174 1299 5504 206912
    208.74.64.40 from 208.74.64.40 (208.74.64.40)
      Origin IGP, localpref 100, valid, external
      Community: 174:21000 174:22013
      path 7FE1733FC848 RPKI State not found
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  1351 6939 1299 5504 206912
    132.198.255.253 from 132.198.255.253 (132.198.255.253)
      Origin IGP, localpref 100, valid, external
      path 7FE0DA7650A8 RPKI State not found
      rx pathid: 0, tx pathid: 0
```

#### 2. Создайте dummy0 интерфейс в Ubuntu. Добавьте несколько статических маршрутов. Проверьте таблицу маршрутизации.

Запуск модуля
```shell
# echo "dummy" > /etc/modules-load.d/dummy.conf
# echo "options dummy numdummies=2" > /etc/modprobe.d/dummy.conf
```
Настройка интерфейса
```shell
# cat << "EOF" >> /etc/systemd/network/10-dummy0.netdev
[NetDev]
Name=dummy0
Kind=dummy
EOF
# cat << "EOF" >> /etc/systemd/network/20-dummy0.network
[Match]
Name=dummy0

[Network]
Address=10.0.8.1/24
EOF
#
#
# systemctl restart systemd-networkd
```
Добавление статического маршрута
```shell
# nano /etc/netplan/02-networkd.yaml
network:
  version: 2
  ethernets:
    eth0:
      optional: true
      addresses:
        - 10.0.2.3/24
      routes:
        - to: 10.0.4.0/24
          via: 10.0.2.2
```
Таблица маршрутизации
```shell
# ip r
default via 10.0.2.2 dev eth0 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.3
10.0.2.2 dev eth0 proto dhcp scope link src 10.0.2.15 metric 100
10.0.4.0/24 via 10.0.2.2 dev eth0 proto static
10.0.8.0/24 dev dummy0 proto kernel scope link src 10.0.8.1
```
Статический маршрут
```shell
# ip r | grep static
10.0.4.0/24 via 10.0.2.2 dev eth0 proto static
```


#### 3. Проверьте открытые TCP порты в Ubuntu, какие протоколы и приложения используют эти порты? Приведите несколько примеров.
```shell
# ss -tnlp
State    Recv-Q   Send-Q      Local Address:Port       Peer Address:Port   Process
LISTEN   0        4096              0.0.0.0:111             0.0.0.0:*       users:(("rpcbind",pid=555,fd=4),("systemd",pid=1,fd=35))
LISTEN   0        4096        127.0.0.53%lo:53              0.0.0.0:*       users:(("systemd-resolve",pid=556,fd=13))
LISTEN   0        128               0.0.0.0:22              0.0.0.0:*       users:(("sshd",pid=1325,fd=3))
LISTEN   0        4096                 [::]:111                [::]:*       users:(("rpcbind",pid=555,fd=6),("systemd",pid=1,fd=37))
LISTEN   0        128                  [::]:22                 [::]:*       users:(("sshd",pid=1325,fd=4))
```
:53 - DNS  
:22 - SSH

#### 4. Проверьте используемые UDP сокеты в Ubuntu, какие протоколы и приложения используют эти порты?
```shell
# ss -unap
State    Recv-Q   Send-Q      Local Address:Port       Peer Address:Port   Process
UNCONN   0        0           127.0.0.53%lo:53              0.0.0.0:*       users:(("systemd-resolve",pid=556,fd=12))
UNCONN   0        0          10.0.2.15%eth0:68              0.0.0.0:*       users:(("systemd-network",pid=12712,fd=20))
UNCONN   0        0                 0.0.0.0:111             0.0.0.0:*       users:(("rpcbind",pid=555,fd=5),("systemd",pid=1,fd=36))
UNCONN   0        0                    [::]:111                [::]:*       users:(("rpcbind",pid=555,fd=7),("systemd",pid=1,fd=38))
```
:53 - DNS  
:68 - Используется клиентскими машинами для получения информации о динамической IP-адресации от DHCP-сервера.
#### 5. Используя diagrams.net, создайте L3 диаграмму вашей домашней сети или любой другой сети, с которой вы работали.
![](pic/network_diagram.png)
#### 6*. Установите Nginx, настройте в режиме балансировщика TCP или UDP.

Создаем 4 VM (1-ый - клиент, 2-ой - балансировщик, 3-ий и 4-ый - веб-серверы)

vagrantfile
```shell
boxes = {
  'netology1' => '10',
  'netology2' => '60',
  'netology3' => '90',
  'netology4' => '120'
}

Vagrant.configure("2") do |config|
  config.vm.network "private_network", virtualbox__intnet: true, auto_config: false
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 1
  end
  config.vm.box = "bento/ubuntu-20.04"

  boxes.each do |k, v|
    config.vm.define k do |node|
      node.vm.provision "shell" do |s|
        s.inline = "hostname $1;"\
          "ip addr add $2 dev eth1;"\
          "ip link set dev eth1 up;"\
          "apt -y update;"\
          "apt -y install nginx;"\
          "mkdir -p /data/www;"\
          "echo Hello from $1 >> /data/www/index.html;"
        s.args = [k, "172.28.128.#{v}/24"]
      end
    end
  end
end
```
На балансировщике (VM2) добавляем конфиг
```shell
$ sudo nano /etc/nginx/conf.d/proxyTCP.conf
     upstream backend1 {
         server 172.28.128.90:8080;
         server 172.28.128.120:8080;
     }
     server {
         listen 8080;
         location / {
             proxy_pass http://backend1;
         }
     }

$ sudo nginx -s reload
```
На веб-серверах (VM3, VM4) меняем конфиги
```shell
$ sudo nano /etc/nginx/sites-enabled/default
server {
     listen 8080;
     location / {
             root /data/www;
             index  index.html index.htm;
     }
}

$ sudo nginx -s reload
```
Отдаем запрос с VM1
```shell
$ curl 172.28.128.60:8080
Hello from netology3
$ curl 172.28.128.60:8080
Hello from netology4
$ curl 172.28.128.60:8080
Hello from netology3
$ curl 172.28.128.60:8080
Hello from netology4
```

#### 7*. Установите bird2, настройте динамический протокол маршрутизации RIP.

#### 8*. Установите Netbox, создайте несколько IP префиксов, используя curl проверьте работу API.

Установка docker
```shell
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# sudo apt-get update
# sudo apt-get install docker-ce docker-ce-cli containerd.io
# apt install docker-compose
```
Запуск Netbox
```shell
# git clone -b release https://github.com/netbox-community/netbox-docker.git
# cd netbox-docker
# tee docker-compose.override.yml <<EOF
version: '3.4'
services:
  netbox:
    ports:
      - 8000:8080
EOF
# docker-compose pull
# docker-compose up
```
Запрос на создание префикса через `curl`
```shell
$ sudo curl -ss -X POST -H "Authorization: Token 0123456789abcdef0123456789abcdef01234567" -H "Content-Type: application/json" -H "Accept: application/json; indent=4" http://10.0.2.15:8000/api/ipam/prefixes/ --data '{"prefix": "10.0.8.0/24"}'
{
    "id": 8,
    "url": "http://10.0.2.15:8000/api/ipam/prefixes/8/",
    "display": "10.0.8.0/24",
    "family": {
        "value": 4,
        "label": "IPv4"
    },
    "prefix": "10.0.8.0/24",
    "site": null,
    "vrf": null,
    "tenant": null,
    "vlan": null,
    "status": {
        "value": "active",
        "label": "Active"
    },
    "role": null,
    "is_pool": false,
    "mark_utilized": false,
    "description": "",
    "tags": [],
    "custom_fields": {},
    "created": "2021-12-02",
    "last_updated": "2021-12-02T15:03:45.193570Z",
    "children": 0,
    "_depth": 0
}
```
![](pic/netbox.png)
