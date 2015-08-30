Docker myths and recipes. Monkey patch
========

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

### Настройка локально

В этой статье я предполагаю, что служба docker запущена на той же машине, на которой выполняются команды, и у процесса есть доступ на чтение к текущей папке. Еще я подразумеваю, что вы умеете настраивать связку PHP-FPM и Nginx.

Беру образы Nginx и PHP 7.
```
~$ docker pull nginx
...
~$ docker pull php:7-fpm
Status: Downloaded newer image for php:7-fpm
```

Теперь у меня есть два чужих класса, которые надо связать вместе через внедрение зависимостей. Самый простой способ добавлять зависимости в чужой код, конечно же, [monkeypatching](https://ru.wikipedia.org/wiki/Monkey_patch)!
Сначала создаю контейнеры. Помню о [второй сложности программирования](http://martinfowler.com/bliki/TwoHardThings.html) - даю контейнерам вразумительные имена, они будут нужны, чтобы контейнеры могли взаимодействовать между собой.
```
~$ docker create --name=php7 php:7-fpm
3d1b737edfcc3f1102fa54c91f9120da4b86d8cbba3092b6f80156c0e31b4d8f
~$ docker create --name=nginx nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
```

### PHP

Начну с PHP - его настроить сложнее. Где лежат конфиги для PHP - можно увидеть в его [Dockerfile](https://github.com/docker-library/php/blob/f5e091ac3815dce80ca496298e0cb94638844b10/7.0/fpm/Dockerfile):
```
	ENV PHP_INI_DIR /usr/local/etc/php
	   --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
	WORKDIR /var/www/html
	COPY php-fpm.conf /usr/local/etc/
```
Копирую себе из контенера содержимое каталога с файлами конфигурации php
```
~$ mkdir monkeypatch
~$ cd monkeypatch/
$ docker cp php7:/usr/local/etc localetc
$ ls localetc/
pear.conf		php			php-fpm.conf		php-fpm.conf.default	php-fpm.d
$ ls localetc/php
conf.d
```
Мейнтейнеры положили в образ php-fpm.conf, но не положили дефолтный php.ini. Придется взять его из исходников php.

	$ docker cp "$PHP7:/usr/src/php/php.ini-development" localetc/php/php.ini

Правлю конфиги, как обычно. В какой папке PHP ищет расширения?
Узнать можно, запустив php, например, во временном контейнере.
```
$ docker run --rm php:7-fpm php -i |grep extension_dir
extension_dir => /usr/local/lib/php/extensions/no-debug-non-zts-20141001 => /usr/local/lib/php/extensions/no-debug-non-zts-20141001
$ docker run --rm php:7-fpm ls /usr/local/lib/php/extensions/no-debug-non-zts-20141001
opcache.a
opcache.so
```
В расширениях только opcache, подключаю его. 
```
$ echo extension_dir = "/usr/local/lib/php/extensions/no-debug-non-zts-20141001" >>  localetc/php/php.ini
$ echo zend_extension = opcache.so >> localetc/php/php.ini
```
Пересоздаю контейнер php и монтирую в него папку с конифгами. Путь к монтируемой папке должен быть от корня - служба не знает, из какой папки вызывается клиент docker.
```
$ docker rm php7
php7
$ docker run -v "$(pwd)/localetc:/usr/local/etc" --name=php7 php:7-fpm php -i |grep Configuration
Configuration File (php.ini) Path => /usr/local/etc/php
Loaded Configuration File => /usr/local/etc/php/php.ini
```
Теперь можно пересоздать контейнер php7 с тестовым приложением на php. Создатели образа не позаботились о том, чтобы php-fpm работал как демон, так что надо самим запускать его фоном, не освобождая стандартные каналы ввода-вывода.
```
$ docker rm php7
$ mkdir scripts
$ echo "<?php echo 'Hello world! ',PHP_VERSION,PHP_EOL;" > scripts/test.php
$ docker run -v "$(pwd)/localetc:/usr/local/etc" \
	-v "$(pwd)/scripts:/scripts" \
	--name=php7 php:7-fpm &
[29-Aug-2015 15:19:25] NOTICE: fpm is running, pid 1
[29-Aug-2015 15:19:25] NOTICE: ready to handle connections
```
Пока что для удобства отладки я оставляю вывод из контейнера php-fpm в свою консоль.

### NGINX

С Nginx всё просто и стандартно. Копирую на диск папку конфигов:
```
$ docker cp nginx:/etc/nginx .	
```
В папке `nginx/` надо отредактировать nginx.conf, fastcgi_params по вкусу, и создать конфигурационный файл для своего сайта в `nginx/conf.d/`.
Основное для связи nginx с php - это указать в имени хоста имя контейнера с php, а директивы root и SCRIPT_FILENAME должны указывать на путь, который php поймёт в своём контейнере php7.

    location ~ \.php$ {
        fastcgi_pass   php7:9000;
        fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;

Монтирую конфиги в контейнер nginx и запускаю с маппингом 80-го порта контейнера на локальный 8080. 

```
$ docker rm nginx
$ docker run -v "$(pwd)/nginx:/etc/nginx" -p 8080:80 --name=nginx nginx &
$ curl 127.0.0.1:8080/test.php
172.17.0.65 -  29/Aug/2015:15:50:29 +0000 "GET /test.php" 200
Hello world! 7.0.0RC1
```
Rock'n'Roll!

### Логи

Мейнтейнеры образа php решили писать все логи fpm в /proc/self/fd/2, он же STDERR - как error_log, так и access.log. Однако, лог запросов у меня будет писать nginx, а в работе php меня интересуют только ошибки, поэтому предлагаю отредактировать localetc/php-fpm.conf и написать что-то привычное:

	error_log = /var/log/php/php-fpm.error.log
	;access.log = /proc/self/fd/2 

В Nginx обошлись без самодеятельности, так что включаю access log в конфиге сайта nginx/conf.d/site.ru.conf

    access_log  /var/log/nginx/host.access.log  main;

Теперь можно создать папку для логов c правом записи для демона docker и подмонтировать ее в контейнеры. 
В эту же папку можно писать и вывод контейнеров, при этом контейнеры можно детачить:
```
$ mkdir log
$ sudo chgrp docker log
$ sudo chmod g+rwx log/
$ docker stop nginx php7
$ docker rm nginx php7
$ docker run -d --name=php7 \
	-v "$(pwd)/localetc:/usr/local/etc" \
	-v "$(pwd)/scripts:/scripts" \
	-v "$(pwd)/log:/var/log/php" \
	php:7-fpm >>log/docker.php.log 2>&1
$ docker run -d --name=nginx \
	-v "$(pwd)/nginx:/etc/nginx" \
	-v "$(pwd)/log:/var/log/nginx" \
	-p 8080:80 \
	nginx >>log/docker.nginx.log 2>&1
$ curl 127.0.0.1:8080/test.php
Hello world! 7.0.0RC1
```

Когда надо, можно отредактировать конфиги php, fpm и nginx, и перезапускать php.
```
$ docker exec php7 pkill -o -USR2 php-fpm
$ docker exec nginx service nginx restart
Restarting nginx: nginx
```
Надеюсь, когда-нибудь для официального образа php напишут init-скрипт. При желании, можно добавить его самостоятельно из дистрибутива по выбору.

Продолжение: https://github.com/grikdotnet/docker_articles/blob/master/docker4.md

