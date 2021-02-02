# SYSTEMD

## Введение

Копируем Vagrantfile в директорию на компьютере.

## Запуск тестового окружения

Открываем консоль, перейдим в директорию с проектом и выполнить `vagrant up`
```shell
vagrant up
```

## Подключение к серверу и переходим в директорию со скриптом

Для подключения к серверу необходимо выполнить
```shell
vagrant ssh centos
sudo -i
```

## 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig)

Создаем для нашего сервиса конфиг-файл следующего содержания:

```shell
[root@centos system]# cat /etc/sysconfig/watchlog
# Configuration file for my watchdog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Создаем лог-файл по пути /var/log/watchlog.log, заполняем его рандомными строками, а также словом "ALERT".

Создаем скрипт /opt/watchlog.sh для будущего сервиса:

```shell
[root@centos system]# cat /opt/watchlog.sh
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

Создаем service unit:

```shell
[root@centos system]# cat /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создаем timer unit:

```shell
[root@centos system]# cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second
[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
[Install]
WantedBy=multi-user.target
```

# Вывод
<details><summary>Пример вывода</summary>
<p>

```log
[root@centos system]# systemctl daemon-reload
[root@centos system]# systemctl start watchlog.service
[root@centos system]# systemctl start watchlog.timer
[root@centos system]# tail -f /var/log/messages
Feb  2 08:53:06 centos systemd: Started Run watchlog script every 30 second.
Feb  2 08:56:20 centos systemd: Starting Cleanup of Temporary Directories...
Feb  2 08:56:21 centos systemd: Started Cleanup of Temporary Directories.
Feb  2 09:01:01 centos systemd: Created slice User Slice of root.
Feb  2 09:01:01 centos systemd: Started Session 2 of user root.
Feb  2 09:01:01 centos systemd: Removed slice User Slice of root.
Feb  2 09:07:32 centos systemd: Reloading.
Feb  2 09:07:41 centos systemd: Starting My watchlog service...
Feb  2 09:07:41 centos root: Tue Feb  2 09:07:41 UTC 2021: I found word, Master!
Feb  2 09:07:41 centos systemd: Started My watchlog service.
```
</p>
</details>

## 2. Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi)

Устанавливаем spawn-fcgi и необходимые пакеты

```shell
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```

Необходимо раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi

```shell
sed -i 's/#SOCKET/SOCKET/' /etc/sysconfig/spawn-fcgi
sed -i 's/#OPTIONS/OPTIONS/' /etc/sysconfig/spawn-fcgi
```

Добавляем юнит

```shell
cp /vagrant/provision/spawn-fcgi.service /etc/systemd/system/spawn-fcgi.service
```

Включаем и стартуем

```shell
systemctl daemon-reload
systemctl enable spawn-fcgi
systemctl start spawn-fcgi
```

## 3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами

Устанавливаем httpd:

```shell
yum install httpd -y
```

Копируем юнит из шаблона

```shell
cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
```

Добавляем параметр для запуска нескольких экземпляров 

```shell
sed -i '/^EnvironmentFile/ s/$/-%I/' /etc/systemd/system/httpd@.service
echo "OPTIONS=-f conf/httpd-first.conf" > /etc/sysconfig/httpd-first
echo "OPTIONS=-f conf/httpd-second.conf" > /etc/sysconfig/httpd-second
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd-first.conf
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd-second.conf
mv /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.OLD
sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd-second.conf
sed -i '/ServerRoot "\/etc\/httpd"/a PidFile \/var\/run\/httpd-second.pid' /etc/httpd/conf/httpd-second.conf
```

Включаем и стартуем

```shell
systemctl disable httpd
systemctl daemon-reload
systemctl start httpd@first
systemctl start httpd@second
```