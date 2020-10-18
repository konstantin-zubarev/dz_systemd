#### Стенд для занятия с Systemdсоздаём файл с конфигурацией.

Цель:
- 1. Написать сервис, который будет раз в 30 секунд мониторить лог на
предмет наличия ключевого слова. Файл и слово должны
задаваться в /etc/sysconfig
- 2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл.
Имя сервиса должно называться также
- 3. Дополнить юнит-файл apache httpd возможностью запустить
несколько инстансов сервера с разными конфигами
- 4*. Скачать демо-версию Atlassian Jira и переписать основной скрипт
запуска на unit-файл

1. Servers time

Создаём файл с конфигурацией для сервиса в директории `/etc/sysconfig`
```
# vi /etc/sysconfig/watchlog
```
```
# Configuration file for my watchdog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```
Затем создадим лог файл
```
# vi /var/log/watchlog.log
```
```
var
dir
echo
ALERT
proc
etc
bin
cat
```
Создадим скрипт:
```
# vi /opt/watchlog.sh
```
```
#!/bin/bash

WORD=$1
LOG=$2
DATE=$(date)

if grep $WORD $LOG &> /dev/null
then
	logger "$DATE: I found word, Master!"
else
	exit 0
fi
```
```
# chmod +x /opt/watchlog.sh
```

Создадим юнит для сервиса:

```
# vi /etc/systemd/system/watchlog.service
```
```
[Unit]
Description=My wathlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создадим юнит для таймера:

```
# vi /etc/systemd/system/watchlog.timer
```
```
[Unit]
Description=Run watchlog script every 30 second

[Timer]
#Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
AccuracySec=1us

[Install]
WantedBy=multi-user.target
```

Запустим сервис, таймер и проверим результат:

```
# systemctl daemon-reload
# systemctl start watchlog.service
# systemctl start watchlog.timer
# tail -f /var/log/messages
```
```
Oct 18 16:22:21 localhost systemd: Started My wathlog service.
Oct 18 16:22:51 localhost systemd: Starting My wathlog service...
Oct 18 16:22:51 localhost root: Sun Oct 18 16:22:51 UTC 2020: I found word, Master!
Oct 18 16:22:51 localhost systemd: Started My wathlog service.
Oct 18 16:22:53 localhost systemd: Starting My wathlog service...
Oct 18 16:22:53 localhost root: Sun Oct 18 16:22:53 UTC 2020: I found word, Master!
Oct 18 16:22:53 localhost systemd: Started My wathlog service.
Oct 18 16:23:23 localhost systemd: Starting My wathlog service...
Oct 18 16:23:23 localhost root: Sun Oct 18 16:23:23 UTC 2020: I found word, Master!
Oct 18 16:23:23 localhost systemd: Started My wathlog service.

```

2. Устанавливаем spawn-fcgi и необходимые для него пакеты:

```
# yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```
Раскомментировать строки в `/etc/sysconfig/spawn-fcgi`

Приведем к следующему виду
```
# cat /etc/sysconfig/spawn-fcgi
```
```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

Создадим юнит для spawn-fcgi
```
# vi /etc/systemd/system/spawn-fcgi.service
```
```
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Проверим
```
# systemctl start spawn-fcgi
# systemctl status spawn-fcgi
```
```
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-10-18 16:26:32 UTC; 8s ago
 Main PID: 3576 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─3576 /usr/bin/php-cgi
           ├─3577 /usr/bin/php-cgi
           ├─3578 /usr/bin/php-cgi
           ├─3579 /usr/bin/php-cgi
           ├─3580 /usr/bin/php-cgi
           ├─3581 /usr/bin/php-cgi
           ├─3582 /usr/bin/php-cgi
           ├─3583 /usr/bin/php-cgi
           ├─3584 /usr/bin/php-cgi
           ├─3585 /usr/bin/php-cgi
           ├─3586 /usr/bin/php-cgi
           ├─3587 /usr/bin/php-cgi
           ├─3588 /usr/bin/php-cgi
           ├─3589 /usr/bin/php-cgi
           ├─3590 /usr/bin/php-cgi
           ├─3591 /usr/bin/php-cgi
           ├─3592 /usr/bin/php-cgi
           ├─3593 /usr/bin/php-cgi
           ├─3594 /usr/bin/php-cgi
           ├─3595 /usr/bin/php-cgi
           ├─3596 /usr/bin/php-cgi
           ├─3597 /usr/bin/php-cgi
           ├─3598 /usr/bin/php-cgi
           ├─3599 /usr/bin/php-cgi
           ├─3600 /usr/bin/php-cgi
           ├─3601 /usr/bin/php-cgi
           ├─3602 /usr/bin/php-cgi
           ├─3603 /usr/bin/php-cgi
           ├─3604 /usr/bin/php-cgi
           ├─3605 /usr/bin/php-cgi
           ├─3606 /usr/bin/php-cgi
           ├─3607 /usr/bin/php-cgi
           └─3608 /usr/bin/php-cgi

```

3. Юнит-файл apache httpd

Установим apache

```
# yum install httpd -y
```

Создадим юнит файл для httpd

```
# vi /etc/systemd/system/httpd@.service
```
```
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
Зададим параметры
```
# vi /etc/sysconfig/httpd-first
```
```
OPTIONS=-f conf/first.conf
```
```
# vi /etc/sysconfig/httpd-second
```
```
OPTIONS=-f conf/second.conf
```
Соответственно в директории с конфигами httpd должны лежать два конфига, в нашем случае это будут first.conf и second.conf

Теперь можно запустить экземпляры сервиса:
```
# systemctl start httpd@first
# systemctl start httpd@second
```
Проверим порты:
```
ss -tnulp | grep httpd
```
```
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=4089,fd=4),("httpd",pid=4088,fd=4),("httpd",pid=4087,fd=4),("httpd",pid=4086,fd=4),("httpd",pid=4085,fd=4),("httpd",pid=4084,fd=4),("httpd",pid=4083,fd=4))
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=4076,fd=4),("httpd",pid=4075,fd=4),("httpd",pid=4074,fd=4),("httpd",pid=4073,fd=4),("httpd",pid=4072,fd=4),("httpd",pid=4071,fd=4),("httpd",pid=4070,fd=4))
```
