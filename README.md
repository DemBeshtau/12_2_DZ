# Практика с SELinux #
1. Обеспечить работоспособность приложения при включенном SELinux:
    - развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
    - выяснить причину неработоспособности механизма обновления зоны;
    - предложить решение (или решения) для данной проблемы;
    - выбрать одно из решений для реализации, предварительно обосновав выбор;
    - реализовать выбранное решение и продемонстрировать его работоспособность.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14, Ansible 9.3.0 и образа CentOS 7 версии 1804_2.<br/> 

&ensp;&ensp;По условию задания при попытке  удалённо (с рабочей станции) внести изменения в зону ddns.lab происходит ошибка update failed: SERVFAIL. Также сообщается,
инженер перепроверил содержимое конфигурационных файлов и, убедившись, что с ними всё в порядке, предположил наличие проблемы из-за работы SELinux.<br/>
&ensp;&ensp;В ходе выполнения задания, после развёртывания стенда и повторения вышеуказанных действий, оценивалось содержимое audit-лога на ВМ ns01.<br/>
ВМ client:    
```shell
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
&ensp;&ensp;&ensp;&ensp;ВМ ns01:
```shell
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1717502327.586:1799): avc:  denied  { search } for  pid=5565 comm="isc-worker0000" name="net" dev="proc" ino=29746 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1717502327.586:1800): avc:  denied  { search } for  pid=5565 comm="isc-worker0000" name="net" dev="proc" ino=29746 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1717503988.835:1858): avc:  denied  { create } for  pid=5565 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
&ensp;&ensp;&ensp;&ensp;В методическом материале к заданию, при оценке audit-лога внимание уделялось строке type=AVC msg=audit(1717503988.835:1858): avc:  denied  { create } for  pid=5565 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file. По итогам изучения лога установлено, что ошибка возникла из-за контекста безопасности - несоответствие типов субъекта (named_t) и объекта (etc_t). После изменения типа контекста безопасности для каталога /etc/named с etc_t на named_zone_t проблема была устранена.<br/>
&ensp;&ensp;&ensp;&ensp;После отработки вышеуказанной методики было замечено, что в выводе таких утитлит, как audit2why, sealert, а также в /var/log/messages содержатся рекомендации системы SELinux о создании загружаемых модулей, устраняющих возникающие проблемы с доступом. Данная возможность показалась оптимальной с точки зрения оперативности решения проблем. Создание и загрузка модулей в соответствии с инструкциями SELinux, представляет собой достаточно тривиальную задачу для системного администратора, не требующую погружения в работу политик и контекстов безопасности системы. Руководствуясь данными соображениями, было принято решение разрешить возникшие проблемы с помощью создания и загрузки модуля SELinux.
### Ход решения ###
1. Перевод системы SELinux в режим permissive. Данное действие необходимо для того, что бы позволить внести изменения в зону ddns.lab с удалённого хоста client и получить максимум предупреждающих сообщений системы безопасности в audit.log:<br/>
```shell
[root@ns01 ~]# getenforce
Enforcing
[root@ns01 ~]# setenforce 0
[root@ns01 ~]# getenforce
[root@ns01 ~]# echo > /var/log/audit/audit.log
Permissive
```
2. Внесение изменений в зону ddns.lab и проверка их применения c удалённого хоста client:<br/>
```shell
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47816
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 5 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jun 05 18:38:06 UTC 2024
;; MSG SIZE  rcvd: 96
```
3. Проверка audit-лога на ВМ ns01 с помощью утилит audit2why и sealert:<br/>
```shell
[root@ns01 ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1717612588.365:752): avc:  denied  { create } for  pid=879 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1717612588.365:752): avc:  denied  { write } for  pid=879 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="dm-0" ino=100834444 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# sealert -a /var/log/audit/audit.log | grep ""

found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/named доступ create к файл named.ddns.lab.view1.jnl.

*****  Модуль catchall_labels предлагает (точность 83.8)  ********************

Если вы хотите разрешить named иметь create доступ к named.ddns.lab.view1.jnl $TARGET_УЧЕБНЫЙ КЛАСС
То необходимо изменить метку на named.ddns.lab.view1.jnl
Сделать
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
где FILE_TYPE может принимать значения: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t. 
Затем выполните: 
restorecon -v 'named.ddns.lab.view1.jnl'


*****  Модуль catchall предлагает (точность 17.1)  ***************************

Если вы считаете, что named должно быть разрешено create доступ к named.ddns.lab.view1.jnl file по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
# semodule -i my-iscworker0000.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:named_t:s0
Целевой контекст              system_u:object_r:etc_t:s0
Целевые объекты               named.ddns.lab.view1.jnl [ file ]
Источник                      isc-worker0000
Путь к источнику              /usr/sbin/named
Порт                          <Unknown>
Узел                          <Unknown>
Исходные пакеты RPM           bind-9.11.4-26.P2.el7_9.15.x86_64
Целевые пакеты RPM            
Пакет регламента              selinux-policy-3.13.1-192.el7_5.3.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Permissive
Имя узла                      ns01
Платформа                     Linux ns01 3.10.0-862.2.3.el7.x86_64 #1 SMP Wed
                              May 9 18:05:47 UTC 2018 x86_64 x86_64
Счетчик уведомлений           1
Впервые обнаружено            2024-06-05 18:36:28 UTC
В последний раз               2024-06-05 18:36:28 UTC
Локальный ID                  f4cca6b2-1761-439a-9392-f25ce2173afb

Построчный вывод сообщений аудита
type=AVC msg=audit(1717612588.365:752): avc:  denied  { create } for  pid=879 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file


type=AVC msg=audit(1717612588.365:752): avc:  denied  { write } for  pid=879 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="dm-0" ino=100834444 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file


type=SYSCALL msg=audit(1717612588.365:752): arch=x86_64 syscall=open success=yes exit=ENXIO a0=7f8c946d4050 a1=241 a2=1b6 a3=24 items=0 ppid=1 pid=879 auid=4294967295 uid=25 gid=25 euid=25 suid=25 fsuid=25 egid=25 sgid=25 fsgid=25 tty=(none) ses=4294967295 comm=isc-worker0000 exe=/usr/sbin/named subj=system_u:system_r:named_t:s0 key=(null)

Hash: isc-worker0000,named_t,etc_t,file,create
```
4. Подготовка и загрузка локального модуля политики, переключение SELinux в режим enforcing:<br/>
```shell
[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker0000.pp

[root@ns01 ~]# semodule -i my-iscworker0000.pp
[root@ns01 ~]# setenforce 1
[root@ns01 ~]# getenforce
Enforcing
```
5. Повторное внесение изменений в зону ddns.lab и проверка их применения c удалённого хоста client:<br/>
```shell
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.17
> send
> quit
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43270
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15
www.ddns.lab.		60	IN	A	192.168.50.17

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jun 05 18:52:44 UTC 2024
;; MSG SIZE  rcvd: 112
```
&ensp;&ensp;&ensp;&ensp;Таким образом продемонстрирована состоятельность выбора метода решения проблемы обновления зоны с помощью создания и загрузки локального модуля политики безопасности.<br/>
&ensp;&ensp;&ensp;&ensp;Однако стоит отметить, что при просмотре содержимого файла с описанием сущностей для политики /root/my-iscworker0000.te, видно, что субъектам с типом named_t разрешено создание, переименование файлов с типом etc_t, также разрешена запись в них и удаление на них ссылок. Несмотря на простоту реализации данный метод является менее безопасным в сравнении с методом, в котором тип контекста безопасности изменяется прицезионно - только для директории /etc/named.
```shell
[root@ns01 ~]# cat my-iscworker0000.te

module my-iscworker0000 1.0;

require {
	type etc_t;
	type named_t;
	class file { create rename unlink write };
}

#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file { create rename unlink write };

```
##### &ensp;&ensp; Более подробный ход решения и вывод команд представлен в прилагающихся логах утилиты Script - ns01_1.log и client_1.log. #####
