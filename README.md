### Курсовая работа по итогам модуля "DevOps и системное администрирование"

<details>

#### 1. Создайте виртуальную машину Linux.
```
% vagrant ssh
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 06 Dec 2021 05:16:31 PM UTC

  System load:  1.93              Processes:             119
  Usage of /:   2.3% of 61.31GB   Users logged in:       0
  Memory usage: 15%               IPv4 address for eth0: 10.0.2.15
  Swap usage:   0%


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
vagrant@vagrant:~$
```
#### 2. Установите ufw и разрешите к этой машине сессии на порты 22 и 443, при этом трафик на интерфейсе localhost (lo) должен ходить свободно на все порты.
```
vagrant@vagrant:~$ sudo ufw status
Status: inactive
vagrant@vagrant:~$ sudo ufw allow 22
Rules updated
Rules updated (v6)
vagrant@vagrant:~$ sudo ufw allow 443
Rules updated
Rules updated (v6)
vagrant@vagrant:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
vagrant@vagrant:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
443                        ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
443 (v6)                   ALLOW       Anywhere (v6)
```
#### 3. Установите hashicorp vault (инструкция по ссылке).
```
vagrant@vagrant:~$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
OK
vagrant@vagrant:~$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
vagrant@vagrant:~$ sudo apt-get update && sudo apt-get install vault
vagrant@vagrant:~$ sudo vault
Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    agent       Start a Vault agent
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit          Interact with audit devices
    auth           Interact with auth methods
    debug          Runs the debug command
    kv             Interact with Vault's Key-Value storage
    lease          Interact with leases
    monitor        Stream log messages from a Vault server
    namespace      Interact with namespaces
    operator       Perform operator-specific tasks
    path-help      Retrieve API help for paths
    plugin         Interact with Vault plugins and catalog
    policy         Interact with policies
    print          Prints runtime configurations
    secrets        Interact with secrets engines
    ssh            Initiate an SSH session
    token          Interact with tokens
```
#### 4. Создайте центр сертификации по инструкции (ссылка), и выпустите сертификат для использования его в настройке веб-сервера nginx (срок жизни сертификата - месяц).

Запуск Vault server в dev-режиме
```
vagrant@vagrant:~$ sudo vault server -dev -dev-root-token-id 2mFgnI7QiRtCfQT4ynGQUdKe4N
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.2
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.9.0

==> Vault server started! Log data will stream in below:
....
```
```
root@vagrant:~# export VAULT_ADDR='http://127.0.0.1:8200'
root@vagrant:~# export VAULT_TOKEN=2mFgnI7QiRtCfQT4ynGQUdKe4N
```
```
root@vagrant:~# vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.9.0
Storage Type    inmem
Cluster Name    vault-cluster-d18425f4
Cluster ID      a492c217-c0f4-2411-7d10-0066ac1be454
HA Enabled      false
```
Создание Root CA и Intermediate CA
```
root@vagrant:~# vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/

root@vagrant:~# vault secrets tune -max-lease-ttl=8760h pki
Success! Tuned the secrets engine at: pki/

root@vagrant:~# vault write -field=certificate pki/root/generate/internal common_name="example.com" ttl=87600h > CA_cert.crt

root@vagrant:~# vault write pki/config/urls issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"
Success! Data written to: pki/config/urls

root@vagrant:~# vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/

root@vagrant:~# vault secrets tune -max-lease-ttl=8760h pki_int
Success! Tuned the secrets engine at: pki_int/

root@vagrant:~# apt install jq

root@vagrant:~# vault write -format=json pki_int/intermediate/generate/internal common_name="example.com Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr

root@vagrant:~# vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl="8760h" | jq -r '.data.certificate' > intermediate.cert.pem

root@vagrant:~# vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed

root@vagrant:~# vault write pki_int/roles/example-dot-com allowed_domains="example.com" allow_subdomains=true max_ttl="4380h"
Success! Data written to: pki_int/roles/example-dot-com

root@vagrant:~# vault list pki_int/roles/
Keys
----
example-dot-com
```
Создание сертификатов для devops.example.com
```
root@vagrant:~# vault write -format=json pki_int/issue/example-dot-com common_name="devops.example.com" ttl=720h > devops.example.com.crt

root@vagrant:~# cat devops.example.com.crt
....
serial_number       40:fa:18:00:fb:7c:9b:97:95:50:10:da:2f:48:7f:f7:48:08:c1:4a

root@vagrant:~# cat devops.example.com.crt | jq -r .data.certificate > devops.example.com.crt.pem

root@vagrant:~# cat devops.example.com.crt | jq -r .data.issuing_ca >> devops.example.com.crt.pem

root@vagrant:~# cat devops.example.com.crt | jq -r .data.private_key > devops.example.com.crt.key
```
#### 5. Установите корневой сертификат созданного центра сертификации в доверенные в хостовой системе.
```
root@vagrant:~# ln -s /root/CA_cert.crt /usr/local/share/ca-certificates/CA_cert.crt
root@vagrant:~# update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```
#### 6. Установите nginx.
```
root@vagrant:~# apt install nginx

root@vagrant:~# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2021-12-07 10:15:15 UTC; 11s ago
       Docs: man:nginx(8)
   Main PID: 14592 (nginx)
      Tasks: 3 (limit: 1071)
     Memory: 4.4M
     CGroup: /system.slice/nginx.service
             ├─14592 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             ├─14593 nginx: worker process
             └─14594 nginx: worker process

Dec 07 10:15:15 vagrant systemd[1]: Starting A high performance web server and a reverse proxy server...
Dec 07 10:15:15 vagrant systemd[1]: Started A high performance web server and a reverse proxy server.

root@vagrant:~# nano /etc/hosts
127.0.0.1       localhost
127.0.1.1       vagrant.vm      vagrant
127.0.0.1       devops.example.com

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

root@vagrant:~# ping devops.example.com
PING devops.example.com (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.035 ms
^C
--- devops.example.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1031ms
rtt min/avg/max/mdev = 0.021/0.028/0.035/0.007 ms
```
#### 7. По инструкции (ссылка) настройте nginx на https, используя ранее подготовленный сертификат:
- можно использовать стандартную стартовую страницу nginx для демонстрации работы сервера;
- можно использовать и другой html файл, сделанный вами;
```
root@vagrant:~# nano /etc/nginx/sites-enabled/default
....
server {
....

        # SSL configuration
        #
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;
        ssl_certificate /root/devops.example.com.crt.pem;
        ssl_certificate_key /root/devops.example.com.crt.key;
....
root@vagrant:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

root@vagrant:~# systemctl reload nginx
root@vagrant:~# root@vagrant:~# curl -I https://devops.example.com
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 07 Dec 2021 19:22:40 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 07 Dec 2021 19:19:05 GMT
Connection: keep-alive
ETag: "61afb3a9-264"
Accept-Ranges: bytes
```
#### 8. Откройте в браузере на хосте https адрес страницы, которую обслуживает сервер nginx.
![](pic/sert.png)
#### 9. Создайте скрипт, который будет генерировать новый сертификат в vault:
- генерируем новый сертификат так, чтобы не переписывать конфиг nginx;
- перезапускаем nginx для применения нового сертификата.
```
root@vagrant:~# nano sert.sh
#!/bin/bash
vault write -format=json pki_int/issue/example-dot-com common_name="devops.example.com" ttl=720h > /root/devops.example.com.crt
cat /root/devops.example.com.crt | jq -r .data.certificate > /root/devops.example.com.crt.pem
cat /root/devops.example.com.crt | jq -r .data.issuing_ca >> /root/devops.example.com.crt.pem
cat /root/devops.example.com.crt | jq -r .data.private_key > /root/devops.example.com.crt.key
systemctl reload nginx

root@vagrant:~# chmod ugo+x sert.sh
```
![](pic/sert_renew.png)
#### 10. Поместите скрипт в crontab, чтобы сертификат обновлялся какого-то числа каждого месяца в удобное для вас время.
```
root@vagrant:~# crontab -l
....
# m h  dom mon dow   command
0 0 7 * * /root/sert.sh
```

</details>

### Домашнее задание к занятию "3.9. Элементы безопасности информационных систем"

<details>

