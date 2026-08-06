### Ceph — это программно-определяемое распределённое хранилище (SDS). 
Простыми словами: это система, которая превращает кучу обычных жёстких дисков на нескольких серверах в единое отказоустойчивое хранилище с огромным объёмом и высокой скоростью.

Представьте, что у вас 3 сервера(ноды), на каждом по 4 диска по 4 ТБ. Вместо того чтобы на каждом сервере отдельно создавать хранилища, Ceph объединяет все диски в одну виртуальную «ферму» объёмом ~48 ТБ, которая автоматически дублирует данные, чтобы при поломке любого диска или сервера ничего не потерялось.

> Зачем это нужно:

> - Отказоустойчивость — если сломается один сервер или диск, виртуальные машины продолжат работать на других серверах, потому что их данные сохранены в нескольких копиях.

> - Живая миграция — можно переносить работающие ВМ с одного физического сервера на другой без остановки, потому что хранилище общее для всего кластера.

> - Масштабируемость — хотите добавить ещё 10 ТБ? Просто воткните новый диск в любой сервер, добавьте его в Ceph — и объём хранилища увеличится автоматически.

> - Единое пространство — все серверы кластера видят одни и те же данные, не нужно думать «а на каком сервере лежит диск этой ВМ».

#### Как работает Ceph (упрощённо)
Архитектура Ceph состоит из трёх основных компонентов:

1. **OSD (Object Storage Daemon)** — ``«хранители данных»``<br>
Это демон, который работает на каждом диске (или группе дисков), который вы отдали Ceph.

Он отвечает за хранение данных, их репликацию (копирование) и восстановление в случае сбоев.

В моём случае на дисках ``sdc, sdd, sde, sdf`` работают OSD-демоны.

2. **MON (Monitor)** — ``«диспетчеры»``<br>
Это небольшие демоны, которые следят за состоянием всего кластера.

Они знают, какие OSD живы, какие диски заполнены, кто является «главным» для каждой копии данных.

В кластере должно быть нечётное количество мониторов (1, 3, 5) для принятия решений большинством голосов (кворум). У меня 1 монитор (на первой ноде), но в продакшене рекомендуется 3.

3. **MGR (Manager)** — ``«менеджеры»``<br>
Это демон, который собирает статистику и предоставляет интерфейс для управления (веб-дашборд, API, балансировка нагрузки).

#### Как данные попадают в Ceph и читаются оттуда?
##### Запись:

- Клиент (на Proxmox) отправляет запрос на запись данных.

- Монитор (MON) говорит клиенту: «Иди к OSD с номером 5, 12 и 23 — они сейчас свободны».

- Клиент пишет данные одновременно на три OSD (если настроена репликация 3) на разных серверах.

- Когда все три подтвердили запись, клиент получает ответ «Успешно».

##### Чтение:

- Клиент спрашивает монитор: «Где лежит объект с ID таким-то?»

- Монитор отвечает: «На OSD 5, 12 и 23. Но сейчас быстрее всего ответит OSD 12 — иди к нему».

- Клиент читает данные с ближайшего/быстрейшего OSD.

#### Типы данных в Ceph
Ceph умеет работать с тремя интерфейсами:

- **RADOS** — низкоуровневый объектный доступ (для разработчиков).

- **RBD (Block Device)** — предоставляет блочные устройства, которые используются как диски для ВМ. Это мой случай.

- **CephFS** — файловая система (как NFS, но распределённая).

- **RADOS Gateway (S3)** — объектное хранилище с API, совместимым с Amazon S3.

#### Аналогия для понимания
Представьте, что Ceph — это библиотека с автоматическим дублированием книг:

- **OSD** — это книжные полки (каждый диск — отдельная полка).

- **MON** — это библиотекарь, который знает, на какой полке лежит каждая книга и есть ли её копии на других полках.

- **MGR** — это система учёта, которая считает, сколько книг выдано, какие полки заполнены и строит отчёты.

