Docker myths and recipes. MySQL 
========
*незаконченная статья*

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

В этой статье я опишу как можно работать с mysql в docker чтобы команде было удобно.
Основные алгоритмы хорошо описаны в документации к образу [MySQL](https://hub.docker.com/r/mysql/mysql-server/), который предоставляет Oracle.

У меня в работе достаточно крупная база данных. Работать с базой большого объема локально разработчикам неудобно, поэтому я создал урезанную версию небольшого размера со структурой, но с частью данных, достаточной для процесса разработки.
А на рабочем сервере я хочу хранить файлы базы в удобном каталоге и работать с базой из командной строки через сокет.

Поэтому для файлов базы данных я буду использовать так называемый Data Volume сontainer, который у меня ассоциациируется с data transfer object.

****
Скачиваю образ MySQL, запускаю во временном контейнере, беру из него конфиг и редактирую:
```console
$ docker run --rm mysql/mysql-server cat /etc/my.cnf > my.cnf
$ vi my.cnf
	--log-error=/var/log/mysqld.log
	++log-error=/var/log/mysql/mysqld.log
```
Конечно, стоит настроить и все нужные параметры. Я перемещаю лог, чтобы он писался в подмонтированную папку, и можно было посмотреть его без открытия консоли в контейнере. MySQL открывает лог не под рутом, а под mysql, так что надо решить проблему сохранения аттрибутов монтированных каталогов в контейнере, описанную в первой статье.

Я создаю свой контейнер на базе пустого образа, который называется "scratch".
Если я создам его из образа базы данных, как предлагают в [документации](https://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container), при экспорте в архив пойдет вся операционная система, и размер файла составит порядка 500 мегабайт. Раздавать дамп такого размера неудобно. Лучше я создам контейнер, в котором будут только файлы базы данных.
К сожалению, создать пустой контейнер напрямую нельзя, сначала надо отнаследоваться от scratch и создать пустой образ.
```console
$ mkdir data_volume
$ cd data_volume/
$ echo "FROM scratch">Dockerfile
$ echo "CMD true">>Dockerfile
$ docker build -t grikdotnet/data_volume .
$ cd ..; rm -rf data_volume/
```
Совсем пустой образ Docker создать не разрешает, а с LABEL или CMD - можно.

Создаю группу docker и обавляю в нее mysql.
```console

```
Создаю и подключаю data volume для файлов базы данных, а так же общую папку логов, которую я создавал при написании прошлой статьи.
```console
$ docker create -v /var/lib/mysql --name mysql_data grikdotnet/data_volume
$ docker create -d --name mysql --volumes-from mysql_data \
    -v "$(pwd)/log:/var/log/mysql" \
    -v "$(pwd)/my.cnf:/etc/my.cnf" \
    -e MYSQL_ROOT_PASSWORD=my_password mysql/mysql-server
01effb0bb481d06dc1642a4f18950224049900971d972beea2a6ab311bd60ceb
$
$ docker inspect mysql_data |grep Source
            "Source": "/mnt/sda1/var/lib/docker/volumes/1b1870f9e7...81989db3eb72/_data",
```
Создаю в контейнере базу из дампа моего приложения.
```
$ cat dump.sql  | docker exec -i mysql mysql -B
```
Теперь я могу экспортировать контейнер чтобы раздавать его в команде.
```
$ docker export mysql_data >mysql_data.dc.tar
```


Можно подключиться к mysql прямо по сокету в data volume container.
```console
$ sudo mysql -S /var/lib/docker/volumes/1b1870f9e7...81989db3eb72/_data/mysql.sock -pmy_password
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

