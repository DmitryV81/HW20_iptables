Домашняя работа № 20. Сценарии iptables.

Задание:

реализовать knocking port

centralRouter может попасть на ssh inetrRouter через knock скрипт

добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост.

запустить nginx на centralServer.

пробросить 80й порт на inetRouter2 8080.

дефолт в инет оставить через inetRouter.

Ход работы.

В Vagrantfile конфигурируются 4 виртуальные машины(inetRouter, centralRouter, inetRouter2, centralServer)

В разделе provision вызывается ansible-playbook для дальнейшей настройки ВМ

Далее через ansible roles происходит настройка групп ВМ:

server - centralServer

router - inetRouter, centralRouter, inetRouter2

В результате получаем на ВМ inetRouter правила iptables реализующие port knocking:

```
[vagrant@inetRouter ~]$ sudo cat /etc/sysconfig/iptables
# sample configuration for iptables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

-N CHAIN-1
-A CHAIN-1 -m recent --name TRY1 --remove
-A CHAIN-1 -m recent --name TRY2 --set
-A CHAIN-1 -j LOG --log-prefix "CHAIN-1: "

-N CHAIN-2
-A CHAIN-2 -m recent --name TRY2 --remove
-A CHAIN-2 -m recent --name TRY3 --set
-A CHAIN-2 -j LOG --log-prefix "CHAIN-2: "

-A INPUT -m recent --update --name TRY1

-A INPUT -p tcp --dport 6543 -m recent --name TRY1 --set
-A INPUT -p tcp --dport 5432 -m recent --rcheck --seconds 10 --name TRY1 -j CHAIN-1
-A INPUT -p tcp --dport 4321 -m recent --rcheck --seconds 10 --name TRY2 -j CHAIN-2


-A INPUT -p tcp --dport 22 -m recent --rcheck --seconds 10 --name TRY3 -j ACCEPT

-A INPUT -p tcp --dport 22 -m state --state NEW -j DROP

COMMIT
*nat
:PREROUTING ACCEPT [1:161]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
```

Вывод iptables -L -v -n

```
[vagrant@inetRouter ~]$ sudo iptables -L -v -n
Chain INPUT (policy DROP 5 packets, 300 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   31  1240 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
  314  708K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    2   120            all  --  *      *       0.0.0.0/0            0.0.0.0/0            recent: UPDATE name: TRY1 side: source mask: 255.255.255.255
    2   120            tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:6543 recent: SET name: TRY1 side: source mask: 255.255.255.255
    1    60 CHAIN-1    tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:5432 recent: CHECK seconds: 10 name: TRY1 side: source mask: 255.255.255.255
    1    60 CHAIN-2    tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:4321 recent: CHECK seconds: 10 name: TRY2 side: source mask: 255.255.255.255
    1    60 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 recent: CHECK seconds: 10 name: TRY3 side: source mask: 255.255.255.255
    7   420 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW

Chain FORWARD (policy ACCEPT 24586 packets, 29M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 257 packets, 43617 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain CHAIN-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    1    60            all  --  *      *       0.0.0.0/0            0.0.0.0/0            recent: REMOVE name: TRY1 side: source mask: 255.255.255.255
    1    60            all  --  *      *       0.0.0.0/0            0.0.0.0/0            recent: SET name: TRY2 side: source mask: 255.255.255.255
    1    60 LOG        all  --  *      *       0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 4 prefix "CHAIN-1: "

Chain CHAIN-2 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    1    60            all  --  *      *       0.0.0.0/0            0.0.0.0/0            recent: REMOVE name: TRY2 side: source mask: 255.255.255.255
    1    60            all  --  *      *       0.0.0.0/0            0.0.0.0/0            recent: SET name: TRY3 side: source mask: 255.255.255.255
    1    60 LOG        all  --  *      *       0.0.0.0/0            0.0.0.0/0            LOG flags 0 level 4 prefix "CHAIN-2: "

```

С ВМ centralRouter сперва пытаемся подключиться по ssh к inetRouter, а после того как попытка не удалась подклюаемся через port knocking

```
[vagrant@centralRouter ~]$ ssh vagrant@192.168.255.1
ssh: connect to host 192.168.255.1 port 22: Connection timed out
[vagrant@centralRouter ~]$ telnet 192.168.255.1 6543
Trying 192.168.255.1...
^C
[vagrant@centralRouter ~]$ telnet 192.168.255.1 5432
Trying 192.168.255.1...
^C
[vagrant@centralRouter ~]$ telnet 192.168.255.1 4321
Trying 192.168.255.1...
^C
[vagrant@centralRouter ~]$ ssh vagrant@192.168.255.1
The authenticity of host '192.168.255.1 (192.168.255.1)' can't be established.
ECDSA key fingerprint is SHA256:cWXtPd+DL3UHcXp01zsKATHav17mlMexx0yKJ/rQUKQ.
ECDSA key fingerprint is MD5:8a:b4:1d:98:b4:3e:92:d7:68:11:b4:df:23:5e:2a:50.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.255.1' (ECDSA) to the list of known hosts.
vagrant@192.168.255.1's password: 
Last login: Sat Apr 15 20:59:12 2023 from 192.168.50.1
[vagrant@inetRouter ~]$ 

```

Проброс порта 8080 с inetRouter2 на порт 80 centralServer реализован с помощью ansible, встроенного модуля iptables:

```
- name: enable iptables NAT/DNAT on inetRouter2
  iptables:
    table: nat
    chain: PREROUTING
    protocol: tcp
    destination_port: "8080"
    jump: DNAT
    to_destination: 192.168.0.2:80
  when: (ansible_hostname == "inetRouter2")
  
- name: add postrouting rule on inetRouter2
  iptables:
    table: nat
    chain: POSTROUTING
    jump: MASQUERADE
  when: (ansible_hostname == "inetRouter2")
```

