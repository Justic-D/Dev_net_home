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