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

### [Домашнее задание к занятию "4.2. Использование Python для решения типовых DevOps задач"](https://github.com/Bora2k3/devops-netology/blob/main/04-script-02-py.md)
### [Домашнее задание к занятию "4.1. Командная оболочка Bash: Практические навыки"](https://github.com/Bora2k3/devops-netology/blob/main/04-script-01-bash.md)
### [Домашнее задание к занятию "3.9. Элементы безопасности информационных систем"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-09-security.md)
### [Домашнее задание к занятию "3.8. Компьютерные сети, лекция 3"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-08-net.md)
### [Домашнее задание к занятию "3.7. Компьютерные сети, лекция 2"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-07-net.md)
### [Домашнее задание к занятию "3.6. Компьютерные сети, лекция 1"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-06-net.md)
### [Домашнее задание к занятию "3.5. Файловые системы"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-05-fs.md)
### [Домашнее задание к занятию "3.4. Операционные системы, лекция 2"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-04-os.md)
### [Домашнее задание к занятию "3.3. Операционные системы, лекция 1"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-03-os.md)
### [Домашнее задание к занятию "3.2. Работа в терминале, лекция 2"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-02-terminal.md)
### [Домашнее задание к занятию "3.1 Работа в терминале, лекция 1"](https://github.com/Bora2k3/devops-netology/blob/main/03-sysadmin-01-terminal.md)
### [Домашнее задание к занятию "2.4 Инструменты Git"](https://github.com/Bora2k3/devops-netology/blob/main/02-git-04-tools.md)
### [Домашнее задание к занятию "2.1 Системы контроля версий."](https://github.com/Bora2k3/devops-netology/blob/main/02-git-01-vcs.md)
