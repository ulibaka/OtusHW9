### Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig).
----

#### создаём файл с конфигурацией
```
# cat << _EOF_ > /etc/sysconfig/watchlog
> # Configuration file for my watchdog service
> # Place it to /etc/sysconfig
> # File and word in that file that we will be monit
> WORD="ALERT"
> LOG=/var/log/watchlog.log
> _EOF_
```
#### наполняем журнал /var/log/watchlog.log текстом и искомым словом ALERT

```
# cat << _EOF_ > /var/log/watchlog.log
> aserhja
> fhads
> fghasfdg
> ALERT
> dfshfhsfhdx
> dsaf
> ghsdfh
> _EOF_
```

#### Создадим скрипт и сделаем его исполняемым 
```
# cat << _EOF_ > /opt/watchlog.sh
> #!/bin/bash
> WORD=$1
> LOG=$2
> DATE=`date`
> if grep $WORD $LOG &> /dev/null
> then
> logger "$DATE: I found word, Master!"
> else
> exit 0
> fi
> _EOF_

chmod +x /opt/watchlog.sh
```

#### Создадим юнит для сервиса
```
cat << _EOF_ > /etc/systemd/system/watchlog.service
> [Unit]
> Description=My watchlog service
> [Service]
> Type=oneshot
> EnvironmentFile=/etc/sysconfig/watchlog
> ExecStart=/opt/watchlog.sh $WORD $LOG
> [Install]
> WantedBy=multi-user.target
> _EOF_
```

#### Создадим юнит для таймера
```
# cat << _EOF_ > /etc/systemd/system/watchlog.timer
> [Unit]
> Description=Run watchlog script every 30 second
> [Timer]
> # Run every 30 second
> OnUnitActiveSec=30
> Unit=watchlog.service
> [Install]
> WantedBy=multi-user.target
> _EOF_
```

#### Стартуем таймер и проверяем результат:
```
# systemctl daemon-reload
# systemctl start watchlog.timer
```
```
# tail /var/log/messages | grep -i master
Sep  9 18:36:29 localhost root: Fri Sep  9 18:36:29 UTC 2022: I found word, Master!
Sep  9 18:37:39 localhost root: Fri Sep  9 18:37:39 UTC 2022: I found word, Master!
Sep  9 18:38:39 localhost root: Fri Sep  9 18:38:39 UTC 2022: I found word, Master!
```

### Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.

#### Устанавливаем spawn-fcgi и необходимые для него пакеты
```
#  yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```

### Открываем строки с SOCKET и OPTION
```
# sed -i 's/#SOCKET/SOCKET/g' /etc/sysconfig/spawn-fcgi && sed -i 's/#OPTIONS/OPTIONS/g' /etc/sysconfig/spawn-fcgi
```

### Создаем UNIT
```
# cat << _EOF_ > /etc/systemd/system/spawn-fcgi.service
> [Unit]
> Description=Spawn-fcgi startup service by Otus
> After=network.target
> [Service]
> Type=simple
> PIDFile=/var/run/php-fcgi.pid
> EnvironmentFile=/etc/sysconfig/spawn-fcgi
> ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
> KillMode=process
> [Install]
> WantedBy=multi-user.target
> _EOF_
```

### Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

#### Создание инстансов
```
# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf && cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf 
```

#### Редактирование конфигов 
```
# sed -i '32s;^;PidFile /var/run/httpd-first.pid;' /etc/httpd/conf/first.conf && sed -i 's/Listen\ 80/Listen\ 81/g' /etc/httpd/conf/first.conf
# sed -i '32s;^;PidFile /var/run/httpd-second.pid;' /etc/httpd/conf/second.conf && sed -i 's/Listen\ 80/Listen\ 8080/g' /etc/httpd/conf/second.conf
```
#### Задаем опции для запуска веб-сервера с необходимым конфигурационным файлом
```
# cat << _EOF_ > /etc/sysconfig/httpd-first
> OPTIONS="-f conf/first.conf"
> _EOF_
```
и
```
# cat << _EOF_ > /etc/sysconfig/httpd-second
> OPTIONS="-f conf/second.conf"
> _EOF_
```

#### Создаем доп. юниты

```
# cp /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/first.service && cp /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/second.service

# sed -i 's/\/etc\/sysconfig\/httpd/\/etc\/sysconfig\/httpd-first/g' /usr/lib/systemd/system/first.service && sed -i 's/\/etc\/sysconfig\/httpd/\/etc\/sysconfig\/httpd-second/g' /usr/lib/systemd/system/second.service
```

#### Перезапускаем демона, запускаем units и проверяем
```
# systemctl daemon-reload
# systemctl start first.service && systemctl start second.service 
# ss -tnulp | grep httpd
 

```