### Разберём как подключить Ceph 

У меня есть два физических сервера (две ноды), по рекомендации (1,3,5), в лучшем варианте должно быть три, но у меня две ноды, не самый лучший вариант.

Для начала разметим правильно дисковое пространство, так как оно у меня не совсем правильно размечено. Выполним команду 
```bash
lsblk
```
<img width="1294" height="509" alt="image" src="https://github.com/user-attachments/assets/51ed0326-ca07-41df-bb0f-ac4aab78d482" />

Видим штатное отображение thin provisioning (thin pool) в LVM.

То, что вы видите — это не «дублирование» данных, а мета-устройства (**tmeta и tdata**), из которых состоит пул **pve-data-tpool**. Именно так LVM показывает внутреннее устройство thin-пула.

Вот что означают эти строки:

**pve-data_tmeta (253:2)** — это метаданные пула (где и какие блоки лежат).

**pve-data_tdata (253:3)** — это сами данные пула.

**pve-data-tpool (253:4)** — это объединение метаданных и данных в единый пул.

**pve-data (253:5)** — это «главный» том в этом пуле, через который Proxmox управляет тонкими дисками.

**pve-vm--100--disk--0** и **pve-vm--102--disk--0** — это виртуальные диски ВМ, которые лежат внутри этого пула (используют его пространство динамически).

##### Почему данные дублируются в выводе?
> Потому что lsblk показывает древовидную структуру:

> - tmeta и tdata являются компонентами tpool, а tpool — компонентом data. 

Поэтому lsblk честно рисует их и как части пула, и как родительские элементы для виртуальных дисков.

Чтобы убедиться, что всё в порядке с LVM, выполните:
```bash
pvs && vgs && lvs
```

<img width="1254" height="283" alt="image" src="https://github.com/user-attachments/assets/e622adc6-be7b-4e20-b589-45cc41a08db2" />

##### Текущее состояние ноды
Локальное хранилище (LVM-thin):

VG pve объединяет два физических тома: **/dev/sda2 (894 ГБ)** и **/dev/sdb (894 ГБ)**.

Однако тонкий пул data использует только 846.75 ГБ – это соответствует размеру /dev/sda2. Диск /dev/sdb не добавлен в тонкий пул, хотя в VG есть свободное место (795 ГБ).

Существующие тома ВМ и снапшоты занимают ~12% пула, свободно ~740 ГБ в самом пуле (но не в VG).

Неиспользуемые диски:

**sdc, sdd, sde, sdf** – по 3.6 ТБ, полностью свободны (нет разделов, не в LVM).

И есть задача расширить корневой диск.

##### 1. Расширяем корневой диск с подключением не использованным пространством диска sdb добавляем к корневому 100Гб
```bash
vgextend pve /dev/sdb
lvextend -L +100G /dev/pve/root
resize2fs /dev/mapper/pve-root
```

<img width="1427" height="197" alt="image" src="https://github.com/user-attachments/assets/95ff53de-849c-49b3-b380-c042eafb38cb" />

##### 2. Расширяем локальное LVM-thin хранилище за счёт /dev/sdb
###### Расширяем тонкий пул data на всё свободное место в VG

<img width="1423" height="87" alt="image" src="https://github.com/user-attachments/assets/eef54bc9-b79c-4d67-97b7-de631d60605b" />

```bash
lvextend -l +100%FREE pve/data
```
###### Проверяем новый размер
```bash
lvs -a -o +devices,size,data_percent,metadata_percent pve/data
```

<img width="1427" height="175" alt="image" src="https://github.com/user-attachments/assets/00bd90a8-1fe0-42ef-9307-2b5f880735a1" />

***

##### 3. Теперь когда всё привели в порядок, настраиваем Ceph на этой первой ноде (как отдельный кластер или часть общего)
**4 диска по 3.6 ТБ** – идеально для создания **OSD**. Если вы хотите использовать Ceph для отказоустойчивого хранилища

