Docker myths and recipes. Portable database.
========

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

В этой статье я опишу свой способ работать с mysql в docker - так, чтобы команде разработчиков было удобно.
Основные алгоритмы хорошо описаны в документации к образу [MySQL](https://hub.docker.com/r/mysql/mysql-server/), который предоставляет Oracle.
С PostgreSQL можно работать точно так же.

**Вводная**

Обычно я не хочу раздавать полный дамп рабочей базы разработчикам.
По разным причинам - будь то большой размер рабочей базы, или в ней хранятся критические данные, для разработчиков часто создается урезанная версия небольшого размера со структурой и частью данных.
Эту базу я хочу раздавать в виде образа и запускать работающий сервер БД одной командой.
На рабочем сервере я хочу хранить файлы базы в удобном каталоге и работать с базой из командной строки через сокет.

**Запускаю локально**

Скачиваю образ MySQL, запускаю во временном контейнере, беру из него конфиг и редактирую:
```console
$ docker run --rm mysql/mysql-server cat /etc/my.cnf > my.cnf
$ vi my.cnf
```
Правлю в my.cnf нужные мне параметры.

Чтобы файлы базы данных и лог mysql писались в мою папку, подключаю их как data volume.

```console
$ touch log/mysqld.log
$ chmod a+w log/mysqld.log
$ mkdir mysql_data
$ docker run -d --name mysql \
	-v "$(pwd)/mysql_data:/var/lib/mysql" \
    -v "$(pwd)/log/mysqld.log:/var/log/mysqld.log" \
    -v "$(pwd)/my.cnf:/etc/my.cnf" \
    -e MYSQL_ROOT_PASSWORD=my_password mysql/mysql-server
01effb0bb481d06dc1642a4f18950224049900971d972beea2a6ab311bd60ceb
```
Создаю базу из файла дампа.
```
$ cat dump.sql  | docker exec -i mysql mysql -B
```
Если клиент mysql установлен локально, можно подключиться к серверу в контейнере через сокет:
```
$ mysql -S mysql_data/mysql.sock -p
```

Теперь у меня есть работающий сервер, в котором данные, конфиг и логи пишутся в мои локальные каталоги.

**Готовлю образ c файлами базы данных для раздачи**

Создаю образ для Data Volume.

Docker-файл:
```Dockerfile
FROM busybox
VOLUME /var/lib/mysql
RUN adduser -D -u 56789 docker_volumes
COPY source/* /var/lib/mysql/
USER docker_volumes
CMD test "$(ls -A "/var/lib/mysql/" 2>/dev/null)" || cp /source/* /var/lib/mysql/
```

```
$ docker run -d --name mysql --volumes-from mysql_data \
    -v "$(pwd)/log/mysqld.log:/var/log/mysqld.log" \
    -v "$(pwd)/my.cnf:/etc/my.cnf" \
    -e MYSQL_ROOT_PASSWORD=my_password mysql/mysql-server
01effb0bb481d06dc1642a4f18950224049900971d972beea2a6ab311bd60ceb

```

```
$ docker export mysql_data |xz mysql_data.dc.tar
```
Раздаю полученный mysql_data.dc.tar.xz
Импортировать контейнер так же просто:
```
$ docker export mysql_data |xz mysql_data.dc.tar
```


Можно подключиться к mysql прямо по сокету в data volume container.
```console
$ mysql -S /var/lib/docker/volumes/1b1870f9e7...81989db3eb72/_data/mysql.sock -pmy_password
Welcome to the MySQL monitor.  Commands end with ; or \g.
Server version: 5.6.26 MySQL Community Server (GPL)
...
mysql> \q
Bye
```

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
	-v "$(pwd)/localetc:/usr/local/etc" \
	-v "$(pwd)/scripts:/scripts" \
	-v "$(pwd)/log:/var/log/php" \
	grikdotnet/php-pdo_mysql
```

