Херак-херак, и в production!
========

В предыдущей серии: https://github.com/grikdotnet/docker_articles/blob/master/docker3.md я рассказал как легким движением руки запустить сервисы php-fpm и nginx в докере на локальной машине. Это можно сделать удаленно.

Точно так же создаю контейнеры PHP 7 и Nginx, сразу включа маппинг порта.
```
$ docker run -d --name=php7 php:7-fpm
3d1b737edfcc3f1102fa54c91f9120da4b86d8cbba3092b6f80156c0e31b4d8f
$ docker run -d -p 8080:80 --name=nginx nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
```

Когда служба и клиент работают на разных машинах, у службы нет доступа к файловой системе клиентского приложения. Файлы конфигов можно копировать в контейнер. Что надо править в конифгах - смотрите в прошлой статье.

```
$ docker cp "php7:/usr/src/php/php.ini-development" php.ini
$ vi php.ini
$ docker cp php.ini "php7:/usr/local/etc/php/"
$ docker exec php7 pkill -o -USR2 php-fpm

$ echo "<?php echo 'Hello cruel world! ',PHP_VERSION,PHP_EOL;" >test2.php
$ docker exec php7 mkdir /scripts
$ docker cp test2.php "php7:/scripts/"

$ docker cp "nginx:/etc/nginx" nginx
$ vi nginx/nginx.conf
$ vi nginx/conf.d/default.conf
$ docker cp nginx "nginx:/etc/"
$ docker exec nginx service nginx reload
Reloading nginx: nginx.
```

Теперь можно проверить что php видит мой конфиг.
```
~$ docker exec php7 php -i |grep php.ini
Configuration File (php.ini) Path => /usr/local/etc/php
Loaded Configuration File => /usr/local/etc/php/php.ini
```
Запускаю:
```
airgri:articles gri$ docker-machine ip dev
192.168.99.104
airgri:monkeypatch gri$ curl 192.168.99.104:8080/test2.php
Hello cruel world! 7.0.0RC1
```

Настраивать логи в такой конфигурации бессмысленно.