Docker myths and recipes. MySQL 
========

*незаконченная статья*

Подключить MySQL в связке с php достаточно просто, как это сделать - хорошо описано в документации к образу [MySQL](https://hub.docker.com/r/mysql/mysql-server/), который предоставляет Oracle.

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

Скачиваю образ MySQL, запускаю во временном контейнере и беру из него конфиг:
```console
$ docker run --rm mysql/mysql-server cat /etc/my.cnf > my.cnf
$ vi my.cnf
	--log-error=/var/log/mysqld.log
	++log-error=/var/log/mysql/mysqld.log
	--socket=/var/lib/mysql/mysql.sock
	++socket=/var/run/mysql.sock
```

Я хочу выделить данные из приложения с контейнером Mysql. Хочу подготовить для разработчиков контейнер базы данных со структурой и частью данных небольшого размера, который они могут скачать и запустить у себя. На рабочем сервере лучше хранить файлы базы в удобном для меня каталоге, а не в хранилище docker. Для этого я создаю так называемый Data Volume сontainer. Затем я хочу этот контейнер наполнить базой данных, экспортировать и передать другим разработчикам в команде. Чтобы не экоспортировать всю операционную систему, я создаю свой data transfer object на базе пустого образа, который называется "scratch".
К сожалению, создать пустой контейнер напрямую нельзя, сначала надо отнаследоваться от scratch и создать пустой образ.
```console
$ mkdir data_volume
$ cd data_volume/
$ echo "FROM scratch">Dockerfile
$ echo "CMD true">>Dockerfile
$ docker build -t grikdotnet/data_volume .
$ cd ..; rm -rf data_volume/
```
Совсем пустой образ Docker создать не разрешает, а с LABEL - можно.

Создаю и подключаю data volume для файлов базы данных, а так же общую папку логов.
```console
$ docker create -v /var/lib/mysql --name mysql_data grikdotnet/data_volumes
$ docker run -d --name mysql --volumes-from mysql_data -v "$(pwd)/log:/var/log/mysql" -v "$(pwd)/my.cnf:/etc/my.cnf" -e MYSQL_ROOT_PASSWORD=my_password mysql/mysql-server
01effb0bb481d06dc1642a4f18950224049900971d972beea2a6ab311bd60ceb
$ docker inspect mysql_data |grep Source
            "Source": "/mnt/sda1/var/lib/docker/volumes/1b1870f9e7...81989db3eb72/_data",
$ sudo ls /mnt/sda1/var/lib/docker/volumes/1b1870f9e7...81989db3eb72/_data
auto.cnf            ib_logfile1         mysql               test
ib_logfile0         ibdata1             performance_schema

$ cat dump.sql  | docker exec -i mysql mysql -B
$ echo "show databases" | docker exec -i mysql mysql
	Database
	information_schema
	my_database
	mysql
	performance_schema
	test
```

