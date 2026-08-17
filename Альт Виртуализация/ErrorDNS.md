### Ошибка пинга DNS как исправлять
```bash
[root@r01 ~]# ping ya.ru
ping: ya.ru: Temporary failure in name resolution
```
Есть сервер Альт Виртуализации с такой IP адресацией 
```bash
[root@r01 ~]# ip -c a | grep  BROADCAST,MULTICAST,UP
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr0 state UP group default qlen 1000
10: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
[root@r01 ~]# ip -c a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 10.1.100.53/24 scope global vmbr0
[root@r01 ~]#
```