#### 1. Установите Bitwarden плагин для браузера. Зарегистрируйтесь и сохраните несколько паролей.
![](pic/Bitwarden1.png)
#### 2. Установите Google authenticator на мобильный телефон. Настройте вход в Bitwarden аккаунт через Google authenticator OTP.
![](pic/Bitwarden2.png)
#### 3. Установите apache2, сгенерируйте самоподписанный сертификат, настройте тестовый сайт для работы по HTTPS.
```shell
$ sudo apt install apache2
```
```shell
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
Generating a RSA private key
.............+++++
................+++++
writing new private key to '/etc/ssl/private/apache-selfsigned.key'
-----
.....
```
```shell
$ sudo nano /etc/apache2/conf-available/ssl-params.conf
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
# Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```
```shell
$ sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak
$ sudo nano /etc/apache2/sites-available/default-ssl.conf
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin your_email@example.com
                ServerName localhost

                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>

        </VirtualHost>
</IfModule>
```
```shell
$ sudo nano /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
        . . .

        Redirect "/" "https://localhost/"

        . . .
</VirtualHost>
```
```shell
$ sudo a2enmod ssl
$ sudo a2enmod headers
$ sudo a2ensite default-ssl
$ sudo a2enconf ssl-params
$ sudo apache2ctl configtest
$ sudo systemctl restart apache2
```
![](pic/ssl.png)

#### 4. Проверьте на TLS уязвимости произвольный сайт в интернете.
```shell
# docker run --rm -ti  drwetter/testssl.sh -U --sneaky https://ya.ru
...
 Further IP addresses:   2a02:6b8::2:242
 rDNS (87.250.250.242):  ya.ru.
 Service detected:       HTTP


 Testing vulnerabilities

 Heartbleed (CVE-2014-0160)                not vulnerable (OK), no heartbeat extension
 CCS (CVE-2014-0224)                       not vulnerable (OK)
 Ticketbleed (CVE-2016-9244), experiment.  not vulnerable (OK)
 ROBOT                                     not vulnerable (OK)
 Secure Renegotiation (RFC 5746)           supported (OK)
 Secure Client-Initiated Renegotiation     not vulnerable (OK)
 CRIME, TLS (CVE-2012-4929)                not vulnerable (OK)
 BREACH (CVE-2013-3587)                    no gzip/deflate/compress/br HTTP compression (OK)  - only supplied "/" tested
 POODLE, SSL (CVE-2014-3566)               not vulnerable (OK)
 TLS_FALLBACK_SCSV (RFC 7507)              Downgrade attack prevention supported (OK)
 SWEET32 (CVE-2016-2183, CVE-2016-6329)    VULNERABLE, uses 64 bit block ciphers
 FREAK (CVE-2015-0204)                     not vulnerable (OK)
 DROWN (CVE-2016-0800, CVE-2016-0703)      not vulnerable on this host and port (OK)
                                           make sure you don't use this certificate elsewhere with SSLv2 enabled services
                                           https://censys.io/ipv4?q=26EB381642B07A05F7CA935101FC6492F91F7F0721995A8E577EDFB6723EBD1F could help you to find out
 LOGJAM (CVE-2015-4000), experimental      not vulnerable (OK): no DH EXPORT ciphers, no DH key detected with <= TLS 1.2
 BEAST (CVE-2011-3389)                     TLS1: ECDHE-RSA-AES128-SHA AES128-SHA
                                                 DES-CBC3-SHA
                                           VULNERABLE -- but also supports higher protocols  TLSv1.1 TLSv1.2 (likely mitigated)
 LUCKY13 (CVE-2013-0169), experimental     potentially VULNERABLE, uses cipher block chaining (CBC) ciphers with TLS. Check patches
 Winshock (CVE-2014-6321), experimental    not vulnerable (OK)
 RC4 (CVE-2013-2566, CVE-2015-2808)        no RC4 ciphers detected (OK)
```
#### 5. Установите на Ubuntu ssh сервер, сгенерируйте новый приватный ключ. Скопируйте свой публичный ключ на другой сервер. Подключитесь к серверу по SSH-ключу.
```shell
vagrant@netologyVM1:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
....
vagrant@netologyVM1:~$ ssh-copy-id -i .ssh/id_rsa vagrant@172.28.128.60
....
vagrant@netologyVM1:~$ ssh vagrant@172.28.128.60
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
....
vagrant@netologyVM2:~$
```
#### 6. Переименуйте файлы ключей из задания 5. Настройте файл конфигурации SSH клиента, так чтобы вход на удаленный сервер осуществлялся по имени сервера.
```shell
vagrant@netologyVM1:~$ sudo mv ~/.ssh/id_rsa ~/.ssh/id_rsa_netology
vagrant@netologyVM1:~$ sudo nano ~/.ssh/config
Host netologyVM2
        HostName 172.28.128.60
        User vagrant
        Port 22
        IdentityFile ~/.ssh/id_rsa_netology
vagrant@netologyVM1:~$ ssh netologyVM2
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
....
vagrant@netologyVM2:~$
```
#### 7. Соберите дамп трафика утилитой tcpdump в формате pcap, 100 пакетов. Откройте файл pcap в Wireshark.
```shell
# tcpdump -nnei any -c 100 -w 100packets.pcap
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
100 packets captured
178 packets received by filter
0 packets dropped by kernel
```
![](pic/Wireshark.png)

#### 8*. Просканируйте хост scanme.nmap.org. Какие сервисы запущены?

```shell
# nmap scanme.nmap.org
Starting Nmap 7.80 ( https://nmap.org ) at 2021-12-03 15:52 EET
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.22s latency).
Other addresses for scanme.nmap.org (not scanned): 2600:3c01::f03c:91ff:fe18:bb2f
Not shown: 996 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
9929/tcp  open  nping-echo
31337/tcp open  Elite

Nmap done: 1 IP address (1 host up) scanned in 2.68 seconds
```
Подняты ssh, web-server, nping-echo server, открыт порт 31337

#### 9*. Установите и настройте фаервол ufw на web-сервер из задания 3. Откройте доступ снаружи только к портам 22,80,443
```shell
vagrant@netologyVM1:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80,443/tcp (Apache Full)   ALLOW IN    Anywhere
80/tcp (Apache (v6))       ALLOW IN    Anywhere (v6)
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
80,443/tcp (Apache Full (v6)) ALLOW IN    Anywhere (v6)
```
</details>


### Домашнее задание к занятию "3.8. Компьютерные сети, лекция 3"  

<details>


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


</details>


### Домашнее задание к занятию "3.7. Компьютерные сети, лекция 2"

<details>


#### 1. Проверьте список доступных сетевых интерфейсов на вашем компьютере. Какие команды есть для этого в Linux и в Windows?  

Linux
- `ip link show`
```shell
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether xx:xx:27:73:xx:xx brd ff:ff:ff:ff:ff:ff
```
- `ifconfig -a`
```shell
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether xx:xx:27:73:xx:xx brd ff:ff:ff:ff:ff:ff
vagrant@vagrant:~$ ifconfig -a
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 xxxx::a00:27ff:fe73:xxxx  prefixlen 64  scopeid 0x20<link>
        ether xx:xx:27:73:xx:xx  txqueuelen 1000  (Ethernet)
        RX packets 16095  bytes 15813251 (15.8 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7794  bytes 984770 (984.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 284  bytes 26504 (26.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 284  bytes 26504 (26.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Windows

- `ifconfig -a`


#### 2. Какой протокол используется для распознавания соседа по сетевому интерфейсу? Какой пакет и команды есть в Linux для этого?  

Протокол LLDP.  
Пакет lldpd.  
Команда lldpctl.

#### 3. Какая технология используется для разделения L2 коммутатора на несколько виртуальных сетей? Какой пакет и команды есть в Linux для этого? Приведите пример конфига.  

Технология называется VLAN (Virtual LAN).  
Пакет в Ubuntu Linux - vlan  
Пример конфига:
```shell
network:
  version: 2
  renderer: networkd
  ethernets:
    ens4:
      optional: yes
      addresses: 
        - 192.168.0.2/24
  vlans:
    vlan88:
      id: 88
      link: ens4 
      addresses:
        - 192.168.1.2/24
