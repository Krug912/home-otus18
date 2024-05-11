# SELinux - когда все запрещено

##   1. Задание  Запустить nginx на нестандартном порту 3-мя разными способами:

 - переключатели setsebool;
 - добавление нестандартного порта в имеющийся тип;
 - формирование и установка модуля SELinux.
    К сдаче:
    README с описанием каждого решения (скриншоты и демонстрация приветствуются).
Результат работы команды vagrant up
```
    selinux: Installed:
    selinux:   almalinux-logos-httpd-90.5.1-1.1.el9.noarch
    selinux:   nginx-1:1.20.1-14.el9_2.1.alma.1.x86_64
    selinux:   nginx-core-1:1.20.1-14.el9_2.1.alma.1.x86_64
    selinux:   nginx-filesystem-1:1.20.1-14.el9_2.1.alma.1.noarch
    selinux: 
    selinux: Complete!
    selinux: Job for nginx.service failed because the control process exited with error code.
    selinux: See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
    selinux: × nginx.service - The nginx HTTP and reverse proxy server
    selinux:      Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
    selinux:      Active: failed (Result: exit-code) since Thu 2024-05-09 22:56:45 UTC; 46ms ago
    selinux:     Process: 3195 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:     Process: 3210 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:         CPU: 49ms
    selinux: 
    selinux: May 09 22:56:44 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: May 09 22:56:45 selinux nginx[3210]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: May 09 22:56:45 selinux nginx[3210]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: May 09 22:56:45 selinux nginx[3210]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: May 09 22:56:45 selinux systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
    selinux: May 09 22:56:45 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
    selinux: May 09 22:56:45 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
Состояние firewalld.service
```
[vagrant@selinux ~]$ systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Thu 2024-05-09 22:54:48 UTC; 3min 24s ago
       Docs: man:firewalld(1)
   Main PID: 669 (firewalld)
      Tasks: 2 (limit: 5822)
     Memory: 25.6M
        CPU: 1.574s
     CGroup: /system.slice/firewalld.service
             └─669 /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid
```
Отключаем firewalld.service
```
#systemctl stop firewalld.service
#systemctl disable firewalld.service
#systemctl status firewalld.service
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)

May 09 22:54:45 alma9.localdomain systemd[1]: Starting firewalld - dynamic firewall daemon...
May 09 22:54:48 alma9.localdomain systemd[1]: Started firewalld - dynamic firewall daemon.
May 09 23:01:04 selinux systemd[1]: Stopping firewalld - dynamic firewall daemon...
May 09 23:01:04 selinux systemd[1]: firewalld.service: Deactivated successfully.
May 09 23:01:04 selinux systemd[1]: Stopped firewalld - dynamic firewall daemon.
May 09 23:01:04 selinux systemd[1]: firewalld.service: Consumed 1.669s CPU time.
```
Проверим конфигурацию nginx
```
[root@selinux vagrant]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@selinux vagrant]# 
```
Проверим режим работы SELinux
```
[root@selinux vagrant]# getenforce 
Enforcing
```
### Способ 1. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
```
[root@selinux vagrant]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1715295405.013:609): avc:  denied  { name_bind } for  pid=3210 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
[root@selinux vagrant]# grep 1715295405.013:609 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1715295405.013:609): avc:  denied  { name_bind } for  pid=3210 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@selinux vagrant]# 
```
Включим параметр nis_enabled и перезапустим nginx
```
[root@selinux vagrant]# setsebool -P nis_enabled 1
[root@selinux vagrant]# systemctl restart nginx.service 
[root@selinux vagrant]# systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-05-10 01:34:23 UTC; 7s ago
    Process: 53400 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 53401 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 53403 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 53404 (nginx)
      Tasks: 3 (limit: 5822)
     Memory: 4.3M
        CPU: 129ms
     CGroup: /system.slice/nginx.service
             ├─53404 "nginx: master process /usr/sbin/nginx"
             ├─53405 "nginx: worker process"
             └─53406 "nginx: worker process"

May 10 01:34:23 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 10 01:34:23 selinux nginx[53401]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 10 01:34:23 selinux nginx[53401]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 10 01:34:23 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Nginx запущен на 4881.png![image](https://github.com/Krug912/home-otus18/assets/162484306/28581c86-85a3-4496-a273-70bbbad6e716)

Вернём запрет работы nginx на порту 4881
```
[root@selinux vagrant]# setsebool -P nis_enabled 0
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> off
[root@selinux vagrant]# systemctl restart nginx.service 
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
[root@selinux vagrant]# systemctl status nginx.service 
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Fri 2024-05-10 01:39:25 UTC; 4s ago
   Duration: 5min 1.766s
    Process: 53424 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 53425 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 65ms

May 10 01:39:25 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 10 01:39:25 selinux nginx[53425]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 10 01:39:25 selinux nginx[53425]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission deni>
May 10 01:39:25 selinux nginx[53425]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 10 01:39:25 selinux systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
May 10 01:39:25 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
May 10 01:39:25 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```

### Способ 2. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
```
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux vagrant]# systemctl restart nginx.service 
[root@selinux vagrant]# systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-05-10 01:41:57 UTC; 6s ago
    Process: 53464 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 53465 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 53466 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 53467 (nginx)
      Tasks: 3 (limit: 5822)
     Memory: 2.9M
        CPU: 98ms
     CGroup: /system.slice/nginx.service
             ├─53467 "nginx: master process /usr/sbin/nginx"
             ├─53468 "nginx: worker process"
             └─53469 "nginx: worker process"

May 10 01:41:57 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 10 01:41:57 selinux nginx[53465]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 10 01:41:57 selinux nginx[53465]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 10 01:41:57 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
![image](https://github.com/Krug912/home-otus18/assets/162484306/60497ba9-8df6-48dc-aebe-c1699bf360c4)

```
[root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux vagrant]# systemctl restart nginx.service 
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```
### Способ 3. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту
```
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux vagrant]# semodule -i nginx.pp
[root@selinux vagrant]# systemctl start nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-05-10 01:50:28 UTC; 14s ago
    Process: 53643 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 53644 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 53645 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 53646 (nginx)
      Tasks: 3 (limit: 5822)
     Memory: 2.9M
        CPU: 100ms
     CGroup: /system.slice/nginx.service
             ├─53646 "nginx: master process /usr/sbin/nginx"
             ├─53647 "nginx: worker process"
             └─53648 "nginx: worker process"

May 10 01:50:27 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 10 01:50:27 selinux nginx[53644]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 10 01:50:27 selinux nginx[53644]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 10 01:50:28 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
![image](https://github.com/Krug912/home-otus18/assets/162484306/57bfbcae-df84-4166-a5ed-821e39af387b)

##   2. Задание Обеспечение работоспособности приложения при включенном SEL
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ 
```
