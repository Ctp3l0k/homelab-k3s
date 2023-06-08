﻿# Ручная установка HA-кластера k3s с MySQL как БД и HAProxy как балансировщиками

## Вступление

За основу было взято [видео](https://www.youtube.com/watch?v=APsZJbnluXg) от ютубера Techno Tim. В том, что касается непосредственно k3s, данный текст полностью повторяет руководство Тима. Однако в видео ни MySQL, ни используемый балансировщик никак не резервируется, что не очень-то и соответствует определению High Availability. 

Также отмечу, что руководство Тима было создано в те времена, когда k3s не позволял использовать etcd в качестве базы для HA, и именно поэтому в нем рассказывается про вариант с MySQL. Сейчас было бы правильней именно etcd на серверах k3s, что позволит избавиться от как минимум четырех лишних VM. Делается это с помощью ключа --cluster-init при установке первого сервера. А еще ничто не мешает установить keepalived прямо на сервера, избавившись от необходимости в отдельных балансировщиках.

Начинал писать данный текст как шпаргалку для себя, но в процессе решил облагородить и сделать репозиторий публичным.

## Как устроен кластер

![](https://docs.k3s.io/img/k3s-architecture-ha-external.svg)Изображение из официального [руководства](https://docs.k3s.io/architecture) для общего понимания. У меня схема реализована несколько иначе.

1) Сервера k3s, они же Server Nodes. Управляют агентами. Для HA нужно как минимум два.
2) Агенты k3s, они же Agent Nodes. Рабочая нагрузка приходится непосредственно на них. В нашем случае их может быть от одного до бесконечности.
3) Балансировщики серверов. Здесь у меня первое отличие и от официальной схемы, и от варианта Тима. В моём случае балансирощиков от двух штук, они резервируют друг друга, а через них к серверу подключается не только пользователь, но и ноды.
4) База данных, а конкретно MySQL. И у Тима, и на схеме БД всего одна. В моём случае их две.
5) Балансировщики БД. Поскольку HA подразумевает под собой отсутствие ручного переключения БД на резервную в случае падения основной, подключаться к ней нужно через балансировщик. Также балансировщик нужно дублировать, поэтому у меня их два.

## На чем всё это разворачиваем