```

#### 4. Какие типы агрегации интерфейсов есть в Linux? Какие опции есть для балансировки нагрузки? Приведите пример конфига.  

В Linux есть две технологии агрегации (LAG): bonding и teaming.  

Типы агрегации bonding:

```shell
$ modinfo bonding | grep mode:
parm:           mode:Mode of operation; 0 for balance-rr, 1 for active-backup, 2 for balance-xor, 3 for broadcast, 4 for 802.3ad, 5 for balance-tlb, 6 for balance-alb (charp)
```

`active-backup` и `broadcast` обеспечивают только отказоустойчивость  
`balance-tlb`, `balance-alb`, `balance-rr`, `balance-xor` и `802.3ad` обеспечат отказоустойчивость и балансировку

`balance-rr` - Политика round-robin. Пакеты отправляются последовательно, начиная с первого доступного интерфейса и заканчивая последним. Эта политика применяется для балансировки нагрузки и отказоустойчивости.  
`active-backup` - Политика активный-резервный. Только один сетевой интерфейс из объединённых будет активным. Другой интерфейс может стать активным, только в том случае, когда упадёт текущий активный интерфейс. Эта политика применяется для отказоустойчивости.  
`balance-xor` - Политика XOR. Передача распределяется между сетевыми картами используя формулу: [( «MAC адрес источника» XOR «MAC адрес назначения») по модулю «число интерфейсов»]. Получается одна и та же сетевая карта передаёт пакеты одним и тем же получателям. Политика XOR применяется для балансировки нагрузки и отказоустойчивости.  
`broadcast` - Широковещательная политика. Передает всё на все сетевые интерфейсы. Эта политика применяется для отказоустойчивости.  
`802.3ad` - Политика агрегирования каналов по стандарту IEEE 802.3ad. Создаются агрегированные группы сетевых карт с одинаковой скоростью и дуплексом. При таком объединении передача задействует все каналы в активной агрегации, согласно стандарту IEEE 802.3ad. Выбор через какой интерфейс отправлять пакет определяется политикой по умолчанию XOR политика.  
`balance-tlb` - Политика адаптивной балансировки нагрузки передачи. Исходящий трафик распределяется в зависимости от загруженности каждой сетевой карты (определяется скоростью загрузки). Не требует дополнительной настройки на коммутаторе. Входящий трафик приходит на текущую сетевую карту. Если она выходит из строя, то другая сетевая карта берёт себе MAC адрес вышедшей из строя карты.  
`balance-alb` - Политика адаптивной балансировки нагрузки. Включает в себя политику balance-tlb плюс осуществляет балансировку входящего трафика. Не требует дополнительной настройки на коммутаторе. Балансировка входящего трафика достигается путём ARP переговоров.  

`active-backup` на отказоустойчивость:
```shell
 network:
   version: 2
   renderer: networkd
   ethernets:
     ens3:
       dhcp4: no 
       optional: true
     ens5: 
       dhcp4: no 
       optional: true
   bonds:
     bond0: 
       dhcp4: yes 
       interfaces:
         - ens3
         - ens5
       parameters:
         mode: active-backup
         primary: ens3
         mii-monitor-interval: 2
```
`balance-alb` - балансировка:
```shell
   bonds:
     bond0: 
       dhcp4: yes 
       interfaces:
         - ens3
         - ens5
       parameters:
         mode: balance-alb
         mii-monitor-interval: 2
```


#### 5. Сколько IP адресов в сети с маской /29 ? Сколько /29 подсетей можно получить из сети с маской /24. Приведите несколько примеров /29 подсетей внутри сети 10.10.10.0/24.

```shell
$ ipcalc -b 10.10.10.0/29
Address:   10.10.10.0
Netmask:   255.255.255.248 = 29
Wildcard:  0.0.0.7
=>
Network:   10.10.10.0/29
HostMin:   10.10.10.1
HostMax:   10.10.10.6
Broadcast: 10.10.10.7
Hosts/Net: 6                     Class A, Private Internet
```
8 адресов = 6 для хостов, 1 адрес сети и 1 широковещательный адрес.

Сеть с маской /24 можно разбить на 32 подсети с маской /29

#### 6. Задача: вас попросили организовать стык между 2-мя организациями. Диапазоны 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 уже заняты. Из какой подсети допустимо взять частные IP адреса? Маску выберите из расчета максимум 40-50 хостов внутри подсети.

- Можно взять адреса из сети для CGNAT - 100.64.0.0/10.
```shell
$ ipcalc -b 100.64.0.0/10 -s 50
Address:   100.64.0.0
Netmask:   255.192.0.0 = 10
Wildcard:  0.63.255.255
=>
Network:   100.64.0.0/10
HostMin:   100.64.0.1
HostMax:   100.127.255.254
Broadcast: 100.127.255.255
Hosts/Net: 4194302               Class A

1. Requested size: 50 hosts
Netmask:   255.255.255.192 = 26
Network:   100.64.0.0/26
HostMin:   100.64.0.1
HostMax:   100.64.0.62
Broadcast: 100.64.0.63
Hosts/Net: 62                    Class A
```
- Маска для диапазонов будет /26, она позволит подключить 62 хоста.

#### 7. Как проверить ARP таблицу в Linux, Windows? Как очистить ARP кеш полностью? Как из ARP таблицы удалить только один нужный IP?

Проверить таблицу можно так:

- Linux: `ip neigh`, `arp -n`
- Windows: `arp -a`

Очистить кеш так:

- Linux: `ip neigh flush`
- Windows: `arp -d *`

Удалить один IP так:

- Linux: `ip neigh delete <IP> dev <INTERFACE>`, `arp -d <IP>`
- Windows: `arp -d <IP>`



</details>


### Домашнее задание к занятию "3.6. Компьютерные сети, лекция 1"  

<details>

#### 1. Работа c HTTP через телнет.
- Подключитесь утилитой телнет к сайту stackoverflow.com telnet stackoverflow.com 80
- отправьте HTTP запрос
```shell
GET /questions HTTP/1.0
HOST: stackoverflow.com
[press enter]
[press enter]
```
- В ответе укажите полученный HTTP код, что он означает?  

```shell
$ telnet stackoverflow.com 80
Trying 151.101.129.69...
Connected to stackoverflow.com.
Escape character is '^]'.
GET /questions HTTP/1.0
HOST: stackoverflow.com

HTTP/1.1 301 Moved Permanently
cache-control: no-cache, no-store, must-revalidate
location: https://stackoverflow.com/questions
x-request-guid: 6b722bf1-5548-4c23-8783-e7dd3eac8a43
feature-policy: microphone 'none'; speaker 'none'
content-security-policy: upgrade-insecure-requests; frame-ancestors 'self' https://stackexchange.com
Accept-Ranges: bytes
Date: Fri, 26 Nov 2021 15:19:58 GMT
Via: 1.1 varnish
Connection: close
X-Served-By: cache-fra19171-FRA
X-Cache: MISS
X-Cache-Hits: 0
X-Timer: S1637939998.255844,VS0,VE92
Vary: Fastly-SSL
X-DNS-Prefetch-Control: off
Set-Cookie: prov=3b058312-43f7-622e-8757-90187bed30db; domain=.stackoverflow.com; expires=Fri, 01-Jan-2055 00:00:00 GMT; path=/; HttpOnly

Connection closed by foreign host.
```
В ответ получили код 301 - редирект с HTTP на HTTPS протокол того же url

#### 2. Повторите задание 1 в браузере, используя консоль разработчика F12.
- откройте вкладку Network
- отправьте запрос http://stackoverflow.com
- найдите первый ответ HTTP сервера, откройте вкладку Headers
- укажите в ответе полученный HTTP код.
- проверьте время загрузки страницы, какой запрос обрабатывался дольше всего?
- приложите скриншот консоли браузера в ответ.  

![](pic/307.png)

В ответ получили код 307 (Temporary Redirect)

![](pic/time.png)

Страница полностью загрузилась за 2.24 сек. Самый долгий запрос - начальная загрузка страницы 353 мс

![](pic/timing.png)

#### 3. Какой IP адрес у вас в интернете?

```shell
$ dig @resolver4.opendns.com myip.opendns.com +short
82.102.xxx.xxx
```

#### 4. Какому провайдеру принадлежит ваш IP адрес? Какой автономной системе AS? Воспользуйтесь утилитой whois

```shell
$ whois 82.102.xxx.xxx | grep ^descr
descr:          PrimeTel PLC
```
IP адрес принадлежит ISP PrimeTel PLC
```shell
$ whois 82.102.xxx.xxx | grep ^origin
origin:         AS8544
```
AS - AS206912

#### 5. Через какие сети проходит пакет, отправленный с вашего компьютера на адрес 8.8.8.8? Через какие AS? Воспользуйтесь утилитой traceroute

```shell
$ traceroute -An 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  10.0.2.2 [*]  0.580 ms  0.453 ms  0.537 ms
 2  172.20.10.1 [*]  25.259 ms  25.239 ms  29.771 ms
 3  * * *
 4  10.95.130.98 [*]  109.653 ms  109.633 ms  109.614 ms
 5  10.95.130.209 [*]  109.594 ms  109.573 ms  109.552 ms
 6  78.158.134.114 [AS8544]  119.005 ms  107.965 ms  107.917 ms
 7  78.158.141.157 [AS8544]  187.849 ms  124.968 ms  124.899 ms
 8  78.158.141.141 [AS8544]  124.369 ms  124.188 ms  123.989 ms
 9  * * *