<img width="1424" height="701" alt="07" src="https://github.com/user-attachments/assets/f132146e-07cf-49a5-88cc-24d9206cb0f2" />

Для начала перейдём на вкладку **Datacenter -> Ceph -> configure Ceph** и нажимаем на эту кнопку

<img width="1386" height="671" alt="image" src="https://github.com/user-attachments/assets/10813815-42ed-41ad-8de2-a39f8d6fa027" />

Далее выбираем сетевые настройки **(CIDR)** и ок

<img width="1392" height="699" alt="image" src="https://github.com/user-attachments/assets/8373607b-8d09-4231-a448-083995202229" />

<img width="1391" height="699" alt="image" src="https://github.com/user-attachments/assets/3f51c526-7d83-4de9-9ba0-3eef1b928b83" />

***

> - Теперь переходим в Shell на первой ноде

###### Если через графический интерфейс не создали монитор, то создайте монитор (первый на этой ноде)
```bash
pveceph mon create
```
###### Проверьте, что кластер создался (теперь должна быть информация, хоть и с предупреждениями)
```bash
ceph -s   
```
###### Если ключ ``bootstrap-osd`` не создался автоматически – создайте вручную:
```bash
ceph auth get-or-create client.bootstrap-osd mon 'allow profile bootstrap-osd' -o /etc/ceph/ceph.client.bootstrap-osd.keyring
mkdir -p /var/lib/ceph/bootstrap-osd /etc/pve/priv/ceph
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /etc/pve/priv/ceph/ceph.client.bootstrap-osd.keyring
```
###### Очистите диски (если на них были старые метки)
```bash
for disk in sdc sdd sde sdf; do sgdisk --zap-all /dev/$disk; done
```
###### Создайте OSD
```bash
for disk in sdc sdd sde sdf; do ceph-volume lvm create --data /dev/$disk; done
```
###### Исправьте правило CRUSH и размер репликации (если будет предупреждение о PG), установим количество реплик, создадим пул rbd
```bash
ceph osd crush rule create-replicated rule_osd default osd
ceph osd pool set .mgr crush_rule rule_osd
ceph osd pool set .mgr size 2
ceph osd pool create vm-pool 128
ceph osd pool set vm-pool size 2
ceph osd pool application enable vm-pool rbd
```
###### Активируем пул rbd
```bash
pvesm add rbd vm-pool --pool vm-pool --content images --krbd 1
```

<img width="1286" height="405" alt="image" src="https://github.com/user-attachments/assets/6bbb073d-82a9-4a1e-93e8-197b8e7fc7ee" />

###### Проверьте статус
```bash
ceph -s
ceph osd tree
```
##### 4. Теперь настраиваем Ceph на второй ноде (как отдельный кластер или часть общего)
Убедитесь, что на второй ноде есть конфиг и ключ администратора

1️⃣ Если вы ещё не скопировали их с первой ноды, сделайте это сейчас (на второй ноде):

```bash
scp root@10.113.76.53:/etc/ceph/ceph.conf /etc/ceph/
scp root@10.113.76.53:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
```
2️⃣ Создайте ключ bootstrap-osd на второй ноде
```bash
ceph auth get-or-create client.bootstrap-osd mon 'allow profile bootstrap-osd' -o /etc/ceph/ceph.client.bootstrap-osd.keyring
```
Если команда выполняется без ошибок — ключ создан.

3️⃣ Скопируйте ключ в стандартные каталоги
```bash
mkdir -p /var/lib/ceph/bootstrap-osd /etc/pve/priv/ceph
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /etc/pve/priv/ceph/ceph.client.bootstrap-osd.keyring
chmod 600 /etc/ceph/ceph.client.bootstrap-osd.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring /etc/pve/priv/ceph/ceph.client.bootstrap-osd.keyring
```
4️⃣ Очистите диски (если ещё не делали)
```bash
for disk in sdc sdd sde sdf; do sgdisk --zap-all /dev/$disk; done
```
5️⃣ Создайте OSD
```bash
for disk in sdc sdd sde sdf; do ceph-volume lvm create --data /dev/$disk; done
```
Теперь процесс должен пройти успешно.

