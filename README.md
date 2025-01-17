# selinux_1-3
## описание ```selinux.yml```
1. Задача ```Update and install package```
* Обновляем пакеты.
* Устанавливаем пакеты ```nginx```, ```setroubleshoot-server```, ```selinux-policy-mls```, ```setools-console```, ```policycoreutils-python-utils```, ```policycoreutils-newrole```
2. Задача ```Copy nginx configuration```
* Копируем конфигурационный файл ```nginx.conf``` в папку ```/etc/nginx```
* При изменении конфигурационного файла ```nginx.conf``` перезапускаем сервис ```nginx.service```

### Добавление ```nginx``` в ```SELinux```
#### Разрешение работы нестандартного порта ```4881``` с помощью переключателя ```setsebool```
* Запускаем ```nginx``` и получаем ошибку
```bash
[root@localhost ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```
* Проверяем статус сервиса ```nginx```
```bash
[root@localhost ~]# systemctl status nginx
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since Fri 2025-01-17 10:36:03 MSK; 12min ago
   Duration: 20.437s
    Process: 15419 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 15420 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 114ms

Jan 17 10:36:03 localhost.localdomain systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jan 17 10:36:03 localhost.localdomain nginx[15420]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 17 10:36:03 localhost.localdomain nginx[15420]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 17 10:36:03 localhost.localdomain nginx[15420]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 17 10:36:03 localhost.localdomain systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Jan 17 10:36:03 localhost.localdomain systemd[1]: nginx.service: Failed with result 'exit-code'.
Jan 17 10:36:03 localhost.localdomain systemd[1]: Failed to start nginx.service - The nginx HTTP and reverse proxy server.
```
1. В файле ```/var/log/audit/audit.log``` ищем события связанные с нашим нестандартным портом ```4881```
```bash
[root@localhost ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1737099326.966:1243): avc:  denied  { name_bind } for  pid=14042 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1737099363.258:1609): avc:  denied  { name_bind } for  pid=15420 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1737100109.504:1626): avc:  denied  { name_bind } for  pid=15552 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
2. Теперь отфильтруем события точно по времени
```bash
[root@localhost ~]# cat /var/log/audit/audit.log | grep '1737100109.504:1626'
type=AVC msg=audit(1737100109.504:1626): avc:  denied  { name_bind } for  pid=15552 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1737100109.504:1626): arch=c000003e syscall=49 success=no exit=-13 a0=8 a1=55c9e85f7508 a2=10 a3=7ffe94000af0 items=0 ppid=1 pid=15552 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=PROCTITLE msg=audit(1737100109.504:1626): proctitle=2F7573722F7362696E2F6E67696E78002D74```
```
3. Передаём эти события в утилиту ```audit2why```
```bash
cat /var/log/audit/audit.log | grep '1737100109.504:1626' | audit2why
```
4. Получаем описание того почему не работает порт ```4881``` и как решить эту проблему
```bash
type=AVC msg=audit(1737100109.504:1626): avc:  denied  { name_bind } for  pid=15552 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
5. Выполняем команду ```setsebool -P nis_enabled 1``` которой разрешаем использование NIS в системе при активированном SELinux
```bash
setsebool -P nis_enabled 1
```
6. Запускаем ```nginx```
```bash
systemctl start nginx
```
7. Проверяем что ```nginx``` работает и убеждаемся что сервис работает
```bash
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2025-01-17 10:58:00 MSK; 32s ago
    Process: 15645 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 15646 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 15647 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 15649 (nginx)
      Tasks: 3 (limit: 4668)
     Memory: 3.3M
        CPU: 191ms
     CGroup: /system.slice/nginx.service
             ├─15649 "nginx: master process /usr/sbin/nginx"
             ├─15650 "nginx: worker process"
             └─15651 "nginx: worker process"