10  8.8.8.8 [AS15169]  123.991 ms  123.943 ms  123.758 ms
```
Пакет проходит через AS - AS8544, AS15169

```shell
$ grep org-name <(whois AS8544)
org-name:       Primetel PLC
$ grep OrgName <(whois AS15169)
OrgName:        Google LLC
```

#### 6. Повторите задание 5 в утилите mtr. На каком участке наибольшая задержка - delay?

```shell
$ mtr 8.8.8.8 -znrc 1
Start: 2021-11-26T15:17:47+0000
HOST: vagrant                     Loss%   Snt   Last   Avg  Best  Wrst StDev
  1. AS???    10.0.2.2             0.0%     1    0.4   0.4   0.4   0.4   0.0
  2. AS???    172.20.10.1          0.0%     1    5.7   5.7   5.7   5.7   0.0
  3. AS???    ???                 100.0     1    0.0   0.0   0.0   0.0   0.0
  4. AS???    10.95.130.98         0.0%     1   29.2  29.2  29.2  29.2   0.0
  5. AS???    10.95.130.209        0.0%     1   43.0  43.0  43.0  43.0   0.0
  6. AS8544   78.158.134.114       0.0%     1   48.0  48.0  48.0  48.0   0.0
  7. AS8544   78.158.141.157       0.0%     1  121.2 121.2 121.2 121.2   0.0
  8. AS8544   78.158.141.141       0.0%     1  110.6 110.6 110.6 110.6   0.0
  9. AS15169  108.170.236.175      0.0%     1   97.4  97.4  97.4  97.4   0.0
 10. AS15169  142.250.229.59       0.0%     1   88.4  88.4  88.4  88.4   0.0
 11. AS15169  8.8.8.8              0.0%     1   94.6  94.6  94.6  94.6   0.0
```
Наибольшая задержка на 7 хопе

#### 7. Какие DNS сервера отвечают за доменное имя dns.google? Какие A записи? воспользуйтесь утилитой dig

```shell
$ dig +short NS dns.google
ns4.zdns.google.
ns2.zdns.google.
ns1.zdns.google.
ns3.zdns.google.
```
NS записи

```shell
$ dig +short A dns.google
8.8.4.4
8.8.8.8
```
A записи

#### 8. Проверьте PTR записи для IP адресов из задания 7. Какое доменное имя привязано к IP? воспользуйтесь утилитой dig

```shell
$ for ip in `dig +short A dns.google`; do dig -x $ip | grep ^[0-9].*in-addr; done
8.8.8.8.in-addr.arpa.	18561	IN	PTR	dns.google.
4.4.8.8.in-addr.arpa.	21274	IN	PTR	dns.google.
```
dns.google

</details>

### Домашнее задание к занятию "3.5. Файловые системы"

<details>

#### 1. Узнайте о [sparse](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB) (разряженных) файлах.

Файлы с пустотами на диске. Разреженный файл эффективен, потому что он не хранит нули на диске, вместо этого он содержит достаточно метаданных, описывающих нули, которые будут сгенерированы. Разрежённые файлы используются для хранения, например, контейнеров.

Обычный файл заполненный нулями
```shell
$ dd if=/dev/zero of=output1 bs=1G count=4
$ stat output1
File: ouput1
  Size: 4294967296      Blocks: 8388616    IO Block: 4096   regular file
```
Разреженный файл заполненный нулями
```shell
$ dd if=/dev/zero of=output2 bs=1G seek=0 count=0
$ stat output2
File: output2
  Size: 4294967296      Blocks: 0          IO Block: 4096   regular file
```

#### 2. Могут ли файлы, являющиеся жесткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?

Нет, не могут, т.к. это просто ссылки на один и тот же inode - в нём и хранятся права доступа и имя владельца.

#### 3. Сделайте `vagrant destroy` на имеющийся инстанс Ubuntu. Замените содержимое Vagrantfile следующим:

```shell
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.provider :virtualbox do |vb|
    lvm_experiments_disk0_path = "/tmp/lvm_experiments_disk0.vmdk"
    lvm_experiments_disk1_path = "/tmp/lvm_experiments_disk1.vmdk"
    vb.customize ['createmedium', '--filename', lvm_experiments_disk0_path, '--size', 2560]
    vb.customize ['createmedium', '--filename', lvm_experiments_disk1_path, '--size', 2560]
    vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', lvm_experiments_disk0_path]
    vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', lvm_experiments_disk1_path]
  end
end
```
Данная конфигурация создаст новую виртуальную машину с двумя дополнительными неразмеченными дисками по 2.5 Гб.

```shell
vagrant@vagrant:~$ lsblk
NAME                 MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                    8:0    0   64G  0 disk
├─sda1                 8:1    0  512M  0 part /boot/efi
├─sda2                 8:2    0    1K  0 part
└─sda5                 8:5    0 63.5G  0 part
  ├─vgvagrant-root   253:0    0 62.6G  0 lvm  /
  └─vgvagrant-swap_1 253:1    0  980M  0 lvm  [SWAP]
sdb                    8:16   0  2.5G  0 disk
sdc                    8:32   0  2.5G  0 disk
```

#### 4. Используя `fdisk`, разбейте первый диск на 2 раздела: 2 Гб, оставшееся пространство.

```shell
vagrant@vagrant:~$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xa3918d14.

Command (m for help): F
Unpartitioned space /dev/sdb: 2.51 GiB, 2683305984 bytes, 5240832 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes

Start     End Sectors  Size
 2048 5242879 5240832  2.5G

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-5242879, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-5242879, default 5242879): +2G

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (4196352-5242879, default 4196352):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4196352-5242879, default 5242879):

Created a new partition 2 of type 'Linux' and of size 511 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

#### 5. Используя `sfdisk`, перенесите данную таблицу разделов на второй диск.

```shell
vagrant@vagrant:~$ sudo sfdisk -d /dev/sdb > sdb.dump
vagrant@vagrant:~$ sudo sfdisk /dev/sdc < sdb.dump
Checking that no-one is using this disk right now ... OK

Disk /dev/sdc: 2.51 GiB, 2684354560 bytes, 5242880 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Created a new DOS disklabel with disk identifier 0xa3918d14.
/dev/sdc1: Created a new partition 1 of type 'Linux' and of size 2 GiB.
/dev/sdc2: Created a new partition 2 of type 'Linux' and of size 511 MiB.
/dev/sdc3: Done.

New situation:
Disklabel type: dos
Disk identifier: 0xa3918d14

Device     Boot   Start     End Sectors  Size Id Type
/dev/sdc1          2048 4196351 4194304    2G 83 Linux
/dev/sdc2       4196352 5242879 1046528  511M 83 Linux

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

#### 6. Соберите `mdadm` RAID1 на паре разделов 2 Гб.

```shell
root@vagrant:~# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sd[bc]1
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

#### 7. Соберите `mdadm` RAID0 на второй паре маленьких разделов.

```shell
root@vagrant:~# mdadm --create /dev/md1 --level=0 --raid-devices=2 /dev/sd[bc]2
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
```

#### 8. Создайте 2 независимых PV на получившихся md-устройствах.

```shell
root@vagrant:~# pvcreate /dev/md0
  Physical volume "/dev/md0" successfully created.
root@vagrant:~# pvcreate /dev/md1
  Physical volume "/dev/md1" successfully created.
```

#### 9. Создайте общую volume-group на этих двух PV.

```shell
root@vagrant:~# vgcreate netology /dev/md0 /dev/md1
  Volume group "netology" successfully created
```

```shell
root@vagrant:~# vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  netology    2   0   0 wz--n-  <2.99g <2.99g
  vgvagrant   1   2   0 wz--n- <63.50g     0
```

