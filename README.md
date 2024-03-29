# laravel

Вопросы:
# 1. Как из данной архитектуры можно реализовать отказоустойчивый ресурс?
   
   Для обеспечения отказоустойчивой архитектуры необходимо: добавить перед среверами приложения, сервера балансировки. На каждом из серверов необходимо добавить службу keeplived, которая позволит добавить виртуальный ip адрес. К этому адресу необходимо присвоить DNS имя. Так же небходимо добавить health check`и, что бы в случае если 1 из серверов в кластере балансировщиков был недоступен, другой мог взять виртуальный ip и трафик шел уже через него. По мимо этого необходимо прибегнуть к горизонтальной балансировки в отношение серверов с приложением laravel, предоврительно их добавив в группу серверов на которые прокси балансировщиков направляет трафик, необходимо так же добавить health checks для этих серверов. Что бы в случае недоступности трафик перестал идти на не рабочий приклад. Так же можно реализовать возможность autoscale laraver приложений используя инструмент terraform если hosting профвайдер подтдерживает данный инструмент. Так же необходимо создать кластер баз данных. Для того что бы в случае если одна из баз данных выйдет из строя приложение могло подключится к остальным. Полезно будет обеспечить мониторинг инфраструктуры и алертинг в случаее отклонения от нормы, это поможет спрогназировать будущие недосутпуности (Например: колличество пользователей приложения постепенно расло до тех пор пока не закончились ресурсы, в следствии чего началась замедленная работа приложения, а в некоторых случаях и недоступность)
   
# 2. Какие способы создания реплики от работающей Mysql базы вы знаете? В чём между ними различия? Какой из способов стоит использовать и почему? Не нужно перечислять типы репликации и писать о различиях. Нужно описать процесс создания реплики от работающей БД, кратко.
   
   MySQL поддерживает различные методы репликации:
master - slave, где master работает в режиме read-writte, slave - работают в режиме read-only, который основан на репликации событий (events) из бинарного лога источника (binary log replication).

Для реализации данного метода с нуля необходимо:
   ### на master:
   ```
root@repl-master:~# sudo apt-get update
root@repl-master:~# sudo apt-get install mysql-server mysql-client -y
root@repl-master:~# sudo mysql_secure_installation

```
добавим в /etc/mysql/mysql.conf.d/mysqld.cnf:
```
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
```

Создаем пользователя для подключения slave к master:
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepassword';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
mysql> FLUSH PRIVILEGES;
mysql> FLUSH TABLES WITH READ LOCK;
```
Создаем дамп базы данных и отправляем его на slave server:
```
mysqldump -u root --all-databases --master-data > masterdump.sql
scp masterdump.sql 10.11.12.102:
```
### на slave
```
root@repl-slave:~# sudo apt-get install -y mysql-server
```
добавим в /etc/mysql/mysql.conf.d/mysqld.cnf:
```
server-id = 2
```
Задаем параметры того как подключается slave к master:
```
mysql> CHANGE MASTER TO
mysql> MASTER_HOST='10.11.12.101',
mysql> MASTER_USER='repl',
mysql> MASTER_PASSWORD='slavepassword';
```
Вгружаем дамп базыЖ
```
root@repl-slave:~# mysql -uroot < masterdump.sql
```
Стартуем и проверяем статус репликации:
```
mysql> start slave;
mysql> show slave status\G;
```



master - master, где master работает в режиме read-writte

Добавим в /etc/mysql/mysql.conf.d/mysqld.cnf server1:
```
[mysqld]
server-id=1
log-bin="mysql-bin"
auto-increment-offset = 1
```
Добавим в /etc/mysql/mysql.conf.d/mysqld.cnf server1:
```
[mysqld]
server-id=2
log-bin="mysql-bin"
auto-increment-offset = 2
```
Создаем пользователя на 2 серверах:
MySQL8 and Above
```
mysql> CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
```
Below MySQL8
```
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%' IDENTIFIED BY 'password';
```
После того как команды выше выполнены на двух серверах выполняем на обоих серверах:
```
root@repl-server1:~# mysql -u replicator -p -h x.x.x.x -P 3306
```
Узнаем на какой позиции находится наш binlog на server1:
```
mysql> SHOW MASTER STATUS
```
на server2 заменяем MASTER_LOG_POS на тот который мы получили ранее:
```
mysql> STOP SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = 'IP ADDRESS OF SERVER A', MASTER_USER = 'replicator', MASTER_PASSWORD = 'password', MASTER_LOG_FILE = 'mysql-bin.000003', MASTER_LOG_POS = 157, GET_MASTER_PUBLIC_KEY=1;
mysql> START SLAVE;
```
Узнаем на какой позиции находится наш binlog на server2:
```
mysql> SHOW MASTER STATUS
```
на server1 заменяем MASTER_LOG_POS на тот который мы получили ранее:
```
mysql> STOP SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = 'IP ADDRESS OF SERVER A', MASTER_USER = 'replicator', MASTER_PASSWORD = 'password', MASTER_LOG_FILE = 'mysql-bin.000003', MASTER_LOG_POS = 157, GET_MASTER_PUBLIC_KEY=1;
mysql> START SLAVE;
```
### Преимущества репликации master-master между master-slave:

