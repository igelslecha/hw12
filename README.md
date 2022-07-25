# hw12
SELinux

**Домашнее задание**

*Практика с SELinux*

Цель:
Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

* Описание/Пошаговая инструкция выполнения домашнего задания:

* Запустить nginx на нестандартном порту 3-мя разными способами:

*переключатели setsebool;

*добавление нестандартного порта в имеющийся тип;

*формирование и установка модуля SELinux. К сдаче:

*README с описанием каждого решения (скриншоты и демонстрация приветствуются).

*Обеспечить работоспособность приложения при включенном selinux.

*развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;

*выяснить причину неработоспособности механизма обновления зоны (см. README);

*предложить решение (или решения) для данной проблемы;

*выбрать одно из решений для реализации, предварительно обосновав выбор;

*реализовать выбранное решение и продемонстрировать его работоспособность. К сдаче:

*README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;

*исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

* Нахожу время когда система ругалась на порт 4881*

```
type=AVC msg=audit(1658487548.945:858): avc:  denied  { name_bind } for  pid=2987 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
* Натравливаю audit2why на стоку с меткой 1658487548.945:858*
* И получаю ошибку потому как не установил соотвествующий пакет*
* Устанавливаю *
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