#### 10. Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.

```shell
root@vagrant:~# lvcreate -L 100m -n netology-lv netology /dev/md1
  Logical volume "netology-lv" created.
root@vagrant:~# lvs -o +devices
  LV          VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices
  netology-lv netology  -wi-a----- 100.00m                                                     /dev/md1(0)
  root        vgvagrant -wi-ao---- <62.54g                                                     /dev/sda5(0)
  swap_1      vgvagrant -wi-ao---- 980.00m                                                     /dev/sda5(16010)
```

#### 11. Создайте `mkfs.ext4` ФС на получившемся LV.

```shell
oot@vagrant:~# mkfs.ext4 -L netology-ext4 -m 1 /dev/mapper/netology-netology--lv
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 25600 4k blocks and 25600 inodes

Allocating group tables: done
Writing inode tables: done
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done
root@vagrant:~# blkid | grep netology-netology--lv
/dev/mapper/netology-netology--lv: LABEL="netology-ext4" UUID="e8724683-3555-4082-b27e-939ae1d519de" TYPE="ext4"
```

#### 12. Смонтируйте этот раздел в любую директорию, например, `/tmp/new`.

```shell
root@vagrant:~# mkdir /tmp/new
root@vagrant:~# mount /dev/mapper/netology-netology--lv /tmp/new/
root@vagrant:~# mount | grep netology-netology--lv
/dev/mapper/netology-netology--lv on /tmp/new type ext4 (rw,relatime,stripe=256)
```

#### 13. Поместите туда тестовый файл, например `wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz`.

```shell
root@vagrant:~# cd /tmp/new/
root@vagrant:/tmp/new# wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz
--2021-11-22 19:16:22--  https://mirror.yandex.ru/ubuntu/ls-lR.gz
Resolving mirror.yandex.ru (mirror.yandex.ru)... 213.180.204.183, 2a02:6b8::183
Connecting to mirror.yandex.ru (mirror.yandex.ru)|213.180.204.183|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 22478375 (21M) [application/octet-stream]
Saving to: ‘/tmp/new/test.gz’

/tmp/new/test.gz                              100%[===================================>]  21.44M  2.30MB/s    in 8.0s

2021-11-22 19:16:30 (2.67 MB/s) - ‘/tmp/new/test.gz’ saved [22478375/22478375]
```

#### 14. Прикрепите вывод `lsblk`.

```shell
root@vagrant:/tmp/new# lsblk
NAME                        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda                           8:0    0   64G  0 disk
├─sda1                        8:1    0  512M  0 part  /boot/efi
├─sda2                        8:2    0    1K  0 part
└─sda5                        8:5    0 63.5G  0 part
  ├─vgvagrant-root          253:0    0 62.6G  0 lvm   /
  └─vgvagrant-swap_1        253:1    0  980M  0 lvm   [SWAP]
sdb                           8:16   0  2.5G  0 disk
├─sdb1                        8:17   0    2G  0 part
│ └─md0                       9:0    0    2G  0 raid1
└─sdb2                        8:18   0  511M  0 part
  └─md1                       9:1    0 1018M  0 raid0
    └─netology-netology--lv 253:2    0  100M  0 lvm   /tmp/new
sdc                           8:32   0  2.5G  0 disk
├─sdc1                        8:33   0    2G  0 part
│ └─md0                       9:0    0    2G  0 raid1
└─sdc2                        8:34   0  511M  0 part
  └─md1                       9:1    0 1018M  0 raid0
    └─netology-netology--lv 253:2    0  100M  0 lvm   /tmp/new
```

#### 15. Протестируйте целостность файла:

````shell
root@vagrant:/tmp/new# gzip -t /tmp/new/test.gz
root@vagrant:/tmp/new# echo $?
0
````
#### 16. Используя pvmove, переместите содержимое PV с RAID0 на RAID1.

```shell
root@vagrant:/tmp/new# pvmove -n netology-lv /dev/md1 /dev/md0
  /dev/md1: Moved: 28.00%
  /dev/md1: Moved: 100.00%
root@vagrant:/tmp/new# lvs -o +devices
  LV          VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Devices
  netology-lv netology  -wi-ao---- 100.00m                                                     /dev/md0(0)
  root        vgvagrant -wi-ao---- <62.54g                                                     /dev/sda5(0)
  swap_1      vgvagrant -wi-ao---- 980.00m                                                     /dev/sda5(16010)
```

#### 17. Сделайте `--fail` на устройство в вашем RAID1 md.

```shell
root@vagrant:/tmp/new# mdadm --fail /dev/md0 /dev/sdb1
mdadm: set /dev/sdb1 faulty in /dev/md0
```

#### 18. Подтвердите выводом `dmesg`, что RAID1 работает в деградированном состоянии.

```shell
root@vagrant:/tmp/new# dmesg | grep md0 | tail -n 2
[ 2636.150864] md/raid1:md0: Disk failure on sdb1, disabling device.
               md/raid1:md0: Operation continuing on 1 devices.
```

#### 19. Протестируйте целостность файла, несмотря на "сбойный" диск он должен продолжать быть доступен:
```shell
root@vagrant:/tmp/new# gzip -t /tmp/new/test.gz
root@vagrant:/tmp/new# echo $?
0
```
#### 20. Погасите тестовый хост, vagrant destroy.

```shell
root@vagrant:/tmp/new# exit
logout
vagrant@vagrant:~$ exit
logout
Connection to 127.0.0.1 closed.
user@User-MacBook-Pro Vagrant % vagrant destroy
    default: Are you sure you want to destroy the 'default' VM? [y/N] y
==> default: Forcing shutdown of VM...
==> default: Destroying VM and associated drives...
```

</details>

### Домашнее задание к занятию "3.4. Операционные системы, лекция 2"  

<details>

#### 1. На лекции мы познакомились с node_exporter. В демонстрации его исполняемый файл запускался в background. Этого достаточно для демо, но не для настоящей production-системы, где процессы должны находиться под внешним управлением. Используя знания из лекции по systemd, создайте самостоятельно простой unit-файл для node_exporter: 

- поместите его в автозагрузку,
- предусмотрите возможность добавления опций к запускаемому процессу через внешний файл (посмотрите, например, на `systemctl cat cron`),
- удостоверьтесь, что с помощью systemctl процесс корректно стартует, завершается, а после перезагрузки автоматически поднимается.

```shell
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.0/node_exporter-1.3.0.linux-amd64.tar.gz
tar xzf node_exporter-1.3.0.linux-amd64.tar.gz
rm -f node_exporter-1.3.0.linux-amd64.tar.gz
rm -r node_exporter-1.3.0.linux-amd64
sudo touch opt/node_exporter.env
echo "EXTRA_OPTS=\"--log.level=info\"" | sudo tee opt/node_exporter.env
sudo mv node_exporter-1.3.0.linux-amd64/node_exporter /usr/local/bin/
```
```shell
sudo tee /etc/systemd/system/node_exporter.service<<EOF
[Unit]
Description=Node Exporter
After=network.target
 
[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter $EXTRA_OPTS
StandardOutput=file:/var/log/node_explorer.log
StandardError=file:/var/log/node_explorer.log
 
[Install]
WantedBy=multi-user.target
EOF
```

