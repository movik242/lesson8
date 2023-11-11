# lesson8

Цель: Управление автозагрузкой сервисов происходит через systemd. Вместо cron'а тоже используется systemd. И много других возможностей. В ДЗ нужно написать свой systemd-unit.
1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig
2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно так же называться.
3. Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

При выполнении ДЗ использован Vagrant boxes файл системы Centos Stream 8 сборка 20230501.0

1. В установленной системе требуется отключить SElinux и firewalld

   Правлю файл /etc/selinux/config

          SElinux=disabled


   Проверяю, что firewalld сервис остановлен и отключен

          systemctl stop firewalld
          systemctl disable firewalld


   Создаю файл конфигурации для сервиса touch /etc/sysconfig/watchlog

          # Configuration file for my watchdog service
          # Place it to /etc/sysconfig
          # File and word in that file that we will be monit
          WORD="ALERT"
          LOG=/var/log/watchlog.log

   Создаю touch /var/log/watchlog.log и пишу туда строку ALERT

   Далее создаю скрипт /opt/watchlog.sh:

           #!/bin/bash
            WORD=$1
            LOG=$2
            DATE=`date`
            if grep $WORD $LOG &> /dev/null
            then
              logger "$DATE: I found word, Master!" # logger отправляет лог в системный журнал (/var/log/messages)
            else
              exit 0
            fi


  Создаю unit сервис, файл nano /etc/systemd/system/watchlog.service 

          [Unit]
          Description=My watchlog service

          [Service]
          Type=oneshot
          EnvironmentFile=/etc/sysconfig/watchlog
          ExecStart=/opt/watchlog.sh $WORD $LOG


  Создаю unit для таймера (timer файл), nano /etc/systemd/system/watchlog.timer

          [Unit]
          Description=Run watchlog script every 30 second

          [Timer]
          # Run every 30 second
          OnUnitActiveSec=30
          Unit=watchlog.service

          [Install]
          WantedBy=multi-user.target

 Даю права на запуск

         chmod +x /opt/watchlog.sh

 Стартую таймер

         systemctl start watchlog.timer

 Результат  

          tail -f /var/log/messages


![image](https://github.com/movik242/lesson8/assets/143793993/1cacfbfe-9ea8-4516-9062-3203bbdb74b0)


  2.Устанавливаю spawn-fcgi и необходимые для него пакеты:


          yum install epel-release -y && yum install spawn-fcgi php php-climod_fcgid httpd -y

  Захожу в /etc/sysconfig/spawn-fcgi и привожу его к следующему виду


          # You must set some working options before the "spawn-fcgi" service will work.
          # If SOCKET points to a file, then this file is cleaned up by the init script.
          #
          # See spawn-fcgi(1) for all possible options.
          #
          # Example :
          SOCKET=/var/run/php-fcgi.sock
          OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"


 Создаю unit файл nano /etc/systemd/system/spawn-fcgi.service

         [Unit]
         Description=Spawn-fcgi startup service by Movik
         After=network.target
         [Service]
         Type=simple
         PIDFile=/var/run/spawn-fcgi.pid
         EnvironmentFile=/etc/sysconfig/spawn-fcgi
         ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
         KillMode=process
         [Install]
         WantedBy=multi-user.target

  ![image](https://github.com/movik242/lesson8/assets/143793993/c7de13d0-792f-4f2f-8157-0d219d5e177e)

  
  3. Для запуска нескольких экземпляров сервиса httpd будем использовать шаблон в конфигурации файла окружения


  Копирую файл из /usr/lib/systemd/system/, cp /usr/lib/systemd/system/httpd.service /etc/systemd/system, далее переименовываю 
  mv /etc/systemd/system/httpd.service/etc/systemd/system/httpd@.service и привожу к виду


          [Unit]
          Description=The Apache HTTP Server
          After=network.target remote-fs.target nss-lookup.target
          Documentation=man:httpd(8)
          Documentation=man:apachectl(8)
          
          [Service]
          Type=notify
          EnvironmentFile=/etc/sysconfig/httpd-%I #(%I - это и есть подстановка)
          ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
          ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
          ExecStop=/bin/kill -WINCH ${MAINPID}
          KillSignal=SIGCONT
          PrivateTmp=true

          [Install]
          WantedBy=multi-user.target


  Задаю опции для запуска веб-сервера с необходимым конфигурационным файлом


        # /etc/sysconfig/httpd-first
        OPTIONS=-f conf/first.conf

        # /etc/sysconfig/httpd-second
        OPTIONS=-f conf/second.conf


  Копирую файл httpd.conf с именами first.conf и second.conf

        cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf
        cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf

  Правлю файл second.conf, добавляю параметр 
        
        PidFile /var/run/httpd-second.pid
        Listen 8999

  Запускаю

        systemctl start httpd@first
        systemctl start httpd@second


![image](https://github.com/movik242/lesson8/assets/143793993/2be87720-606a-4d60-97bc-dfd6e821597c)

  Проверяю


  ![image](https://github.com/movik242/lesson8/assets/143793993/caa52f8f-6a0c-4077-9721-63b7a727afcd)

  


  




   
