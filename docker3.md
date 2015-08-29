Docker myths and receipts. Monkey patch
========

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

### Настройка локально

В этой статье я предполагаю, что служба docker запущена на той же машине, на которой выполняются команды, и у процесса есть доступ на чтение к текущей папке.

Беру образы Nginx и PHP 7.
```
~$ docker pull nginx
...
~$ docker pull php:7-fpm
Status: Downloaded newer image for php:7
```

Теперь у меня есть два чужих класса обычного качества, которые надо связать вместе через инъекцию завиимостей. Самый простой способ инъектить зависимости в чужой код, конечно же, [monkeypatching](https://ru.wikipedia.org/wiki/Monkey_patch)!
Сначала создаю контейнеры. Помню о [второй сложности программирования](http://martinfowler.com/bliki/TwoHardThings.html) - даю контейнерам вразумительные имена, они будут нужны чтобы контейнеры могли взаимодействовать между собой.
```
~$ docker create --name=php7 php:7-fpm
3d1b737edfcc3f1102fa54c91f9120da4b86d8cbba3092b6f80156c0e31b4d8f
~$ docker create --name=nginx nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
```

Для Nginx расположение конфигов стандартизировано. Где лежат конфиги для PHP можно увидеть в его [Dockerfile]((https://github.com/docker-library/php/blob/f5e091ac3815dce80ca496298e0cb94638844b10/7.0/fpm/Dockerfile)):
```
	ENV PHP_INI_DIR /usr/local/etc/php
	   --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
	WORKDIR /var/www/html
	COPY php-fpm.conf /usr/local/etc/
```
```
~$ mkdir monkeypatch
~$ cd monkeypatch/
$ docker cp php7:/usr/local/etc localetc
$ docker cp nginx:/etc/nginx .
$ ls localetc/
pear.conf		php			php-fpm.conf		php-fpm.conf.default	php-fpm.d
$ ls localetc/php
conf.d
```
Мейнтейнеры положили в образ php-fpm.conf, но не удосужились положить дефолтный php.ini. Придется взять его из исходников php.

	$ docker cp $PHP7:/usr/src/php/php.ini-development localetc/php/php.ini

Правлю конфиги, как обычно. В какой папке PHP ищет расширения? Узнать придется у самого php.
Посмотреть конфигурацию php можно, запустив его, например, во временном контейнере.
```
$ docker run --rm php:7-fpm php -i |grep extension_dir
extension_dir => /usr/local/lib/php/extensions/no-debug-non-zts-20141001 => /usr/local/lib/php/extensions/no-debug-non-zts-20141001
$ docker run --rm php:7-fpm ls /usr/local/lib/php/extensions/no-debug-non-zts-20141001
opcache.a
opcache.so
```
В расширениях только опкеш, подключаю его. 
```
$ echo extension_dir = "/usr/local/lib/php/extensions/no-debug-non-zts-20141001" >>  localetc/php/php.ini
$ echo zend_extension = opcache.so >> localetc/php/php.ini
```
Пересоздаю контейнер php и монтирую в него папку с конифгами. Путь к монтируемой папке должен быть от корня - служба не знает из какой папки вызывается клент docker.
```
$ docker rm php7
php7
$ docker run -v `pwd`/localetc:/usr/local/etc --name=php7 php:7-fpm php -i |grep Configuration
Configuration File (php.ini) Path => /usr/local/etc/php
Loaded Configuration File => /usr/local/etc/php/php.ini
```
Теперь можно запустить контейнер php7 с тестовым приложением на php в фоне.
```
$ docker rm php7
$ mkdir scripts
$ echo "<?php echo 'Hello world!'.PHP_EOL;" > scripts/test.php
$ docker run -v `pwd`/localetc:/usr/local/etc \
	-v `pwd`/scripts:/scripts \
	--name=php7 php:7-fpm &
[29-Aug-2015 15:19:25] NOTICE: fpm is running, pid 1
[29-Aug-2015 15:19:25] NOTICE: ready to handle connections
```
Пока что для удобства дебага я оставляю вывод из контейнера php-fpm в свою консоль.
Когда будет надо, можно отредактировать конфиги php и fpm в `localetc/` и перезапускать контейнер.

С Nginx все стандартно: в папке `nginx/` надо отредактировать nginx.conf, fastcgi_params по вкусу и создать конфигурационный файл для своего сайта в `nginx/conf.d/`.
Основное для связи nginx с php - это указать в имени хоста имя контейнера с php.
Директивы root и SCRIPT_FILENAME должны указывать на путь, который php поймет в своем контейнере php7.

    location ~ \.php$ {
        fastcgi_pass   php7:9000;
        fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;

Монтирую конфиги в контейнер nginx и запускаю с маппингом 80-го порта контейнера на локальный 8080.

```
$ docker rm nginx
$ docker run -v `pwd`/nginx:/etc/nginx -p 8080:80 --name=nginx nginx 
$ curl 127.0.0.1:8080/test.php
172.17.0.65 -  29/Aug/2015:15:50:29 +0000 "GET /test.php" 200
Hello world!
```
Rock'n'Roll!

