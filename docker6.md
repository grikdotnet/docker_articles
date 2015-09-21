Portable database.
========

В этой заметке я опишу свое исследование работы с mysql в docker - стремление сделать так, чтобы команде разработчиков было удобно.
Основные алгоритмы хорошо описаны в документации к образу [MySQL](https://hub.docker.com/r/mysql/mysql-server/), который предоставляет Oracle.
С PostgreSQL можно работать точно так же.

**Вводная**

Обычно я не хочу раздавать полный дамп рабочей базы разработчикам. Причины могут быть разные - большой размер рабочей базы или критические данные. Для разработчиков часто создается урезанная версия небольшого размера со структурой и частью данных.
Эту базу я хочу раздавать в виде образа и запускать работающий сервер БД одной командой.
На рабочем сервере я хочу хранить файлы базы в удобном каталоге и работать с базой из командной строки через сокет.

**UID/GID в разных образах**

В так называемом "официальном" образе mysql (и mariadb) у пользователя mysql логичное для docker значение 999:999 в /etc/passwd. В другом официальном образе mysql-server, который предоставляет Oracle, у пользователя mysql стандартный для red-hat uid 27 - и это удобно для доступа к файлам базы утилитами из host-системы. В то же время, в Ubuntu нет стандартного значения uid/gid для mysql.
Выбирайте себе официальный образ по вкусу. Я для примера взял mysql/mysql-server.

**Пароль для MySQL**

Официальные образы предлагают, что для инициализации хранилища пароли для рута указаны в командной строке как переменная окружения контейнера.
Это почти так же небезопасно, как запускать базу без пароля - команда `docker inspect mysql` выведет все переменные в секции "Env". Кроме того, переменный окружения досутпны во всех прилинкованных контейнерах.
Однако, иначе контейнер просто завершает работу с ошибкой.
Я решу проблему так: инициализирую хранилище из временного контейнера, которому при запуске укажу пароль для root, а для постоянной работы сервис запускается без указания пароля.

**Запускаю локально**

Скачиваю образ MySQL, запускаю во временном контейнере, беру из него конфиг и пишу свой дополнительный файл конфига:
```console
$ docker run --rm mysql/mysql-server cat /etc/my.cnf
$ vi extra.cnf
sql-mode = "STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER"
innodb_file_per_table
innodb_flush_log_at_trx_commit = 2

[client]
user=root
password=my-secret-pw
```

Чтобы файлы базы данных и лог mysql писались в мою папку, подключаю их как data volume, и инициализирую базу данных. Файл лога при монтировании должен сущестовать, иначе Docker создаст вместо него каталог.

```console
$ touch log/mysqld.log
$ chmod a+w log/mysqld.log
$ mkdir mysql_data
$ docker run -d --name mysql \
	-v "$(pwd)/mysql_data:/var/lib/mysql" \
    -e MYSQL_ROOT_PASSWORD=my-secret-pw \
    mysql/mysql-server
01effb0bb481d06dc1642a4f18950224049900971d972beea2a6ab311bd60ceb
$ docker inspect mysql
         "Env": [
                "MYSQL_ROOT_PASSWORD=my-secret-pw",

# Жду несколько секунд, чтобы завершилась инициализация хранилища

$ docker stop mysql
$ docker rm mysql
$ docker run -d --name mysql \
	-v "$(pwd)/mysql_data:/var/lib/mysql" \
    -v "$(pwd)/log/mysqld.log:/var/log/mysqld.log" \
    -v "$(pwd)/extra.cnf:/etc/extra.cnf" \
    mysql/mysql-server --defaults-extra-file=/etc/extra.cnf
```

Создаю базу из файла дампа.
```
$ cat dump.sql  | docker exec -i mysql mysql --defaults-extra-file=/etc/extra.cnf -B
```
Этот дамп должен быть создан командой `mysqldump --databases db_name` чтобы в нем были инструкции `create database` и `use databsase`.
Иначе сначала создайте базу данных из консоли, и добавьте ее название в виде параметра.

Из контейнера коносльный клиент mysql вызывается так:
```
$ docker exec -ti mysql mysql --defaults-extra-file=/etc/extra.cnf
Welcome to the MySQL monitor.  Commands end with ; or \g.
```

Можно подключиться к серверу в контейнере через сокет в монтированном каталоге:
```
$ mysql --defaults-file=./extra.cnf -S ./mysql_data/mysql.sock
Welcome to the MySQL monitor.  Commands end with ; or \g.
...
mysql> \q
Bye

$ mysqldump --defaults-file=./extra.cnf -S ./mysql_data/mysql.sock test > dump.sql
```

Теперь у меня есть работающий сервер, в котором мои данные, конфиг и логи лежат в моих каталогах, отдельно от сторонней, неконтролируемой, но изолированной в контейнере [среды исполнения](https://ru.wikipedia.org/wiki/Среда_выполнения) самой СУБД.

**Образ с дампом базы данных**

Рабочие файлы СУБД занимают намного больше места, чем текстовый дамп, они не переносимы между версиями и деривативами, такими как MariaDB и Percona. Базу данных удобно передавать в виде сжатого текстового дампа. 

Я хочу создать и раздавать компактный образ с дампом базы, без полновесной операционной системы и самой СУБД.
В контейнер СУБД, в каталог /var/lib/mysql я хочу монтировать свой локальный каталог.
При первом запуске СУБД надо автоматически инициализировать базу данных дампом из контейнера приложения. При последующих запусках, когда база уже существует, надо ее не трогать.
По всей видимости, мне нужен дополнительный слой абстракции, который позволит вызвать команду mysql извне контейнера mysql чтобы передать ему на вход дамп из другого контейнера.

Здесь я упираюсь в ограничения возможностей docker. Утилита compose, которая создана для объединения контейнеров, не предоставляет возможности автоматически вызвать команду инициализации контейнера, не встраивая ее в образ. Можно встроить скрипт инициализации в собственный образ, унаследованный от официального, однако, в образе MySQL такой скрипт [уже есть](https://github.com/docker-library/mysql/blob/master/5.6/docker-entrypoint.sh).

Надеюсь, что когда-нибудь разработчики docker compose реализуют возможность указывать команды, которые надо запускать в контейнере при инициализации. Пока что есть два пути автоматизировать установку базы из дампа: 
* через docker-compose создавать специальный контейнер для инициализации базы с клиентским модулем mysql на базе [Alpine Linux](https://hub.docker.com/_/alpine/). К счастью, сам контейнер для запуска команды инициализации передавать не нужно, достаточно одного Dockerfile с командами;
* недокументированная возможность - при инициализации базы скрипт entrypoint.sh [обрабатывает](https://github.com/docker-library/mysql/blob/master/5.6/docker-entrypoint.sh#L79) файлы из /docker-entrypoint-initdb.d/

Последний вариант, своеобразный callback handler, поддерживается всеми официальными образами, включая [MariaDB](https://github.com/docker-library/mariadb/blob/master/docker-entrypoint.sh#L79), [Percona](https://github.com/docker-library/percona/blob/master/docker-entrypoint.sh#L79), а так же образом Oracle MySQL и [PostgreSQL](https://github.com/docker-library/postgres/blob/master/9.4/docker-entrypoint.sh#L76). Этот способ самый удобный, буду его использовать.

**Data volume image**

Дамп базы даннных удобно хранить и передавать в одном образе с приложением. Доработаю образ приложения, который я создал в прошлой статье:

```console
$ cd data_volume/
$ cp ~/dump.sql .
$ cat << EOF > Dockerfile
    FROM busybox
    RUN mkdir /docker-entrypoint-initdb.d/
    COPY dump.sql /docker-entrypoint-initdb.d/dump.sql
    VOLUME mysql_dump_v1.0:/docker-entrypoint-initdb.d/
    VOLUME /scripts
    RUN adduser -D -u 56789 docker_volumes
    COPY source /source/
    USER docker_volumes
    CMD test "$(ls -A "/scripts/" 2>/dev/null)" || cp /source/* /scripts/
EOF
$ docker build -t grikdotnet/application .
$ docker save grikdotnet/application | xz > application.image.tar.xz
```

Если в Dockerfile сначала указать команды записи файлов в каталог, а затем объявить этот каталог как volume, при создании контейнера из образа эти файлы будут скопированы в volume, и смогут быть доступны в других контейнерах. Чтобы файл дампа не стал мусором после удаления контейнера, я задал осмысленное имя для каталога в host-системе. Забыть добавить ключ -v в команду `docker rm`, к сожалению, слишком легко.

Теперь я могу импортировать приложение в виде образа, создать из него контейнер, запустить MySQL, и база данных будет автоматически загружена из дампа.

```
$ docker run --name application -v "$(pwd)/application:/scripts" grikdotnet/application
$ sudo ls /var/lib/docker/volumes/
mysql_dump_v1.0
$ docker run -d --name mysql -v "$(pwd)/mysql_data:/var/lib/mysql" --volumes-from application -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql/mysql-server
$ docker exec -ti mysql mysql
...
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| my_database        |
+--------------------+
```
Так как образ я создаю для удобства разработчиков, а порт из контейнера в мир не транслируется, пароль для root в mysql я отключаю.

Чтобы заменить MySQL версии Oracle на MariaDB, достаточно удалить контейнер mysql, файлы базы данных, и в строке запуска заменить название образа:
```Console
$ docker rm -fv mysql
$ sudo rm -rf mysql_data/*
$ docker run -d --name mysql -v "$(pwd)/mysql_data:/var/lib/mysql" --volumes-from application MYSQL_ALLOW_EMPTY_PASSWORD=yes mariadb
```
Можно и не удалять, а запустить несколько контейнеров СУБД, отработать репликацию, или сравнить планы исполнения запросов в разных сборках.

**Подключение PHP**

Подключить MySQL в связке с php достаточно просто.

Сначала добавляю расширение pdo_mysql [как рекомендуют авторы образа php](https://github.com/docker-library/docs/blob/master/php/README.md):

```dockerfile
	FROM php:5.6-fpm
	RUN docker-php-ext-install pdo_mysql
	CMD ["php-fpm"]
```
Создаю образ, контейнер и запускаю php-fpm:
```console
$ docker build -t grikdotnet/php-pdo_mysql .
$ docker run -d --name=php7 \
	--volumes-from application
	-v "$(pwd)/localetc:/usr/local/etc" \
	-v "$(pwd)/log:/var/log/php" \
	grikdotnet/php-pdo_mysql
```