```shell
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

добавление опций к запускаемому процессу через внешний файл
```shell
echo "EXTRA_OPTS=\"--log.level=info\"" | sudo tee opt/node_exporter.env
```
```shell
$ journalctl -u node_exporter.service
-- Logs begin at Sat 2021-11-20 21:06:21 UTC, end at Sat 2021-11-20 21:27:25 UTC. --
Nov 20 21:16:13 vagrant systemd[1]: Started Node Exporter.
Nov 20 21:17:38 vagrant systemd[1]: Stopping Node Exporter...
Nov 20 21:17:38 vagrant systemd[1]: node_exporter.service: Succeeded.
Nov 20 21:17:38 vagrant systemd[1]: Stopped Node Exporter.
Nov 20 21:27:12 vagrant systemd[1]: Started Node Exporter.
-- Reboot --
Nov 20 21:31:22 vagrant systemd[1]: Started Node Exporter.
```

#### 2. Ознакомьтесь с опциями node_exporter и выводом `/metrics` по-умолчанию. Приведите несколько опций, которые вы бы выбрали для базового мониторинга хоста по CPU, памяти, диску и сети.

CPU: system, user покажут время, использованное системой и программами; слишком высокий steal будет означать, что гипервизор перегружен и процессор занят другими ВМ; iowait - поможет отследить, всё ли в порядке с дисковой системой.  
```shell
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 27.36
node_cpu_seconds_total{cpu="0",mode="iowait"} 0.52
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0
node_cpu_seconds_total{cpu="0",mode="softirq"} 0.17
node_cpu_seconds_total{cpu="0",mode="steal"} 0
node_cpu_seconds_total{cpu="0",mode="system"} 3.47
node_cpu_seconds_total{cpu="0",mode="user"} 2.96
node_cpu_seconds_total{cpu="1",mode="idle"} 28.92
node_cpu_seconds_total{cpu="1",mode="iowait"} 0.2
node_cpu_seconds_total{cpu="1",mode="irq"} 0
node_cpu_seconds_total{cpu="1",mode="nice"} 0
node_cpu_seconds_total{cpu="1",mode="softirq"} 0.21
node_cpu_seconds_total{cpu="1",mode="steal"} 0
node_cpu_seconds_total{cpu="1",mode="system"} 2.66
node_cpu_seconds_total{cpu="1",mode="user"} 2.34
```
MEM: MemTotal - количество памяти; MemFree и MemAvailable - свободная и доступная память; SwapTotal, SwapFree, SwapCached - своп, если слишком много занято -- RAM не хватает.

```shell
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 7.43829504e+08
# TYPE node_memory_MemFree_bytes gauge
node_memory_MemFree_bytes 6.51558912e+08
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 1.028694016e+09
# TYPE node_memory_SwapCached_bytes gauge
node_memory_SwapCached_bytes 0
# TYPE node_memory_SwapFree_bytes gauge
node_memory_SwapFree_bytes 1.027600384e+09
# TYPE node_memory_SwapTotal_bytes gauge
node_memory_SwapTotal_bytes 1.027600384e+09
```

DISK: size_bytes и avail_bytes покажут объём и свободное место; readonly=1 может говорить о проблемах ФС, из-за чего она перешла в режим только для чтения; io_now - интенсивность работы с диском в текущий момент.
```shell
# TYPE node_filesystem_avail_bytes gauge
node_filesystem_avail_bytes{device="/dev/mapper/vgvagrant-root",fstype="ext4",mountpoint="/"} 6.0764639232e+10
# TYPE node_filesystem_readonly gauge
node_filesystem_readonly{device="/dev/mapper/vgvagrant-root",fstype="ext4",mountpoint="/"} 0
# TYPE node_filesystem_size_bytes gauge
node_filesystem_size_bytes{device="/dev/mapper/vgvagrant-root",fstype="ext4",mountpoint="/"} 6.5827115008e+10
# TYPE node_disk_io_now gauge
node_disk_io_now{device="sda"} 0
```

NET: carrier_down, carrier_up - если много, значит проблема с физическим подключением; info - общая информация по интерфейсу; mtu_bytes - может быть важно для диагностики потерь или если трафик хостов не проходит через маршрутизатор; receive_errs_total, transmit_errs_total, receive_packets_total, transmit_packets_total - ошибки передачи, в зависимости от объёма, вероятно какие-то проблемы сети или с хостом
```shell
# TYPE node_network_carrier_down_changes_total counter
node_network_carrier_down_changes_total{device="eth0"} 1
# TYPE node_network_carrier_up_changes_total counter
node_network_carrier_up_changes_total{device="eth0"} 1
# TYPE node_network_info gauge
node_network_info{address="08:00:27:73:60:cf",broadcast="ff:ff:ff:ff:ff:ff",device="eth0",duplex="full",ifalias="",operstate="up"} 1
# TYPE node_network_mtu_bytes gauge
node_network_mtu_bytes{device="eth0"} 1500
# TYPE node_network_receive_errs_total counter
node_network_receive_errs_total{device="eth0"} 0
# TYPE node_network_receive_packets_total counter
node_network_receive_packets_total{device="eth0"} 351
# TYPE node_network_transmit_errs_total counter
node_network_transmit_errs_total{device="eth0"} 0
# TYPE node_network_transmit_packets_total counter
node_network_transmit_packets_total{device="eth0"} 279
```

#### 3. Установите в свою виртуальную машину Netdata. Воспользуйтесь готовыми пакетами для установки (`sudo apt install -y netdata`). После успешной установки:

- в конфигурационном файле `/etc/netdata/netdata.conf` в секции [web] замените значение с localhost на `bind to = 0.0.0.0`,
```shell
$ sudo nano /etc/netdata/netdata.conf
$ grep -e bind -e web /etc/netdata/netdata.conf
	web files owner = root
	web files group = root
	# bind socket to IP = 127.0.0.1
	bind to = 0.0.0.0
```
- добавьте в Vagrantfile проброс порта Netdata на свой локальный компьютер и сделайте vagrant reload:  
```shell
config.vm.network "forwarded_port", guest: 19999, host: 19999
```
```shell
% sudo nano vagrantfile
% vagrant reload
% vagrant port
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2222 (host)
 19999 (guest) => 19999 (host)
  9100 (guest) => 9100 (host)
```

После успешной перезагрузки в браузере на своем ПК (не в виртуальной машине) вы должны суметь зайти на `localhost:19999`. Ознакомьтесь с метриками, которые по умолчанию собираются Netdata и с комментариями, которые даны к этим метрикам.  

http://localhost:19999

pic![](https://i.ibb.co/rbDG8nR/Screenshot-2021-11-21-at-11-11-04.png)

#### 4. Можно ли по выводу `dmesg` понять, осознает ли ОС, что загружена не на настоящем оборудовании, а на системе виртуализации?

```shell
$ dmesg | grep -i 'Hypervisor detected'
[    0.000000] Hypervisor detected: KVM
```

#### 5. Как настроен sysctl `fs.nr_open` на системе по-умолчанию? Узнайте, что означает этот параметр. Какой другой существующий лимит не позволит достичь такого числа (`ulimit --help`)?

```shell
$ sysctl fs.nr_open
fs.nr_open = 1048576
```
fs.nr_open - жесткий лимит на открытые дескрипторы для ядра (системы)
```shell
$ ulimit -Sn
1024
```
Soft limit на пользователя, может быть изменен как большую, так и меньшую сторону  
```shell
$ ulimit -Hn
1048576
```
Hard limit на пользователя, может быть изменен только в меньшую сторону  
Оба `ulimit` -n не могут превышать `fs.nr_open`

#### 6. Запустите любой долгоживущий процесс (не `ls`, который отработает мгновенно, а, например, `sleep 1h`) в отдельном неймспейсе процессов; покажите, что ваш процесс работает под PID 1 через `nsenter`. Для простоты работайте в данном задании под root (`sudo -i`). Под обычным пользователем требуются дополнительные опции (`--map-root-user`) и т.д.

Terminal #1
```shell
vagrant@vagrant:~$ sudo unshare -f --pid --mount-proc sleep 1h
```
Terminal #2
```shell
vagrant@vagrant:~$ ps -e | grep sleep
   1789 pts/0    00:00:00 sleep
