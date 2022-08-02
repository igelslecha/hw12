# hw12
SELinux

**Домашнее задание**

*Практика с SELinux*

Цель:
Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

*Описание/Пошаговая инструкция выполнения домашнего задания:*

* Запустить nginx на нестандартном порту 3-мя разными способами:
* переключатели setsebool;
* добавление нестандартного порта в имеющийся тип;
* формирование и установка модуля SELinux. К сдаче:
* README с описанием каждого решения (скриншоты и демонстрация приветствуются).
* Обеспечить работоспособность приложения при включенном selinux.
* развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
* выяснить причину неработоспособности механизма обновления зоны (см. README);
* предложить решение (или решения) для данной проблемы;
* выбрать одно из решений для реализации, предварительно обосновав выбор;
* реализовать выбранное решение и продемонстрировать его работоспособность. К сдаче:
* README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
* исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
 
 **Решение**
 *В каталоге данной домашней работы создаю файл Vagrantfile, заполняю по методичке*
 
```
# -*- mode: ruby -*-
# vim: set ft=ruby :
MACHINES = {
  :selinux => {
    :box_name => "centos/7",
    :box_version => "2004.01",
    #:provision => "test.sh",
  },
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
      config.vm.define boxname do |box|
        box.vm.box = boxconfig[:box_name]
        box.vm.box_version = boxconfig[:box_version]
        box.vm.host_name = "selinux"
        box.vm.network "forwarded_port", guest: 4881, host: 4881
        box.vm.provider :virtualbox do |vb|
              vb.customize ["modifyvm", :id, "--memory", "1024"]
              needsController = false
        end
        box.vm.provision "shell", inline: <<-SHELL
          #install epel-release
          yum install -y epel-release
          #install nginx
          yum install -y nginx
          #change nginx port
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen 80;/listen 4881;/' /etc/nginx/nginx.conf
          #disable SELinux
          #setenforce 0
          #start nginx
          systemctl start nginx
          systemctl status nginx
          #check nginx port
          ss -tlpn | grep 4881
        SHELL
    end
  end
end
~      
```
*Запускаю и получаю ошибку о невозможности запустить nginx*
```
***
selinux: 
    selinux: Complete!
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
***
```
*Захожу в виртуальную машину*
*Заранее захожу в судо*
*Проверяю настройки файерволла (выключен)*
```
root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
*Проверяю конфиг nginx'a на ошибки*
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
*Проверяю статус  SELinux (Данный режим означает, что SELinux блокирует запрещаённую активность)*
```
[root@selinux ~]# getenforce
Enforcing
```
**1. Способ**
*Нахожу время когда система ругалась на порт 4881 в логе*
```
[root@selinux ~]# vi /var/log/audit/audit.log
```
*Поиск в vi  осуществляется через разделить*
```
type=AVC msg=audit(1658487548.945:858): avc:  denied  { name_bind } for  pid=2987 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
*Натравливаю audit2why на стоку с меткой 1658487548.945:858b и получаю ошибку потому как не установил соотвествующий пакет*
*Устанавливаю*
```
[root@selinux ~]# yum install policycoreutils-python
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.sale-dedic.com
 * epel: mirror.cloudhosting.lv
 * extras: mirrors.datahouse.ru
 * updates: mirror.yandex.ru
Resolving Dependencies
--> Running transaction check
***
Installed:
  policycoreutils-python.x86_64 0:2.5-34.el7                                    

Dependency Installed:
  audit-libs-python.x86_64 0:2.8.5-4.el7 checkpolicy.x86_64 0:2.5-8.el7        
  libcgroup.x86_64 0:0.41-21.el7         libsemanage-python.x86_64 0:2.5-14.el7
  python-IPy.noarch 0:0.75-6.el7         setools-libs.x86_64 0:3.3.8-4.el7     

Complete!


```
*Натравливаю опять*
```
[root@selinux ~]# grep 1658487548.945:858 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1658487548.945:858): avc:  denied  { name_bind } for  pid=2987 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
*Утилита советует включить nis_enabled, что и делаю, и сразу проверяю*
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-24 03:18:57 UTC; 26s ago
  Process: 24416 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 24414 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 24413 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 24418 (nginx)
   CGroup: /system.slice/nginx.service
           ├─24418 nginx: master process /usr/sbin/nginx
           └─24420 nginx: worker process

Jul 24 03:18:57 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Jul 24 03:18:57 selinux nginx[24414]: nginx: the configuration file /etc/ng...ok
Jul 24 03:18:57 selinux nginx[24414]: nginx: configuration file /etc/nginx/...ul
Jul 24 03:18:57 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
[root@selinux ~]# 
```
*Возвращаю все в исходное состояние*
```
root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2022-07-24 03:25:49 UTC; 8s ago
```
**2. Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью
добавления нестандартного порта в имеющийся тип:**
*Поиск имеющегося типа, для http трафика:*
```
root@selinux ~]# semanage port -l | grep htt
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
*Добавляю порт в тип http_port_t и сразу проверяю*
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-07-26 02:57:54 UTC; 4s ago
  Process: 25436 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25434 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 25433 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25438 (nginx)
   CGroup: /system.slice/nginx.service
           ├─25438 nginx: master process /usr/sbin/nginx
           └─25440 nginx: worker process

Jul 26 02:57:54 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Jul 26 02:57:54 selinux nginx[25434]: nginx: the configuration file /etc/ng...ok
Jul 26 02:57:54 selinux nginx[25434]: nginx: configuration file /etc/nginx/...ul
Jul 26 02:57:54 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```
*Удаляю порт из типа http_port_t перезагружаю nginx и проверяю результат*
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2022-07-26 03:00:56 UTC; 11s ago
  Process: 25436 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25460 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 25458 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25438 (code=exited, status=0/SUCCESS)

