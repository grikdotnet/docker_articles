Docker myths and receipts. Monkey patch
========

Начало: https://github.com/grikdotnet/docker_articles/blob/master/docker1.md

Беру образы Nginx и PHP 7.
```
docker@dev:~$ docker pull nginx
...
docker@dev:~$ docker pull php:7
Status: Downloaded newer image for php:7
```

Теперь у меня два есть чужих класса обычного качества, которые надо связать вместе через инъекцию завиимостей. Самый простой способ инъектить зависимости в чужой код, конечно же, [monkeypatching](https://ru.wikipedia.org/wiki/Monkey_patch)!
Сначала создаю контейнеры. Помню о [второй главной сложности программирования](http://martinfowler.com/bliki/TwoHardThings.html) - даю контейнерам вразумительные имена. Короткие ID моих контейнеров мне нужны чтобы удобней к ним обращаться.
```
docker@dev:~$ mkdir monkeypatch
docker@dev:~$ cd monkeypatch/
docker@dev:~/monkeypatch$ docker create php:7 --name=php7
docker@dev:~/monkeypatch$ docker ps -ql
9bec96509f99
docker@dev:~/monkeypatch$ docker create nginx --name=nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
docker@dev:~/monkeypatch$ docker ps -ql
80be81b27e01
```
Для Nginx расположение конфигов стандартизировано. Где лежат конфиги для PHP можно увидеть в его [Dockerfile]((https://github.com/docker-library/php/blob/789a45b03fe31ca1ac7f490bafe300e728b18bb9/7.0/fpm/Dockerfile)).
> ENV PHP_INI_DIR /usr/local/etc/php

```
docker@dev:~/monkeypatch$ docker cp 9bec96509f99:/usr/local/etc/php .
docker@dev:~/monkeypatch$ docker cp 80be81b27e01:/etc/nginx .
docker@dev:~/monkeypatch$ ls
nginx/ php/
```

Правлю конфиги, как обычно.

(tbc)