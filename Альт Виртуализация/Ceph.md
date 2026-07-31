### Разберём как подключить Ceph мониторинг
Выполним команду 
```bash
lsblk
```
<img width="1294" height="509" alt="image" src="https://github.com/user-attachments/assets/51ed0326-ca07-41df-bb0f-ac4aab78d482" />

Видим штатное отображение thin provisioning (thin pool) в LVM.

То, что вы видите — это не «дублирование» данных, а мета-устройства (tmeta и tdata), из которых состоит пул pve-data-tpool. Именно так LVM показывает внутреннее устройство thin-пула.

Вот что означают эти строки:

pve-data_tmeta (253:2) — это метаданные пула (где и какие блоки лежат).

pve-data_tdata (253:3) — это сами данные пула.

pve-data-tpool (253:4) — это объединение метаданных и данных в единый пул.

pve-data (253:5) — это «главный» том в этом пуле, через который Proxmox управляет тонкими дисками.

pve-vm--100--disk--0 и pve-vm--102--disk--0 — это виртуальные диски ВМ, которые лежат внутри этого пула (используют его пространство динамически).

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

VG pve объединяет два физических тома: /dev/sda2 (894 ГБ) и /dev/sdb (894 ГБ).

Однако тонкий пул data использует только 846.75 ГБ – это соответствует размеру /dev/sda2. Диск /dev/sdb не добавлен в тонкий пул, хотя в VG есть свободное место (795 ГБ).

Существующие тома ВМ и снапшоты занимают ~12% пула, свободно ~740 ГБ в самом пуле (но не в VG).

Неиспользуемые диски:

sdc, sdd, sde, sdf – по 3.6 ТБ, полностью свободны (нет разделов, не в LVM).

И есть задача расширить корневой диск.

##### 1. Расширяем корневой диск с подключением не использованным пространством диска sdb
```bash
vgextend pve /dev/sdb
lvextend -L +100G /dev/pve/root
resize2fs /dev/mapper/pve-root
```

<img width="1427" height="197" alt="image" src="https://github.com/user-attachments/assets/95ff53de-849c-49b3-b380-c042eafb38cb" />

<img width="1424" height="701" alt="image" src="https://github.com/user-attachments/assets/1463636d-eda7-47f0-9409-edcf1b7d9a20" />

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

##### 3. Настраиваем Ceph на этой ноде (как отдельный кластер или часть общего)
4 диска по 3.6 ТБ – идеально для создания OSD. Если вы хотите использовать Ceph для отказоустойчивого хранилища

Для начала перейдём на вкладку Datacenter -> Ceph -> configure Ceph и нажимаем на эту кнопку

<img width="1386" height="671" alt="image" src="https://github.com/user-attachments/assets/10813815-42ed-41ad-8de2-a39f8d6fa027" />

Далее выбираем сетевые настройки (CIDR) и ок

<img width="1392" height="699" alt="image" src="https://github.com/user-attachments/assets/8373607b-8d09-4231-a448-083995202229" />

<img width="1391" height="699" alt="image" src="https://github.com/user-attachments/assets/3f51c526-7d83-4de9-9ba0-3eef1b928b83" />

Теперь переходим в Shell на первой ноде

###### Создайте монитор (первый на этой ноде)
pveceph mon create

###### Проверьте, что кластер создался
ceph -s   # теперь должна быть информация, хоть и с предупреждениями

###### Если ключ bootstrap-osd не создался автоматически – создайте вручную:
ceph auth get-or-create client.bootstrap-osd mon 'allow profile bootstrap-osd' -o /etc/ceph/ceph.client.bootstrap-osd.keyring
mkdir -p /var/lib/ceph/bootstrap-osd /etc/pve/priv/ceph
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
cp /etc/ceph/ceph.client.bootstrap-osd.keyring /etc/pve/priv/ceph/ceph.client.bootstrap-osd.keyring

###### Очистите диски (если на них были старые метки)
for disk in sdc sdd sde sdf; do sgdisk --zap-all /dev/$disk; done

###### Создайте OSD
for disk in sdc sdd sde sdf; do ceph-volume lvm create --data /dev/$disk; done

###### Исправьте правило CRUSH и размер репликации (если будет предупреждение о PG)
ceph osd crush rule create-replicated rule_osd default osd
ceph osd pool set .mgr crush_rule rule_osd
ceph osd pool set .mgr size 2

###### Проверьте статус
ceph -s
ceph osd tree


##### Другой вариант
Выполним команду 
```bash
lsblk
```
<img width="1391" height="488" alt="image" src="https://github.com/user-attachments/assets/3c0837b8-ed16-4ea0-a129-6570b7e04d4c" />













