# Homework #11. Ansible

Домашнее задание
Первые шаги с Ansible
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:
- необходимо использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible

  
*Опциально сделать это все с помощью Ansible role*

---

## Проведем подготовку.
Запустим новую виртуалку согласно vagrant file.пока без применения роли.  
```
vagrant up --no-provision
```

Проверим конфиг ssh  
```
root@mike-UL20A:/home/mike/hometask11/dz11# vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/mike/hometask11/dz11/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel

```

Удостоверимся, что порт 2222 вписан в конфиге, если что -подправим.Создадим файл hosts, где опишем нашу небольшую инфраструктуру
```
cat hosts

[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```

Теперь запустим виртуалку вместе c автоконфигурацией ее ансибл плейбуком nginx-with-role.yml.
```
vagrant provision
...
RUNNING HANDLER [nginx-role : reload nginx] ************************************
changed: [nginx]

PLAY RECAP *********************************************************************
nginx                      : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```


## Проверка.
Перейдем в браузере по http://192.168.11.150:8080/ и увидим открывшуюся страничку на нашем веб сервере.
Залогинившись в виртуалку проверим изменения, внесенные плейбуком

Проверим, что  8080 слушается сервером nginx
```
vagrant ssh

[vagrant@nginx ~]$ sudo ss -tulpn | grep nginx
tcp    LISTEN     0      128       *:8080                  *:*                   users:(("nginx",pid=3172,fd=6),("nginx",pid=3095,fd=6))

```

Кроме того, удостоверимся, что сервис стартован.
```
[vagrant@nginx ~]$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-09-22 21:29:47 UTC; 14min ago
  Process: 3171 ExecReload=/bin/kill -s HUP $MAINPID (code=exited, status=0/SUCCESS)
  Process: 3093 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3092 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3090 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3095 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3095 nginx: master process /usr/sbin/nginx
           └─3172 nginx: worker process

Sep 22 21:29:46 nginx systemd[1]: Starting The nginx HTTP and reverse proxy server...
Sep 22 21:29:47 nginx nginx[3092]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Sep 22 21:29:47 nginx nginx[3092]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Sep 22 21:29:47 nginx systemd[1]: Started The nginx HTTP and reverse proxy server.
Sep 22 21:29:55 nginx systemd[1]: Reloading The nginx HTTP and reverse proxy server.
Sep 22 21:29:55 nginx systemd[1]: Reloaded The nginx HTTP and reverse proxy server.
[vagrant@nginx ~]$ 

```