Высокая отказоустойчивость: Если один из мастеров выходит из строя, другой мастер может продолжать обслуживать запросы без простоев.
Балансировка нагрузки: Оба мастера могут принимать записи, что позволяет распределять нагрузку между ними.
Географическая отказоустойчивость: Распределение мастеров по разным географическим зонам позволяет обеспечить отказоустойчивость в случае недоступности одной из зон.
Лучшая доступность данных: При использовании репликации master-master данные на обоих мастерах всегда актуальны и доступны для чтения и записи.

### Недостатки репликации master-master между master-slave:

Сложность конфигурации и управления: Настройка и управление репликацией master-master более сложны, чем у master-slave, требуется более тщательное планирование и контроль.
Возможность конфликтов данных: При одновременной записи на обоих мастерах могут возникнуть конфликты данных, которые требуют разрешения.
Увеличение нагрузки на сеть: Поскольку данные должны быть синхронизированы между двумя мастерами, это может привести к увеличению нагрузки на сеть.
Потенциальные проблемы с целостностью данных: Из-за возможности конфликтов данных и сложностей в резолюции, может возникнуть риск потери целостности данных.
Сложности в масштабировании: При увеличении количества мастеров в репликации master-master может возникнуть сложность в управлении и масштабировании системы.

   
# 3. Перечислите преимущества и недостатки nginx ingress controller. Какие ещё ingress controller вы знаете? В чём между ними различия?

# Преимущества nginx ingress controller:

Высокая производительность и эффективность обработки HTTP-трафика.
Легкая настройка и гибкость в управлении маршрутизацией трафика.
Поддержка множества функций, таких как балансировка нагрузки, SSL-терминация, кеширование и многое другое.
Широкая поддержка сообществом и активное развитие проекта.
   
# Недостатки nginx ingress controller:

Не является стандартным инструментом для управления Kubernetes-кластером, требует дополнительной установки и настройки.
Ограниченные возможности в сравнении с другими Ingress контроллерами, такими как Istio.
Может потребоваться дополнительная настройка для обеспечения безопасности и защиты от атак.
Не всегда поддерживает все последние функции Kubernetes из коробки.


   ### Сравнение между nginx ingress controller и Istio Ingress Gateway:
   Функциональность:
   - nginx ingress controller: Предоставляет базовые функции маршрутизации HTTP-трафика, балансировки нагрузки, SSL-терминации и кеширования.
   - Istio Ingress Gateway: Предоставляет расширенные функции управления трафиком, безопасности, мониторинга и контроля доступа.

   Управление трафиком:
   - nginx ingress controller: Простое управление маршрутами и балансировкой нагрузки.
   - Istio Ingress Gateway: Позволяет настраивать сложные правила маршрутизации, контролировать трафик с помощью виртуальных сервисов и весов балансировки.

   Безопасность:
   - nginx ingress controller: Требует дополнительной настройки для обеспечения безопасности.
   - Istio Ingress Gateway: Имеет встроенные функции безопасности, такие как управление доступом, шифрование трафика и контроль политик безопасности.

   Сложность настройки:
   - nginx ingress controller: Прост в настройке и использовании.
   - Istio Ingress Gateway: Требует более сложной конфигурации из-за большего количества функций.

   Сообщество и поддержка:
   - nginx ingress controller: Широко используется и имеет большое сообщество пользователей.
   - Istio Ingress Gateway: Активно развивается сообществом Istio и имеет поддержку от команды разработчиков.
