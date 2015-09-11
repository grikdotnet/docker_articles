Docker myths and recipes. Data Volume container.
========

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

В docker есть любопытный инструмент, который называется [Data Volume сontainer](https://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container). 
У меня он ассоциациируется с паттерном data transfer object.

Вот что нужно знать про тома в docker:
* Data Volume - это внешний для контейнера каталог, монтированный в контейнер;
* Если не указать какой каталог монтировать - будет создан новый каталог в /var/lib/docker/volumes/ с названием из 64 цифр в 16-ричной нотации.
 Пример: `docker create -v /shared_folder nginx`;
* При удалении контейнера volume-каталоги не удаляются;
* В каталог /var/lib/docker/volumes/ доступ есть только для рута, так что просмотреть содержимое автоматически созданных томов можно только через `sudo`;
* Данные из тома не экспортируются при выполнении `docker export`, `docker save` и не сохранятся в образ командой `docker commit`;
* Если при монтировании указать несуществующий каталог или если к нему нет доступа - docker не выдаст никакой ошибки;
* При подключении в контейнер внешнего каталога у него остаются uid/gid host-системы.

После удаления контейнеров разобраться к чему относятся тома в /var/lib/docker/volumes/ достаточно сложно. Это может привести к замусориванию и потере места на диске. Удобнее прямо указывать какие каталоги надо монтировать, чтобы можно было просматривать их содержимое и удалять когда они станут не нужны.

Перефразируя фильм "Адвокат дьявола", авторы docker дают нам тома в контейнерах, дарят этот экстраординарный подарок, а потом для своего ролика космических трюков устанавливает противоположные правила игры.
Расшаривай каталог между контейнерами - но будет сложно удалить. Монтируй свою папку и удаляй ее когда надо - но не распространяй в образе. Записывай файлы в образ и распространяй его - но не расшаривай между контейнерами!

Несмотря на проблемы, использовать в контейнерах общие каталоги для хранения файлов приложения и базы данных очень удобно, а проблему можно подпереть костылем в виде пары shell-команд в Dockerfile.

В документации предлагают создавать volume container из образа операционной системы или базы данных.
Если такой контейнер экспортировать, в файл пойдет вся операционная система. Наследование от образа mysql добавит к размеру архива порядка 66 мегабайт при сжатии xz. Чтобы этого избежать, я создаю контейнер от [Busybox](https://hub.docker.com/r/library/busybox/) - он позволяет выполнять простые команды и открывать терминал в контейнере, при этом добавляет в архив менее мегабайта.


**UID/GID и Data Volume**

В контейнерах обычно есть свои пользователи и группы со своими uid/gid, а значения uid/gid монтируемых каталогов сохраняется. Кроме того, процессы в контейнере могут менять uid/gid каталога.
Как будут соотноситься пользователи в контейнере и gid/uid монтируемых каталогов - предсказать невозможно.
С большой вероятностью возникнут проблемы с правами доступа к разделяемому тому.

Чтобы решить эту проблему, можно создать специальную группу в системе и в контейнерах.
К сожалению, у группы docker, которая создатся в host-системе, gid равен 999, и совпадение с ним в сторонних контейнерах весьма вероятно.
Например, в образе mysql-server группа с gid 999 называется ssl.
Создам отдельную группу для томов и добавлю себя в нее для удобства доступа.

```console
$ sudo groupadd -g 56789 docker_volumes
$ sudo usermod -a -G docker_volumes gri
```


**Подготовка образа с приложением**

Я создам образ с пустым volume и файлами приложения. При создании контейнера будет проверяться пустой ли каталог, подключенный как volume.
Если пустой - в него копируются файлы приложения. Если каталог не пустой - не копируются.
При создании контейнера из образа я монтирую в него в качестве тома свой каталог, и в него копируются файлы приложения.
Я могу редактировать файлы приложения и пересоздавать контейнер, подключая к нему свои файлы.
При желании я могу вернуться к начальному состоянию из образа, создав новый контейнер и подключив пустой каталог.

Копирую скрипты приложения в каталоге `source` и готовлю образ со скриптами:
```console
$ mkdir data_volume
$ chgrp docker data_volume
$ cd data_volume/
$ cp ~/app source
$ vi Dockerfile
```

Вот содержимое Dockerfile, из которого я создам образ со своим приложением:
```Dockerfile
FROM busybox
VOLUME /scripts
RUN adduser -D -u 56789 docker_volumes
COPY source /source/
USER docker_volumes
CMD test "$(ls -A "/scripts/" 2>/dev/null)" || cp /source/* /scripts/
```
При запуске контейнера будет выполнятся команда из CMD. Если каталог `application/` пустой, в него будут скопированы скрипты приложения.
Если при запуске контейнера указать команду, директива CMD игнорируется, поэтому я не использую ENTRYPOINT чтобы удобней дебажить, открыв консоль в контейнер.

Создаю образ и контейнер:
```
$ docker build -t grikdotnet/application .
$ cd ..
$ mkdir application
$ sudo chgrp docker_volumes application
$ docker run --name application -v "$(pwd)/application:/scripts" grikdotnet/application
```
Теперь у меня есть контейнер с data volume `/scripts/`, в который монтирован мой каталог `./application/`, и в нем появилось мое приложение.
Если при создании контейнера application каталог scripts/ будет не пустой, так что правки я не потеряю.
Этот data volume я могу подмонтировать в контейнер php.


Этот образ удобно экспортировать
```
~$ docker save grikdotnet/application | xz > application.image.tar.xz
$ ls -lh application.dc.tar.xz
-rw-rw-r-- 1 gri gri 858K Sep  8 14:53 application.dc.tar.xz
```
и импортировать
```
$ docker rm application
$ docker rmi grikdotnet/application
$ rm -rf application/*
$ docker load --input application.image.tar.xz
$ docker run --name application -v "$(pwd)/application:/scripts" grikdotnet/application
$ ls -Al application
total 4
-rw-r--r-- 1 56789 docker_volumes 28 Sep  9 09:22 test.php
```
Да, пользователя с gid 56789 в host-системе нет.

Подключение к [контейнеру php](https://github.com/grikdotnet/docker_articles/blob/master/docker3.md) выполняется параметром `--volumes-from`:
```
$ docker run -d --name=php7 \
	-v "$(pwd)/localetc:/usr/local/etc" \
	--volumes-from application/
	-v "$(pwd)/log:/var/log/php" \
	php:7-fpm >>log/docker.php.log 2>&1
```
Конечно, PHP тоже может исполняться под пользователем docker_volumes.

Для этого надо расширить образ PHP, используя Dockerfile, подобный такому:
```Dockerfile
FROM php:5.6-fpm
RUN adduser -D -u 56789 docker_volumes
RUN apt-get update && apt-get install -y \
        libmcrypt-dev \
    && docker-php-ext-install mcrypt pdo_mysql
CMD ["php-fpm"]
```
и в подключаемом php-fpm.conf указать
	user = docker_volumes
	group = docker_volumes

Пошаговые инструкции по запуску php можно найти в третьей части.

Продолжение следует.