# Homework #13. Selinux

1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).
2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.


## Проведем подготовку.
Запустим новую виртуалку в vagrant.Centos8 + установим все необходимые пакеты:nginx, net-tools,setools-console,policycoreutils-python.Nginx стартован и слушает на 80 порту,не вызывая никаких подозрений у Selinux
```
[root@localhost ~]# uname -r
4.18.0-80.el8.x86_64
[root@localhost ~]# service nginx restart
Redirecting to /bin/systemctl restart nginx.service

[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-10-04 09:34:19 UTC; 5s ago
  Process: 35285 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 35282 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 35281 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 35286 (nginx)
 
[root@localhost ~]# netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      35286/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      35286/nginx: master 
[root@localhost ~]# sestatus 
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      31
```
## 1 способ setsebool ##
Подправим конфиг nginx, сменив порт на нестандартный, чем заинтересуем Selinux.
```
[root@localhost ~]# vi /etc/nginx/nginx.conf
[root@localhost ~]# cat /etc/nginx/nginx.conf | grep listen
        listen       3400 default_server;
        listen       [::]:3400 default_server;
[root@localhost ~]# systemctl restart  nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xe" for details.
```
После смены порта веб-сервер ожидаемо не взлетел.Смотрим логи.

```
[root@localhost ~]# cat /var/log/audit/audit.log 
type=AVC msg=audit(1601804753.720:1665): avc:  denied  { name_bind } for  pid=35372 comm="nginx" src=3400 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1601804753.720:1665): arch=c000003e syscall=49 success=no exit=-13 a0=8 a1=558bfc57d0e8 a2=10 a3=7ffce89b0a10 items=0 ppid=1 pid=35372 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=PROCTITLE msg=audit(1601804753.720:1665): proctitle=2F7573722F7362696E2F6E67696E78002D74
type=SERVICE_START msg=audit(1601804753.834:1666): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'UID="root" AUID="unset"
```
Воспользуемся утилитой audit2why,скоормив ей на вход логи.
```
[root@localhost ~]# audit2why </var/log/audit/audit.log 
type=AVC msg=audit(1601805249.872:1667): avc:  denied  { name_bind } for  pid=35451 comm="nginx" src=3400 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow system to run with NIS

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@localhost ~]# 
```
Применим предложенное решение и видим, что все заработало.
```
[root@localhost ~]# setsebool -P nis_enabled 1
[root@localhost ~]# systemctl restart nginx
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-10-04 09:57:07 UTC; 8s ago
  Process: 35481 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 35479 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 35477 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 35483 (nginx)
    Tasks: 2 (limit: 2881)
   Memory: 12.1M
   CGroup: /system.slice/nginx.service
           ├─35483 nginx: master process /usr/sbin/nginx
           └─35484 nginx: worker process

Oct 04 09:57:06 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Oct 04 09:57:06 localhost.localdomain nginx[35479]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
```
## 2 способ. Добавим нестандартный порт в список разрешенных ##
Вернем setsebool -P nis_enabled 1 обратно в 0, снова сломав наш nginx на порту 3400.
Выясним "а на каком же порту можно держать веб-сервер?"
```
[root@localhost ~]# semanage port -l | grep http_port
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Видим, что наш 3400 надо добавить в этот список.Добавим, рестарт-все ок, заработало.

```
[root@localhost ~]# semanage port -a -t http_port_t -p tcp 3400
[root@localhost ~]# systemctl restart nginx
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-10-04 10:02:14 UTC; 2s ago
  Process: 35532 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 35530 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
```
## 3 способ. формирование и установка модуля SELinux ##
Откатим изменения из п.2 
```
[root@localhost ~]# semanage port -d -t http_port_t -p tcp 3400
```
Теперь посмотрим, какой модуль надо доустановить.

```
[root@localhost ~]# whereis nginx
nginx: /usr/sbin/nginx /usr/lib64/nginx /etc/nginx /usr/share/nginx /usr/share/man/man8/nginx.8.gz /usr/share/man/man3/nginx.3pm.gz
[root@localhost ~]# ls -Z /usr/sbin/nginx 
system_u:object_r:httpd_exec_t:s0 /usr/sbin/nginx
[root@localhost ~]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log 
********************* ВАЖНО ************************
Чтобы сделать этот пакет политики активным, выполните: semodule -i httpd_add.pp

[root@localhost ~]# semodule -i httpd_add.pp
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-10-04 10:24:24 UTC; 4s ago
   
[root@localhost ~]# netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:3400            0.0.0.0:*               LISTEN      35617/nginx: master 
tcp6       0      0 :::3400                 :::*                    LISTEN      35617/nginx: master 
```
Видим снова успешно стартовавший nginx на нашем выбранном порту.




## Задание 2 ##

1.  Запустим приложенный по ссылке стенд, дождемся конфигурации обеих машин. На каждой убедимся во включенном состоянии selinux и наличии всех необходимых вспомогательных инструментов типа audit2allow  и т д.  
2. Зачистим на обеих машинах /var/log/audit/audit.log для чистоты эксперимента и удобства.  
3. Воспроизведем проблему
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
[root@client ~]# cat /var/log/audit/audit.log 

type=USER_ACCT msg=audit(1601859662.192:1621): pid=25135 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:accounting grantors=pam_access,pam_unix,pam_localuser acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
type=CRED_ACQ msg=audit(1601859662.192:1622): pid=25135 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
type=LOGIN msg=audit(1601859662.192:1623): pid=25135 uid=0 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 old-auid=4294967295 auid=0 tty=(none) old-ses=4294967295 ses=9 res=1
type=USER_START msg=audit(1601859662.407:1624): pid=25135 uid=0 auid=0 ses=9 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:session_open grantors=pam_loginuid,pam_keyinit,pam_limits,pam_systemd acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
type=CRED_REFR msg=audit(1601859662.407:1625): pid=25135 uid=0 auid=0 ses=9 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
type=CRED_DISP msg=audit(1601859662.966:1626): pid=25135 uid=0 auid=0 ses=9 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:setcred grantors=pam_env,pam_unix acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
type=USER_END msg=audit(1601859662.966:1627): pid=25135 uid=0 auid=0 ses=9 subj=system_u:system_r:crond_t:s0-s0:c0.c1023 msg='op=PAM:session_close grantors=pam_loginuid,pam_keyinit,pam_limits,pam_systemd acct="root" exe="/usr/sbin/crond" hostname=? addr=? terminal=cron res=success'
[root@client ~]# audit2why < /var/log/audit/audit.log 
Nothing to do
```
Со стороны клиента исправлять нечего...вроде..Посмотрим что на стороне сервера.

```
[root@ns01 ~]# audit2why < /var/log/audit/audit.log 
type=AVC msg=audit(1601860832.151:2084): avc:  denied  { create } for  pid=5547 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
Как показано, доступ блокирован, в соответствии с отсутствием правила Type Enforcement.Воспользуемся предложенным рецептом.
```
[root@ns01 ~]# audit2allow < /var/log/audit/audit.log 


#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
```
Создадим правила и установим необходимый модуль

```
[root@ns01 ~]# audit2allow -a -M mynamed    
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i mynamed.pp

[root@ns01 ~]# semodule -i mynamed.pp
[root@ns01 ~]# 
```
Проверим со стороны клиента...

```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
; TSIG error with server: clocks are unsynchronized
update failed: NOTAUTH(BADTIME)
> quit
```
Ошибка с доступом изначальная ушла, правда вывалилась другого характера... но борьбу с ней я продолжу в другой раз.