Jan 17 10:58:00 localhost.localdomain systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jan 17 10:58:00 localhost.localdomain nginx[15646]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 17 10:58:00 localhost.localdomain nginx[15646]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 17 10:58:00 localhost.localdomain systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.
```
8. В браузере переходим по адресу ```ip_server:4881``` и проверяем что ```nginx``` возвращает страницу.

#### Разрешаем работу ```nginx``` с помощью добавления порта ```4881``` в имеющийся тип ```SELinux```
1. Проверим порт ```4881``` в существующем типе ```SELinux```
```bash
[root@localhost ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Видим, что порт ```4881``` в существующих типах отсутствует
2. Добавляем порт ```4881``` в существующий тип ```http_port_t```
```bash
[root@localhost ~]# semanage port -a -t http_port_t -p tcp 4881
```
3. Проверяем что порт ```4881``` успешно добавлен
```bash
[root@localhost ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
4. Запускаем ```nginx```
```bash
systemctl start nginx
```
5. Проверяем что ```nginx``` работает и убеждаемся что сервис работает
```bash
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2025-01-17 12:39:11 MSK; 3s ago
    Process: 11317 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 11318 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 11319 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 11320 (nginx)
      Tasks: 3 (limit: 4668)
     Memory: 3.3M
        CPU: 85ms
     CGroup: /system.slice/nginx.service
             ├─11320 "nginx: master process /usr/sbin/nginx"
             ├─11321 "nginx: worker process"
             └─11322 "nginx: worker process"

Jan 17 12:39:11 localhost.localdomain systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jan 17 12:39:11 localhost.localdomain nginx[11318]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 17 12:39:11 localhost.localdomain nginx[11318]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 17 12:39:11 localhost.localdomain systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.
```
6. В браузере переходим по адресу ```ip_server:4881``` и проверяем что ```nginx``` возвращает страницу.

#### Разрешаем работу ```nginx``` на нестандартном порту ```4881``` помощью формирования и установки модуля ```SELinux```
1. После попытки запуска сервиса ```nginx``` нужно проверить лог
```bash
[root@localhost ~]# grep nginx /var/log/audit/audit.log
```
2. На основе логов помощью утилиты ```audit2aloow``` сделаем модуль разрешающий работу ```nginx``` на нестандартном порту ```4881```
```bash
[root@localhost ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
3. Применяем сформированный модуль ```nginx.pp``
```bash
[root@localhost ~]# semodule -i nginx.pp
```
4. Запускаем ```nginx```
```bash
[root@localhost ~]# systemctl start nginx
```
5. Проверяем что сервис ```nginx``` работает
```bash
[root@localhost ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2025-01-17 13:23:57 MSK; 34s ago
    Process: 11361 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 11362 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 11363 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 11364 (nginx)
      Tasks: 3 (limit: 4668)
     Memory: 3.3M
        CPU: 148ms
     CGroup: /system.slice/nginx.service
             ├─11364 "nginx: master process /usr/sbin/nginx"
             ├─11365 "nginx: worker process"
             └─11366 "nginx: worker process"

Jan 17 13:23:57 localhost.localdomain systemd[1]: Starting nginx.service - The nginx HTTP and reverse proxy server...
Jan 17 13:23:57 localhost.localdomain nginx[11362]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 17 13:23:57 localhost.localdomain nginx[11362]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 17 13:23:57 localhost.localdomain systemd[1]: Started nginx.service - The nginx HTTP and reverse proxy server.
```

# Восстановление работы ```DNS```
1. На клиенте попробуем внести изменение в зону ```nsupdate -k /etc/named.zonetransfer.key```
```bash
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
```
Получаем ошибку
```bash
update failed: SERVFAIL
```
2. Смотрим логи ```SELinux```
```bash
root@client ~]# cat /var/log/audit/audit.log | audit2why
```
Ошибок нет
3. На машине ```ns01``` проверяем лог
```bash
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
```
Видим ошибку
```bash
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1734081216.004:1775): avc:  denied  { write } for  pid=7378 comm="isc-net-0001" name="dynamic" dev="sda4" ino=34048387 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0


	Was caused by:
		Missing type enforcement (TE) allow rule.


		You can use audit2allow to generate a loadable module to allow this access.
```
4. Посмотрим контекст локальной зоны
```bash
root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Oct  3 05:26 /var/named/named.localhost
```
5. Проверим контекст в папке ```/etc/named```
```bash
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_conf_t:s0       .
drwxr-xr-x. root root  system_u:object_r:named_conf_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_conf_t:s0   dynamic
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.newdns.lab
```
6. Изменим тип контекста безопасности для каталога ```/etc/named```
```bash
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# 
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
7. На клиенте попробуем внести изменение в зону ```nsupdate -k /etc/named.zonetransfer.key```
```bash
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
```
На этот раз изменения успешно внесены.