vagrant@vagrant:~$ sudo nsenter --target 1789 --mount --uts --ipc --net --pid ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   9828   580 pts/0    S+   09:34   0:00 sleep 1h
root           6  0.0  0.3  13216  3336 pts/1    R+   09:39   0:00 ps aux
```

#### 7. Найдите информацию о том, что такое `:(){ :|:& };:`. Запустите эту команду в своей виртуальной машине Vagrant с Ubuntu 20.04 (**это важно, поведение в других ОС не проверялось**). Некоторое время все будет "плохо", после чего (минуты) – ОС должна стабилизироваться. Вызов `dmesg` расскажет, какой механизм помог автоматической стабилизации. Как настроен этот механизм по-умолчанию, и как изменить число процессов, которое можно создать в сессии?

Это fork bomb, бесконечно создающая свои копии (системным вызовом fork())

Стабилизация системы:
```shell
[ 1538.730411] cgroup: fork rejected by pids controller in /user.slice/user-1000.slice/session-3.scope
```

Значение TasksMax (изменение значения в %, конкретное число или infinity, чтобы убрать лимит) в /usr/lib/systemd/system/user-.slice.d/10-defaults.conf регулирует число процессов, которое можно создать в сессии

</details>


### Домашнее задание к занятию "3.3. Операционные системы, лекция 1"  

<details>

#### 1. Какой системный вызов делает команда `cd`? 
```shell
chdir()
```
#### 2. Попробуйте использовать команду `file` на объекты разных типов на файловой системе. Например:  
```shell
vagrant@netology1:~$ file /dev/tty
/dev/tty: character special (5/0)
vagrant@netology1:~$ file /dev/sda
/dev/sda: block special (8/0)
vagrant@netology1:~$ file /bin/bash
/bin/bash: ELF 64-bit LSB shared object, x86-64
```
Используя `strace` выясните, где находится база данных `file` на основании которой она делает свои догадки.  

```shell
openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3
```

#### 3. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).  

1. По pid процесса, который держит файл, выяснить его дескрипторы.
2. В дескриптор отправить пустую строку, например, `echo '' > /proc/123456/fd/3`

#### 4. Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?  

Нет. Когда процесс завершается через `exit`, вся память и связанные с ним ресурсы освобождаются, чтобы их могли использовать другие процессы.  

#### 5. В iovisor BCC есть утилита `opensnoop`:
```shell
root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
/usr/sbin/opensnoop-bpfcc
```
#### На какие файлы вы увидели вызовы группы `open` за первую секунду работы утилиты? Воспользуйтесь пакетом bpfcc-tools для Ubuntu 20.04.  

```shell
$ sudo /usr/sbin/opensnoop-bpfcc
PID    COMM               FD ERR PATH
777    vminfo              4   0 /var/run/utmp
584    dbus-daemon        -1   2 /usr/local/share/dbus-1/system-services
584    dbus-daemon        18   0 /usr/share/dbus-1/system-services
584    dbus-daemon        -1   2 /lib/dbus-1/system-services
584    dbus-daemon        18   0 /var/lib/snapd/dbus-1/system-services/
```

#### 6. Какой системный вызов использует `uname -a`? Приведите цитату из `man` по этому системному вызову, где описывается альтернативное местоположение в `/proc`, где можно узнать версию ядра и релиз ОС.  
`uname()`  
`man uname(2) line 65:`

`Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version, domainname}.`

#### 7. Чем отличается последовательность команд через `;` и через `&&` в `bash`? Например:  

```shell
root@netology1:~# test -d /tmp/some_dir; echo Hi
Hi
root@netology1:~# test -d /tmp/some_dir && echo Hi
root@netology1:~#
```
#### Есть ли смысл использовать в bash &&, если применить set -e?

Операторы последовательного выполнения команд.

- `;` выполнит все команды последовательно, даже если какая-то завершится ошибкой.
- `&&` остановится при завершении какой-то команды в последовательности ошибкой.

~~Скрипт с `set -e` не упадёт, если ошибкой завершится команда, выполненная в конструкции с оператором `&&`. Смысла использования `bash &&` + `set -e` не вижу.~~  
С параметром -e оболочка завершится только при ненулевом коде возврата команды. Если ошибочно завершится одна из команд, разделённых &&, то выхода из шелла не произойдёт. Так что, смысл есть.  

#### 8. Из каких опций состоит режим `bash set -euxo pipefail` и почему его хорошо было бы использовать в сценариях?  

- `-e` прерывает выполнение исполнения при ошибке любой команды кроме последней в последовательности; 
- `-x` вывод трейса простых команд;
- `-u` неустановленные/не заданные параметры и переменные считаются как ошибки, с выводом в stderr текста ошибки и выполнит завершение не интерактивного вызова;
- `-o pipefail` возвращает код возврата набора/последовательности команд, ненулевой при последней команды или 0 для успешного выполнения команд.

Повышает детализацию вывода ошибок и завершит сценарий при наличии ошибок, на любом этапе выполнения сценария, кроме последней завершающей команды.

#### 9. Используя `-o stat` для `ps`, определите, какой наиболее часто встречающийся статус у процессов в системе. В `man ps` ознакомьтесь (`/PROCESS STATE CODES`) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).  

```shell
$ ps -Ao stat  | sort | uniq -c | sort -h
      1 R+
      1 S<s
      1 SLsl
      1 STAT
      1 Sl
      1 Ss+
      2 SN
      2 T
      3 S+
      5 Ssl
      8 I
     15 Ss
     24 S
     40 I<
```

S - процессы спящие, находятся в режиме ожидания  
I - фоновые процессы ядра

</details>

### Домашнее задание к занятию "3.2. Работа в терминале, лекция 2"

<details>

#### 1. Какого типа команда cd?

```
$ type cd
cd is a shell builtin
```

#### 2. Какая альтернатива без pipe команде grep <some_string> <some_file> | wc -l?

```
wc -l < <(<some_string> <some_file>)
```

#### 3. Какой процесс с PID 1 является родителем для всех процессов в вашей виртуальной машине Ubuntu 20.04?  

```shell
$ pstree -a -p  | head -n 1
systemd,1
```
```shell
$ sudo ls -l /proc/1/exe
lrwxrwxrwx 1 root root 0 Nov 10 19:26 /proc/1/exe -> /usr/lib/systemd/systemd
```
```shell
$ ps -F 1
UID          PID    PPID  C    SZ   RSS PSR STIME TTY      STAT   TIME CMD
root           1       0  0 41832 11300   0 16:38 ?        Ss     0:01 /sbin/init
```
```shell
$ ls -lh /sbin/init
lrwxrwxrwx 1 root root 20 Jul 21 19:00 /sbin/init -> /lib/systemd/systemd
```

#### 4. Как будет выглядеть команда, которая перенаправит вывод stderr ls на другую сессию терминала?  

```shell
ls -l \root 2>/dev/pts/1
```

#### 5. Получится ли одновременно передать команде файл на stdin и вывести ее stdout в другой файл? Приведите работающий пример.  

```shell
sed 's/#/##/g' <~/test1 >test2
```

#### 6. Получится ли вывести находясь в графическом режиме данные из PTY в какой-либо из эмуляторов TTY? Сможете ли вы наблюдать выводимые данные?  

```shell
echo 'message' > /dev/tty1
```
Нужно быть авторизованным в этом терминале. Или под `root` без авторизации.

#### 7. Выполните команду bash 5>&1. К чему она приведет? Что будет, если вы выполните echo netology > /proc/$$/fd/5? Почему так происходит?  

`bash 5>&1` запустит экземпляр bash с fd 5 и перенаправит его на fd 1 (stdout).  

`echo netology > /proc/$$/fd/5` выведет в терминал слово "netology". Это произойдёт потому что echo отправляет netology в fd 5 текущего шелла (подсистема /proc содержит информацию о запущенных процессах по их PID, $$ - подставит PID текущего шелла)  

#### 8. Получится ли в качестве входного потока для pipe использовать только stderr команды, не потеряв при этом отображение stdout на pty?  

Да
```shell
cat ~/.bashrc dasdsfad 2>&1 1>/dev/pts/0 | sed 's/cat/test/g' > test;
```

#### 9. Что выведет команда cat /proc/$$/environ? Как еще можно получить аналогичный по содержанию вывод?  

Команда выведет набор переменных окружения.

Что-то похожее вернут команды `env` и `printenv`.

#### 10. Используя man, опишите что доступно по адресам /proc/<PID>/cmdline, /proc/<PID>/exe.  

`/proc/<PID>/cmdline` выведет команду, к которой относится , со всеми агрументами, разделёнными специальными символом '\x0' (это не пробел, cat файла выведёт всё "слипнувшимся")

`/proc/<PID>/exe` это симлинк на полный путь к исполняемому файлоу, из которого вызвана программа с этим пидом  

#### 11. Узнайте, какую наиболее старшую версию набора инструкций SSE поддерживает ваш процессор с помощью /proc/cpuinfo.  

```shell
$ cat /proc/cpuinfo  | grep -o 'sse[0-9_]*' | sort -h | uniq
sse
sse2
sse3
sse4_1
sse4_2
```
SSE 4.2  

#### 12. При открытии нового окна терминала и vagrant ssh создается новая сессия и выделяется pty. Однако ... not a tty ... Почитайте, почему так происходит, и как изменить поведение.  

```shell
$ ssh localhost 'tty'
vagrant@localhost's password:
not a tty
```

Это сделано для правильной работы в скриптах. Если сразу выполнить команду на удалённом сервере через ssh, sshd это поймёт, и запускаемые команды тоже, поэтому они не будут спрашивать что-то у пользователя, а вывод очистят от лишних данных.

Например, если в интерактивном режиме программа задала бы пользователю вопрос и ждала ответа "yes/no", при запуске через ssh она этого делать не станет.

Изменить поведение можно добавив флаг -t при вызове ssh. 

```shell
$ ssh -t localhost 'tty'
vagrant@localhost's password:
/dev/pts/1
Connection to localhost closed.
```


#### 13. Бывает, что есть необходимость переместить запущенный процесс из одной сессии в другую. Попробуйте сделать это, воспользовавшись reptyr.  

Работает по [инструкции](https://github.com/nelhage/reptyr#typical-usage-pattern) проекта reptyr.

#### 14. Узнайте что делает команда tee и почему в отличие от sudo echo команда с sudo tee будет работать.  

Команда `tee` делает вывод одновременно и в файл, указанный в качестве параметра, и в `stdout`. 
В данном примере команда получает вывод из `stdin`, перенаправленный через `pipe` от `stdout` команды `echo`
и т.к. команда запущена от `sudo`, соответственно имеет повышенные права на запись.

</details>

### Домашнее задание к занятию "3.1 Работа в терминале, лекция 1"  

<details>

#### 1. Установите средство виртуализации Oracle VirtualBox.

установлено

#### 2. Установите средство автоматизации Hashicorp Vagrant.

установлено

#### 3. В вашем основном окружении подготовьте удобный для дальнейшей работы терминал. Можно предложить:

установлен iTerm2

#### 4. С помощью базового файла конфигурации запустите Ubuntu 20.04 в VirtualBox посредством Vagrant:

выполнено

#### 5. Ознакомьтесь с графическим интерфейсом VirtualBox, посмотрите как выглядит виртуальная машина, которую создал для вас Vagrant, какие аппаратные ресурсы ей выделены. Какие ресурсы выделены по-умолчанию?  

RAM:1024mb  
CPU:2 cpu  
HDD:64gb  
video:4mb  

#### 6. Ознакомьтесь с возможностями конфигурации VirtualBox через Vagrantfile: документация. Как добавить оперативной памяти или ресурсов процессора виртуальной машине?  

Добавлением команд в VagrantFile  
короткие линки  
```
  v.memory = 2048  
  v.cpus = 4  