6️⃣ Проверьте результат (на любой ноде)
```bash
ceph osd tree
```
Вы увидите новые OSD (ID 4,5,6,7) на хосте r18-izhevsk-r7-v02.


Теперь мы имеем две физические ноды, но все службы управления (mon, mgr) запущены только на первой.

Поскольку этот кластер также работает на базе Proxmox VE, мы сделаем его отказоустойчивым по уже проверенной, безопасной схеме.

#### Шаг 1. Переводим пулы в режим двух нод 
Перед добавлением служб обязательно разрешите кластеру продолжать запись, если одна из нод выйдет из строя. Выполните эти две команды на первой ноде (r18-izhevsk-r7-v01):
```bash
ceph osd pool set vm-pool min_size 1
ceph osd pool set .mgr min_size 1
Проверим применились ли настройки
ceph osd pool ls detail
```
#### Шаг 2. Добавляем второй Монитор и Менеджер
Так как вы уже находитесь в консоли второй ноды (r18-izhevsk-r7-v02), вы вызовете родные утилиты Proxmox PVE. 

Они сами создадут нужные ключи и каталоги.

Выполните на второй ноде (r18-izhevsk-r7-v02) по очереди две команды:
```bash
pveceph mon create

pveceph mgr create
```
>[!Warning]
>Если вдруг ошибка при создании ``pveceph mon createfile '/etc/ceph/ceph.conf' already exists and is not a symlink to /etc/pve/ceph.conf`` и ``pveceph mgr createfile '/etc/ceph/ceph.conf' already exists and is not a symlink to /etc/pve/ceph.conf``
значит на второй ноде (r18-izhevsk-r7-v02) остался старый локальный файл конфигурации /etc/ceph/ceph.conf

##### Делаем бэкап и удаляем старый файл
Выполните на второй ноде (r18-izhevsk-r7-v02) команды, чтобы безопасно убрать мешающий конфиг:
```bash 
Переименовываем старый файл в бэкап (на всякий случай)
mv /etc/ceph/ceph.conf /etc/ceph/ceph.conf.bak
Создаем правильную символическую ссылку на кластерный конфиг Proxmox
ln -s /etc/pve/ceph.conf /etc/ceph/ceph.conf
```
> Теперь повторяем Шаг 3 заново.

Далее проверьте общий статус:
```bash
ceph -s
```
В секции services должно стать:
```text
mon: 2 daemons, quorum r18-izhevsk-r7-v01,r18-izhevsk-r7-v02
mgr: r18-izhevsk-r7-v01(active...), standbys: r18-izhevsk-r7-v02
```
Далее, так как мы находимся в замкнутом контуре без выхода в интернет, нам нужно настроить локальный NTP-сервер на одном из узлов (например, на первом) и использовать его как источник для второго узла, либо настроить оба узла использовать аппаратные часы или друг друга.

В данной ситуации **кластер требует синхронизации времени между узлами**, поэтому достаточно, чтобы они оба имели одинаковое время, даже если оно не абсолютно точное (но желательно близкое к реальному). 

Мы можем настроить один узел как NTP-сервер, а второй синхронизироваться с ним.

**План:**

- На первом узле (r18-izhevsk-r7-v01) настроить chrony как сервер для локальной сети (разрешить синхронизацию с второго узла).

- На втором узле настроить chrony для использования первого узла как источника.

- Запустить chrony на втором узле.

- Проверить синхронизацию.

#### Пошаговая настройка синхронизации времени
##### 1. На первом узле (r18-izhevsk-r7-v01) – настроить chrony как сервер

