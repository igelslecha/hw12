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
*Проверяю статус  SELinux (Данный режим означает, что SELinux блокирует запрещаённую активность*
```
[root@selinux ~]# getenforce
Enforcing
```
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
* Натравливаю опять *
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
**Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью
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
*