```
или командами ВМ  

   ```
   config.vm.provider "virtualbox" do |vb|  
     vb.memory = "2048"  
     vb.cpu = "24"  
   end  
   ```

#### 7. Команда vagrant ssh из директории, в которой содержится Vagrantfile, позволит вам оказаться внутри виртуальной машины без каких-либо дополнительных настроек. Попрактикуйтесь в выполнении обсуждаемых команд в терминале Ubuntu.

выполнено

#### 8. Ознакомиться с разделами man bash, почитать о настройках самого bash:

- какой переменной можно задать длину журнала history, и на какой строчке manual это описывается?  

Число строк журнала задаётся переменной окружения HISTFILESIZE, она описана со строки 1060  

Число команд задаётся переменой окружения HISTSIZE, она описана со строки 1081  

 - что делает директива ignoreboth в bash? 

Директива ignoreboth является сокращением для ignorespace и ignoredups.

Если список значений включает в себя ignorespace, строки, начинающиеся с символа пробела, не сохраняются в списке истории.  
Значение ignoredups приводит к тому, что строки, соответствующие предыдущей записи в истории, не сохраняются.

#### 9. В каких сценариях использования применимы скобки {} и на какой строчке man bash это описано?  

{} - зарезервированные слова, список, в т.ч. список команд, в отличие от () исполнятся в текущем инстансе, 
используется в различных условных циклах, условных операторах, или ограничивает тело функции. Статус возврата - это статус выхода из списка.  
Описана со строки 317 



#### 10. С учётом ответа на предыдущий вопрос, как создать однократным вызовом touch 100000 файлов? Получится ли аналогичным образом создать 300000? Если нет, то почему?  

100000 - да  
```
touch file{1..100000}
```  
300000 -  нет  
```touch file{1..300000}  
  -bash: /usr/bin/touch: Argument list too long
 ```

#### 11. В man bash поищите по `/\[\[`. Что делает конструкция `[[ -d /tmp ]]`  

Проверяет наличие каталога /tmp

#### 12. Основываясь на знаниях о просмотре текущих (например, PATH) и установке новых переменных; командах, которые мы рассматривали, добейтесь в выводе type -a bash в виртуальной машине наличия первым пунктом в списке:  

bash is /tmp/new_path_directory/bash  
bash is /usr/local/bin/bash  
bash is /bin/bash  

```
$ ln -s /usr/bin /tmp/new_path_directory  
$ PATH=/tmp/new_path_directory:${PATH}  
$ type -a bash  
bash is /tmp/new_path_directory/bash  
bash is /usr/bin/bash  
bash is /bin/bash  
```

#### 13. Чем отличается планирование команд с помощью batch и at?  

- at выполняется строго по расписанию  
- batch выполняется, когда позволит нагрузка на систему (load average упадёт ниже 1.5 или значения, заданного командой atd)  

#### 14. Завершите работу виртуальной машины чтобы не расходовать ресурсы компьютера и/или батарею ноутбука.  

```
% vagrant suspend
```
</details>

### Домашнее задание к занятию "2.4 Инструменты Git"

<details>

#### 1. Найдите полный хеш и комментарий коммита, хеш которого начинается на aefea.

Решение  
git show aefea

**Ответ**  
aefead2207ef7e2aa5dc81a34aedf0cad4c32545  
Update CHANGELOG.md

#### 2. Какому тегу соответствует коммит 85024d3?

Решение  
git show 85024

**Ответ**  
commit 85024d3100126de36331c6982bfaac02cdab9e76 (tag: v0.12.23)

#### 3. Сколько родителей у коммита b8d720? Напишите их хеши.

Решение  
git show --pretty=format:' %P' b8d720

**Ответ**  
56cd7859e05c36c06b56d013b55a252d0bb7e158  
9ea88f22fc6269854151c571162c5bcf958bee2b

#### 4. Перечислите хеши и комментарии всех коммитов которые были сделаны между тегами v0.12.23 и v0.12.24.

Решение  
git log  v0.12.23..v0.12.24  --oneline

**Ответ**  
33ff1c03b (tag: v0.12.24) v0.12.24  
b14b74c49 [Website] vmc provider links  
3f235065b Update CHANGELOG.md  
6ae64e247 registry: Fix panic when server is unreachable  
5c619ca1b website: Remove links to the getting started guide's old location  
06275647e Update CHANGELOG.md  
d5f9411f5 command: Fix bug when using terraform login on Windows  
4b6d06cc5 Update CHANGELOG.md  
dd01a3507 Update CHANGELOG.md  
225466bc3 Cleanup after v0.12.23 release  

#### 5. Найдите коммит в котором была создана функция func providerSource, ее определение в коде выглядит так func providerSource(...) (вместо троеточего перечислены аргументы).

Решение  
git log -S'func providerSource(' --oneline

**Ответ**  
8c928e835 main: Consult local directories as potential mirrors of providers

#### 6. Найдите все коммиты в которых была изменена функция globalPluginDirs.

Решение  
git grep 'func globalPluginDirs'  
git log -L :'func globalPluginDirs':plugins.go --oneline  

**Ответ**  
commit 8364383c3 - создана  
commit 66ebff90c - изменена  
commit 41ab0aef7 - изменена  
commit 52dbf9483 - изменена  
commit 78b122055 - изменена  

#### 7. Кто автор функции synchronizedWriters?

Решение  

git log -S'func synchronizedWriters(‘ --pretty=format:'%h - %an %ae'

bdfea50cc - James Bardin j.bardin@gmail.com  
5ac311e2a - Martin Atkins mart@degeneration.co.uk  

git show bdfea50cc - удалена  
git show 5ac311e2a - создана  

**Ответ**  
Author: Martin Atkins <mart@degeneration.co.uk>

</details>

### Домашнее задание к занятию "2.1 Системы контроля версий."

<details>

#### devops-netology  
Игнорировать все файлы в директории .terraform  
Игнорировать все файлы оканчивающиеся на .tfstate и содержащие .tfstate.  
Игнорировать файл crash.log  
Игнорировать файлы оканчивающиеся на .tfvars  
Игнорировать файлы override.tf, override.tf и оканчивающиеся на _override.tf и _override.tf.json  
Игнорировать файлы .terraformrc и terraform.rc  
Игнорировать каталог .idea

</details>