Отредактируйте конфигурационный файл /etc/chrony.conf:
```bash
vi /etc/chrony.conf
```
Добавьте или убедитесь, что присутствуют следующие строки:

```text
# Разрешить синхронизацию с узла r18-izhevsk-r7-v02 (IP 10.113.76.54)
allow 10.113.76.54

# Также можно разрешить всю подсеть
# allow 10.113.76.0/24

# Оставить локальный источник как запасной (уже должно быть)
local stratum 10
```
После редактирования перезапустите chrony:

```bash
systemctl restart chronyd
systemctl enable chronyd
```
Проверьте статус:
```bash
chronyc tracking
```
Теперь этот узел будет отвечать на NTP-запросы с IP 10.113.76.54.

##### 2. На втором узле (r18-izhevsk-r7-v02) – настроить chrony как клиент

Отредактируйте /etc/chrony.conf:
```bash
nano /etc/chrony.conf
```
Закомментируйте все внешние пулы (если есть) и добавьте строку с указанием первого узла как сервера:
```text
server 10.113.76.53 iburst
```
Также можно добавить локальный источник на случай, если сервер недоступен:
```text
local stratum 10
```
Сохраните файл.

Запустите и включите chrony:
```bash
systemctl enable chronyd
systemctl start chronyd
```
Если возникает ошибка «Cannot talk to daemon» – возможно, демон не запущен. Проверьте:
```bash
systemctl status chronyd
```
Если есть ошибки, посмотрите логи:
```bash
journalctl -u chronyd -n 20
```
##### 3. Проверьте синхронизацию
На втором узле выполните:
```bash
chronyc sources -v
```
Вы должны увидеть сервер 10.113.76.53 в списке с состоянием ^* (выбранный источник) или ^+ (кандидат).

Затем проверьте смещение:
```bash
chronyc tracking
```
Если смещение большое, можно принудительно сделать шаг:
```bash
chronyc -a makestep
```
Теперь проверьте время на обоих узлах – разница должна быть менее 0.1 секунды:
```bash
date
```
##### 4. Если порт UDP 123 заблокирован
На втором узле может быть включён межсетевой экран (firewalld или iptables), который блокирует входящие/исходящие NTP-запросы. На первом узле тоже нужно разрешить.

На обоих узлах выполните (если используется firewalld):
```bash
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload
```
Или если используется iptables:
```bash
iptables -I INPUT -p udp --dport 123 -j ACCEPT
iptables -I OUTPUT -p udp --sport 123 -j ACCEPT
service iptables save
```
##### 5. После синхронизации времени – проверьте кластер
На любом узле выполните:
```bash
ceph -s
ceph health detail
```












<img width="1164" height="314" alt="image" src="https://github.com/user-attachments/assets/a62a25ca-42bb-4c4c-a487-37d26a4bc379" />











###### Проверьте разметку дисков командами 
```bash
pvs && vgs && lvs
```
<img width="1531" height="524" alt="image" src="https://github.com/user-attachments/assets/04fdd7dc-5c3f-4d02-ae59-be0affbf5732" />

```bash
ceph osd df
```

<img width="1486" height="225" alt="image" src="https://github.com/user-attachments/assets/dfc76879-48f0-4586-a42c-68ca7a5e0b14" />

```bash

```


<img width="1407" height="566" alt="image" src="https://github.com/user-attachments/assets/90ac29a6-24fa-4360-be8f-235b612826e3" />


<img width="1754" height="1086" alt="image" src="https://github.com/user-attachments/assets/47de0c68-e47d-4492-85f9-f3ce893c3c38" />


<img width="1748" height="598" alt="image" src="https://github.com/user-attachments/assets/b7bbf0fe-b78f-49ae-8408-dfda8f5b8f80" />


<img width="3049" height="832" alt="image" src="https://github.com/user-attachments/assets/48bf9586-ebbc-49d9-b0fc-2d1dd26bf62b" />