Jul 26 03:00:56 selinux systemd[1]: Stopped The nginx HTTP and reverse prox...r.
Jul 26 03:00:56 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Jul 26 03:00:56 selinux nginx[25460]: nginx: the configuration file /etc/ng...ok
Jul 26 03:00:56 selinux nginx[25460]: nginx: [emerg] bind() to [::]:4881 fa...d)
Jul 26 03:00:56 selinux nginx[25460]: nginx: configuration file /etc/nginx/...ed
Jul 26 03:00:56 selinux systemd[1]: nginx.service: control process exited, ...=1
Jul 26 03:00:56 selinux systemd[1]: Failed to start The nginx HTTP and reve...r.
Jul 26 03:00:56 selinux systemd[1]: Unit nginx.service entered failed state.
Jul 26 03:00:56 selinux systemd[1]: nginx.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
```
**3. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:**
*Пробую запустить nginx и проверяю последнюю запись из лога относящуюся к nginx*
```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# grep nginx /var/log/audit/audit.log
***
type=SERVICE_START msg=audit(1658804912.650:1611): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
*Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:*
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp
```
*Утилита проанализировав логи создала модуль и предложила активизироваеть его командой semodule -i nginx.pp, что я и сделал, после этого проверяю запуск nginx*
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-07-26 03:22:25 UTC; 7s ago
  Process: 25547 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 25544 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 25543 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 25549 (nginx)
   CGroup: /system.slice/nginx.service
           ├─25549 nginx: master process /usr/sbin/nginx
           └─25551 nginx: worker process

Jul 26 03:22:25 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Jul 26 03:22:25 selinux nginx[25544]: nginx: the configuration file /etc/ng...ok
Jul 26 03:22:25 selinux nginx[25544]: nginx: configuration file /etc/nginx/...ul
Jul 26 03:22:25 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```
*После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки. Проверяю список модулей*
```
[root@selinux ~]# semodule -l
abrt	1.4.1
accountsd	1.1.0
acct	1.6.0
afs	1.9.0
***
netutils	1.12.1
networkmanager	1.15.2
nginx	1.0
ninfod	1.0.0
nis	1.12.0
nova	1.0.0
***
```
*Удаляю модуль*
```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```
***Вторая часть задания***
*Выполняю клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git, перехожу в каталог со стендом: cd otus-linux-adm/selinux_dns_problems и разворачиваю 2 ВМ с помощью vagrant: vagrant up, после того как машинки поднялись проверяю результат*
```
igels@LaptopAll:~/hw12/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```
*Захожу в ВМ client*
```
igels@LaptopAll:~/hw12/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Wed Jul 27 17:01:39 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
```
*Попробуем внести изменения в зону: n*
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
*Изменения внести не получилось. Нужно посмотреть логи SELinux, чтобы понять в чём может быть проблема. Для этого воспользуюсь утилитой audit2why:*
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```
*Не закрывая сессию на клиенте, подключаюсь к серверу ns01 и проверяю логи SELinux:*
```
igels@LaptopAll:~/hw12/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Wed Jul 27 17:00:12 2022 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1658941998.429:1958): avc:  denied  { create } for  pid=5295 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
*Из лога вижу, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. Проверяю данную проблему в каталоге /etc/named*
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
*В данном случае контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Проверяю в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux:*
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
```
*Изменим тип контекста безопасности для каталога /etc/named:*
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
*Пробую повторно внести изменения*
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44519
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

;; Query time: 9 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jul 27 17:44:31 UTC 2022
;; MSG SIZE  rcvd: 96

```
*Перезагружаю хосты и повторно тестирую через программу dig*
```

[root@client ~]# reboot          
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
igels@LaptopAll:~/hw12/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Wed Jul 27 17:08:53 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
[vagrant@client ~]$ sudo -i
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50943
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

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jul 27 17:56:58 UTC 2022
;; MSG SIZE  rcvd: 96
```
*Работает. После перезагрузки настройки сохранились. Для того, чтобы вернуть правила обратно, можно ввести команду:*
```
[root@ns01 ~]# reboot
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
igels@LaptopAll:~/hw12/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Wed Jul 27 17:20:45 2022 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```
**Представлен исправленный стенд, где в плейбук включена команда исправления**
```
  - name: fix rights under SELinux
    ansible.builtin.command: chcon -R -t named_zone_t /etc/named
```