Для развёртывания я использовал свой сервер Proxmox. Балансировщики и сервера MySQL запускал в LXC-контейнерах с Ubuntu 20.04 от Proxmox, непосредственно k3s в виртуальных машинах с [Ubuntu Cloud Image 20.04](https://cloud-images.ubuntu.com/releases/focal/release/). Ресурсов выделял минимум: по одному ядру и по гигабайту оперативной памяти. Swap и IPv6 везде отключены, все работы проводил от имени пользователя root. Также у меня настроен DNS-сервер, а все хосты общаются между собой по доменным именам. Повторять это не обязательно, ничто не мешает использовать в конфигах IP.

## Подготовка балансировщиков

Поскольку все участники кластера общаются между собой через балансировщики, начнем с них. Создадим 4 сервера:

 - k3s-balancer-01
 - k3s-balancer-02
 - k3s-db-balancer-01
 - k3s-db-balancer-02

На каждом из них нам потребуется по два приложения: keepalived и HAProxy.

```
apt update
apt install haproxy keepalived -y
```

Затем идем в файлы этого репозитория, и складываем соответствующие конфиги в соответствующие директории соответствующих серверов: haproxy.cfg в /etc/haproxy (перезаписываем оригинальный файл и при желании удаляем всё остальное), keepalived.conf в /etc/keepalived.

Теперь нужно либо внести записи "k3s-server-01.lab.lan", "k3s-server-02.lab.lan", "k3s-db-01.lab.lan" и "k3s-db-02.lab.lan" в DNS-сервер, либо поменять данные адреса в конфигах на IP, которые будем использовать на данных серверах. У HAProxy есть неприятная особенность: если он не может зарезолвить адрес, указанный в конфиге, то не запустится.

Также в пункте virtual_ipaddress в конфигах keepalived указаны виртуальные IP, по одному на каждую пару балансировщиков. Их также нужно поменять на свой свой адреса и, в случае использования DNS, назначить A-записи для k3s-balancer.lab.lan и k3s-db-balancer.lab.lan.

После того, как это сделано, запускаем данные приложения.

    systemctl enable --now keepalived
    systemctl enable --now haproxy

Поскольку в Ubuntu 20.04 брандмауэр изначально выключен, на этом с балансировщиками всё.    

  ## Конфигурация балансировщиков
  
  ### keepalived
  
keepalived - приложение, которое позволяет назначить серверу плавающий IP-адрес, который перейдет на другой сервер в случае недоступности основного. Или же, как это сделано в данном конфиге, при недоступности приложения.

У всех четырех запущенных keepalived конфигурация почти не отличается, поэтому для примера разберем только один из них.

```
vrrp_script check_haproxy_stats {
        script "/usr/bin/curl 127.0.0.1:9000/stats"
        interval 2
}

vrrp_instance k3s-balancer {
        state MASTER
        interface eth0
        virtual_router_id 3
        priority 100
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass k3s
        }
        virtual_ipaddress {
              192.168.11.241/24
        }
        track_script {
              check_haproxy_stats
        }
}
```

Первый блок, vrrp_script, описывает проверку доступности HAProxy, а конкретно его страницы со статистикой. Каждые две секунды работает curl, если он получает ответ, то проверка проходит, если нет, значит keepalived считает, что HAProxy лежит. Проверка простая как палка, а еще сама по себе ни на что не влияет, влияение включается ниже по конфигу.

Блок vrrp_instance - это непосредственно то, что описывает переключение адреса.

 - state MASTER или BACKUP - изначальное состояние при запуске приложения, которое затем перекрывается параметром priority.
 - interface eth0 - здесь нужно указать, какой именно сетевой интерфейс необходимо использовать и для назначения плавающего адреса, и для отслеживания состояния второго keepalived. Интерфейсы одной группы keepalived при данной конфигурации должны быть в одном L2-сегменте (а еще точнее между ними должен ходить multicast-трафик). Ничего страшного, если на данном интерфейсе уже висит родной адрес.
 - virtual_router_id 3 - ID должен быть у каждой **пары** балансировщиков свой совпадающий.
- priority 100 - сервер с большим приоритетом получает статус MASTER.
- advert_int 1 - отправляем статус этого балансировщика второму раз в секунду.
- authentication - пока готовил этот текст, обнаружил, что сами разработчики не рекомендуют использовать данный параметр. Менять работающий конфиг не буду, но отметку сделаю.
- virtual_ipaddress - собственно, плавающий адрес, к которому будут обращаться члены кластера.
- track_script - указывает, что при переключении адресов нужно смотреть не только на доступность второго балансировщика, но и на результаты проверки, которая была описана в первом блоке конфигурации.

### HAProxy балансировщика серверов

HAProxy в нашем случае используется в качестве L4-прокси, т.е. прокси, пропускающего через себя весь трафик, не заглядывая внутрь. В нашем случае это нужно для балансировки, т.е. динамического распределения трафика по серверам, до которых он должен дойти.

Т.к. у балансирощиков k3s-серверов и баз данных несколько разные настройки, разберем их по отдельности. Начнем с балансировщика серверов:

```
listen stats
  bind 127.0.0.1:9000
  mode http
  stats enable
  stats hide-version
  stats refresh 10s
  stats show-node
  stats uri /stats

frontend k3s
  bind *:6443
  use_backend k3s

backend k3s
  mode tcp
  option ssl-hello-chk
  server k3s-server-01 k3s-server-01.lab.lan:6443 check 
  server k3s-server-02 k3s-server-02.lab.lan:6443 check backup
```

Первый блок, listen с названием stats, включает страницу HAProxy с технической информацией. Именно доступность этой страницы проверяет keepalived, а еще она незаменима при дебаге и проверки конфигурации.

- bind 127.0.0.1:9000 - поменяйте на "*:9000", если хотите открыть данную страницу у себя в браузере. Порт может быть любым, разве что у меня данная страница не заработала на 80-м.
- mode http и stats enable - необходимы для технической страницы.
- stats hide-version - скрывает версию HAProxy с технической страницы, чтобы у злоумышленников было меньше информации. Не нужно, если держать страницу доступной только локалхосту.
- stats refresh 10s - делает так, чтобы открытая страница обновлялась каждые 10 секунд. Не нужно, если держать страницу доступной только локалхосту.
- stats show-node - показывает на странице имя сервера, на котором запущен HAProxy. Незаменимо при конфигурации keepalived, не нужно, если держать страницу доступной только локалхосту.
- stats uri /stats - путь, по которому страница доступна. В данном случае она находится по пути 127.0.0.1:9000/stats.

Второй блок, frontend, описывает то, что касается входящих подключений. В нашем случае это:

- k3s - название фронтенда.
- bind *:6443 - слушать порт 6443 на всех адресах.
- use_backend k3s - для данного фронтенда использовать бэкенд с названием k3s. Наверное, стоило дать фронту и бэку разные названия, но теперь уже не буду менять рабочий конфиг.

Третий блок, backend, описывает, что делать с входящим трафиком:

- k3s - название бэкенда.
- mode tcp - использовать режим L4-прокси, т.е. перенаправлять весь трафик, не заглядывая внутрь.
- option ssl-hello-chk - при использовании check (см. ниже) проверять доступность SSL на стороне сервера-получателя. k3s использует для работы https на порту 6443. HAProxy при данной конфигурации не проверяет доступность всего хоста и не смотрит, что находится за этим портом. На нем есть https - сервер работает. Нет - помечаем сервер как упавший.
- server k3s-server-01 k3s-server-01.lan.lan:6443 check - первый из двух наших серверов приложений. Сначала идёт название в рамках HAProxy - k3s-server-01. Оно может быть любым. Затем адрес сервера и используемый порт. Напомню, что здесь вам либо нужно вписать свой IP-адрес, либо иметь соответствующую запись в DNS. check - говорим HAProxy проверять, доступен ли данный сервер или нет.
- server k3s-server-02 k3s-server-02.lab.lan:6443 check backup - всё то же самое, но второй сервер мы отметили как резервный. Соединения с ним не будут устанавливаться, пока активен основной.

Отмечу, что в моём конфиге не указаны таймауты для соединений. Из-за этого HAProxy будет ругаться при запуске, но всё будет работать. Таймаут по умолчанию - две секунды. Т.е. при падении первого сервера переключение на второй произойдет через две секунды.

  ### HAProxy балансировщика баз данных

Первые два блока (техническая страница и фронтенд) у балансировщиков БД такие же, как у балансировщиков серверов. Однако в третьем блоке есть небольшие, но очень важные отличия.

```
backend k3s-db
  mode tcp
  option mysql-check user haproxy-check
  server k3s-db-01 k3s-db-01.lab.lan:3306 check on-marked-up shutdown-backup-sessions rise 30 
  server k3s-db-02 k3s-db-02.lab.lan:3306 check backup rise 30
```

- option mysql-check user haproxy-check - вместо проверки наличия SSL мы проверяем доступность MySQL. Конкретно HAProxy пытается подключиться к серверу БД и если видит любой ответ, считает, что сервер жив. Однако сервер MySQL не любит, когда к нему активно пытаются подключиться без успеха, и банит такие подключения по IP. Поэтому мы указываем для проверки пользователя haproxy-check, создание которого описано в разделе, посвященном MySQL-серверу.
- server k3s-db-01 k3s-db-01.lab.lan:3306 check on-marked-up shutdown-backup-sessions rise 30 - первые два новых пункта обозначают, что когда данный основной сервер поднимается, балансировщик обрывает соединения со всеми серверами, отмеченными как backup. Это очень важно, т.к. сервера MySQL у нас работают в режиме master-master, и именно эта настройка значительно уменьшит вероятность сплит-брейна. Последняя новая опция, rise 30, говорит отмечать сервер работающим только после 30 успешных проверок (т.е. одной минуты), чтобы дать время поднявшемуся серверу синхронизировать изменения.

Важно отметить, что такая настройка балансировщика не даёт гаранированной защиты от расхождения баз при использовании репликации master-master. Вполне возможна ситуация, когда в момент переключения базы одна из них отстаёт от другой, или при обрыве соединения транзакция не будет завершена. При использовании больших баз не стоит использовать данное автоматическое переключение туда-обратно. Однако в ситуации домашнего k3s, когда используется мелкая база, я считаю использование подобной схемы оправданным.

## Подготовка серверов БД

Т.к. мы не сможем развернуть наш кластер k3s без базы данных, следующим шагом организовываем серверы для неё. Поднимаем две машины:

 - k3s-db-01
 - k3s-db-02

На обеих из них нам нужно установить сервера MySQL. Устанавливаем и сразу запускаем:

```
apt update
apt install mysql-server -y
systemctl enable --now mysql
```

На момент написания данного текста после выполнения этих команд будет установлен и запущен сервер MySQL 8.0.

На данном этапе я предпочитаю использовать утилиту mysql_secure_installation для первоначальной конфигурации. Но вот проблема: у Ubuntu пользователь root по умолчанию может авторизироваться в MySQL только через сокет и без пароля. Из-за этого mysql_secure_installation будет бесконечно ругаться, а единственным способом его закрыть будет разрыв SSH-соединения.

    mysql --execute="ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';"

Выполнив данную команду, мы зададим пользователю root пароль password. Можете не вводить свой пароль сейчас, ведь мы его поменяем следующим же действием.

```
mysql_secure_installation
```

Утилита спросит пароль от root, который мы только что задали. Вводим его, и получаем следующие вопросы:

 1. Устанавливать ли VALIDATE PASSWORD COMPONENT, т.е. проверку надежности паролей. Я предпочитаю её не ставить, поэтому отвечаю N.
 2. Поменять ли пароль пользователя root.
 3. Удалить ли возможность подключаться к серверу без учетной записи. Отвечаем Y.
 4. Отключить ли возможность подключаться от имени root по сети. Если такая возможность вдруг потребуется, её всегда можно включить. К тому же, MySQL-клиенты умеют подключаться через SSH-тоннель, что для сервера будет будто локальное подключение. Отвечаем Y.
 5. Удалить ли стандартную тестовую базу данных. Y.
 6. Применить ли все изменения прямо сейчас. Y.

И еще небольшое замечание по поводу пароля рута. Если вам ближе схема по умолчанию, т.е. подключение без пароля, то можно вернуть стандартную настройку вот этой командой:

    mysql -p --execute="ALTER USER 'root'@'localhost' IDENTIFIED WITH auth_socket;"

Теперь идём в файл /etc/mysql/mysql.conf.d/mysqld.cnf и первым делом правим

    bind-address  =  127.0.0.1
на

    bind-address  =  0.0.0.0
   
Таким образом при перезапуске сервера он будет доступен не только локально, но и по сети.

Также нам нужно включить возможность репликации. Часть следующих параметров уже есть в конце файла в закомментированном виде, поэтому я их раскомментировал и новые параметры написал рядом, но ничего не мешает прописать их в любом месте файла. Или просто возьмите конфиги из этого репозитория.

```
server-id  =  1
log_bin  =  /var/log/mysql/mysql-bin.log
relay-log  =  /var/log/mysql/mysql-relay-bin.log
max_binlog_size  =  100M
log_slave_updates  =  1
```

 - server-id  - не забудьте поменять на 2 у второго сервера.
 - log_bin - бинарные логи, необходимые для любой репликации.
 - relay-log - бинарный лог, полученный с мастера и необходимый для принимающего сервера. Т.к. у нас схема master-master, данный лог необходим на обоих серверах.
 - log_slave_updates - необходимо включить при последовательной репликации, как у нас. Делает так, чтобы полученные от мастера изменения вносились в бинарный лог реплики.

replicate-do-db указывать не будем, т.к. будем реплицировать всё содержимое сервера. binlog_expire_logs_seconds тоже - в MySQL 8.0, в отличие от более старых версий, по умолчанию логи живут всего 30 дней, а не бесконечность.

Перезапускаем сервер для применения изменений:

    systemctl restart mysql.service

Теперь нам нужно запустить нашу репликацию master-master. Еще раз отмечу, что делать так при больших нагрузках на базу - **плохая идея**. Несмотря на все предосторожности, которые я описал в разделе о HAProxy, риск расхождения информации в базах присутствует. Для подобных вещей в боевых условиях стоит использовать специализированные решения вроде [MySQL orchestrator](https://github.com/openark/orchestrator) и [ProxySQL](https://proxysql.com), а не велосипеды на HAProxy. Однако в нашем случае мы продолжаем.

mysql без аргументов используем, если вернули настройку по умолчанию и root входит по сокету без пароля. Если же вы это делать не стали, то используем "mysql -p" и вводим пароль при запросе.

На обоих серверах вводим команды:

```
mysql
CREATE USER 'slave-user'@'%' IDENTIFIED BY 'RandomPassword';
GRANT REPLICATION SLAVE ON *.* TO 'slave-user'@'%';
FLUSH PRIVILEGES;
exit;
```

Обратите внимание, что я разрешил подключение пользователя для репликации с любого адреса. По-хорошему, вместо '%' нужно было использовать адрес второго сервера из пары, но ¯\\_(ツ)_/¯

Выполняем команду на первом сервере

    mysql --execute="SHOW MASTER STATUS;"

Получаем вывод, показывающий необходимые нам параметры для запуска реплики: file и position. Вписываем их в команду, которую выполняем на втором сервере:

```
mysql
CHANGE MASTER TO master_host='k3s-db-01.lab.lan', master_port=3306, master_user='slave-user', master_password='RandomPassword', master_log_file='mysql-bin.000002', master_log_pos=869, GET_MASTER_PUBLIC_KEY=1;
START SLAVE;
exit;
```

Репликация со стороны первого сервера во второй запущена. Проверим её уже после того, как запустим и в обратную сторону, а пока уточним пару вещей:

 - Напомню лишний раз, что k3s-db-01.lab.lan - это адрес, за который отвечает отдельный DNS-сервер. Вам ничто не мешает здесь вписать IP-адрес второго сервера БД.
 - GET_MASTER_PUBLIC_KEY=1; - включает возможность аутентифицироваться реплике с помощью типа аутентификации caching_sha2_password, который в MySQL 8.0 используется по умолчанию.

Теперь повторяем всё в обратную сторону, т.е. сначала получаем статус мастера на втором сервере:

    mysql --execute="SHOW MASTER STATUS;"

А затем с полученными данными подключаем первый как реплику:

```
mysql
CHANGE MASTER TO master_host='k3s-db-02.lab.lan', master_port=3306, master_user='slave-user', master_password='RandomPassword', master_log_file='mysql-bin.000002', master_log_pos=986, GET_MASTER_PUBLIC_KEY=1;
START SLAVE;
exit;
```

Запросим статус репликации на обоих серверах:

    mysql --execute="SHOW SLAVE STATUS\G"

Slave_IO_Running? Yes. Slave_IO_State? Waiting for source to send event. Базы реплицируются в обе стороны. Можно было бы выполнить тестовый запрос и убедиться, но я просто буду выполнять дальнейшую настройку попеременно на обоих серверах, проверяя, выполнились ли они на паре.

Теперь нам нужен пользователь, с помощью которого HAProxy будет проверять доступность наших серверов. Команду можно выполнить на любом из них, я это сделаю на втором:

    mysql --execute="CREATE USER 'haproxy-check'@'%' IDENTIFIED WITH mysql_native_password;"

После выполнения команды у нас будет создан пользователь с именем haproxy-check. Как и раньше, правильней было бы ограничить его адресами наших балансировщиков, но я этого делать не буду. HAProxy не умеет аутентифицироваться с паролем, поэтому пользователь должен быть без него. Также HAProxy не умеет использовать стандартный в MySQL 8.0 аутентификатор caching_sha2_password, поэтому мы задаём использование старого mysql_native_password.

На всякий случай убедимся, что пользователь прилетел и на первый сервер, используя на нем команду:

    mysql --execute="SELECT user FROM mysql.user;"

Также, как я упоминал выше, HAProxy постоянно проверяет доступность MySQL, и наши свежесозданные сервера наверняка уже забанили наши балансировщики. Для того, чтобы их разбанить, выполним на каждом из серверов БД команду:

    mysqladmin flush-hosts

Также нам нужно создать базу данных, которую будет использовать k3s. Как и в прошлый раз, команды можно выполнить на любом из серверов. Я сделаю это на первом:

```
mysql
CREATE DATABASE k3s;
CREATE USER 'k3s'@'%' IDENTIFIED BY 'RandomPassword';
GRANT ALL PRIVILEGES ON k3s.* TO 'k3s'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit;
```

Как и в прошлые разы, правильней было бы разрешать подключения к базе не откуда угодно, а исключительно с реальных адресов **балансировщиков БД**, не с серверов k3s и не с плавающих адресов.

На всякий случай убедимся, что данная база прилетела и на второй сервер:

    mysql --execute="SHOW DATABASES;"

С серверами БД мы закончили.

## Дальше живут драконы

Если до этого момента я понимал, как всё работает, и в целом использовал свои знания, то дальше по тексту начинается неизвестная мне магия, и поэтому всё, что написано ниже - это адаптация руководства Тима без понимания сути. Однако весь кластер был развернут именно для того, чтобы её понять, поэтому вполне может быть, что на момент прочтения вами этого отступления я уже со всем разобрался.

## Запуск самого k3s

### Серверы

Создадим две машины:

 - k3s-server-01
 - k3s-server-02

На обоих серверах выполняем следующую команду, чтобы задать параметры подключения к нашей базе данных.:

```
export K3S_DATASTORE_ENDPOINT='mysql://k3s:RandomPassword@tcp(k3s-db-balancer.lab.lan:3306)/k3s'
```

 - Первый k3s - имя пользователя, которого мы создали в последнем пункте раздела про БД.
 - RandomPassword - его пароль. Само собой, если вы использовали другой пароль, вам нужно будет поменять его в данной команде.
 - k3s-db-balancer.lab.lan - это не адрес конкретного балансировщика, это **плавающий адрес**, который мы организовали с помощью keepalived. Также снова напомню, что у меня настроен отдельный DNS-сервер, и если у вас его нет, вам нужно указать IP.
 - Второй k3s - имя нашей базы, которую мы также создали 
в последнем пункте раздела про БД.

Теперь на первом сервере выполним команду:

    curl -sfL https://get.k3s.io | sh -s - server --node-taint CriticalAddonsOnly=true:NoExecute --tls-san k3s-balancer.lab.lan

 - Утилита curl, которая включена в Ubuntu 20.04 (а если нет, то "apt install curl -y") в тихом режиме без лишних сообщений скачивает скрипт установки k3s и запускает его с ключами.
 - server - собственно, мы устанавливаем сервер, а не агент.
 - node-taint CriticalAddonsOnly=true:NoExecute - указываем, что на данном сервере не должны работать поды, т.е. контейнеры с запущенными приложениями. Сервер с данным параметром должен исключительно управлять агентами. У k3s есть параметр affinity, который "привлекает" поды на данный конкретный хост, и есть параметр taint, который наоборот не даёт им начать работать на хосте. В данном случае некие "критические аддоны" могут работать на сервере, а любые другие поды нет.
 - tls-san k3s-balancer.lab.lan - вот здесь должен быть плавающий адрес нашего балансировщика серверов. k3s создаёт tsl-сертификат для подключения агентов. Т.к. мы агентов будем подключать не напрямую, а через балансировщик, сертификат должен быть создан в том числе и на адрес балансировщика.

Подождем пару минут, пока сервер запустится, и затем еще пару, пока он выполнит свои внутренние дела. Убедимся, что сервер работает:

    k3s kubectl get nodes

Мы должны получить список хостов, состоящий из одного-единственного нашего сервера. Если у вас при проверке всплывают ошибки с текстом "couldn't get resource list for metrics.k8s.io/v1beta1", подождите еще немного.

Теперь нам нужно узнать токен данного сервера:

    cat /var/lib/rancher/k3s/server/node-token

Скопируем его. Если вы применяли настройки подключения к БД в этой же SSH-сессии, просто выполните следующую команду. Если нет, снова выполните "export K3S_DATASTORE..." из предыдущего пункта, а затем команду:

    curl -sfL https://get.k3s.io | sh -s - server --node-taint CriticalAddonsOnly=true:NoExecute --tls-san k3s-balancer.lab.lan --token=K105e3c3b9c754b9d0947172889776249a23ecb947d356b878f383b9c82505602e0::server:26bcf92bc1a9839d6b14e112f3eda310

Т.е. команда та же самая, что и на первом сервере, но в конце мы указываем имеющийся токен.

Теперь при получении списка нод (см. проверку работоспособности выше) на любом сервере мы получим список из двух серверов. Сервера запущены и работают.

### Агенты

Осталось организовать машины, на которых будут работать контейнеры. Я создал две штуки, но их может быть любое количество:

 - k3s-agent-01
 - k3s-agent-02

Для каждого из серверов-агентов конфигурация полностью одинаковая и состоит из одной-единственной команды:

    curl -sfL https://get.k3s.io | K3S_URL=https://k3s-balancer.lab.lan:6443 K3S_TOKEN=K105e3c3b9c754b9d0947172889776249a23ecb947d356b878f383b9c82505602e0::server:26bcf92bc1a9839d6b14e112f3eda310 sh -

 - K3S_URL=https://k3s-balancer.lab.lan:6443 - наш **плавающий** адрес балансировщиков.
 - K3S_TOKEN - тот самый токен, который мы использовали для подключения второго сервера.

Применили данную команду на каждом из серверов - получили подключенные агенты. Можем проверить их подключение, использовав команду на одном из серверов:

    k3s kubectl get nodes

В моём случае я вижу два сервера, и два агента.

## Отказоустойчивость

Блок с развертыванием самого k3s получился заметно меньше, чем блоки с подготовкой. Так что дала вся эта схема, которую я расписал?

![](https://raw.githubusercontent.com/Ctp3l0k/homelab-k3s/main/images/flowchart.svg)

У каждого компонента кластера, кроме балансировщиков, необходимо настроить только один адрес для подключения ко всей системе. Нет необходимости каждому агенту настраивать подключение к каждому из серверов, нет необходимости вписывать во внешнем kubectl для управления несколько адресов.

Умрёт любая из виртуальных машин - кластер будет жить дальше. Умрёт отдельный сервис, т.е. MySQL, HAProxy, keepalived, сервер k3s - кластер будет жить. Ну и количество серверов и балансировщиков можно увеличить без всяких проблем. По-моему, получилось весьма HA.

## Что дальше

Дальше можно [установить kubectl на десктоп](https://kubernetes.io/docs/tasks/tools/), не забыть прописать в его конфиге адрес балансировщика серверов, и управлять k3s с его помощью.

Можно установить [Kubernetes Dashboard](https://github.com/kubernetes/dashboard), чтобы посмотреть, что вообще происходит внутри кластера.

А вообще я уже разобрал этот кластер и поднял новый, с etcd вместо внешней БД и плавающим адресом прямо на серверах. Пока писал этот текст, хорошо освежил в памяти работу с HAProxy, keepalived и MySQL, но сама по себе получившаяся схема все-таки такая себе. Такие дела.