Docker myths and recipes. Херак-херак, и в production!
========


В предыдущей серии: https://github.com/grikdotnet/docker_articles/blob/master/docker3.md
было рассказано как легким движением руки запустить php в докере на локальной машине.

Сейчас опишу способ, как можно сделать почти то же, но удаленно.

Точно так же создаю контейнеры Nginx и PHP 7.
```
~$ docker create --name=php7 php:7-fpm
3d1b737edfcc3f1102fa54c91f9120da4b86d8cbba3092b6f80156c0e31b4d8f
~$ docker create --name=nginx nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
```

Когда служба и клиент работают на разных машинах, у службы нет доступа к файловой системе клиентского приложения. Поэтому файлы конфигов надо копировать в контейнер.

```
~$ docker cp $PHP7:/usr/src/php/php.ini-development php.ini
~$ vi php.ini
~$ docker cp php.ini php7:/usr/local/etc/
```
Выполнить команду в незапущенном контейнере аналогично `docker run image command` нельзя.
Поэтому я запускаю контейнер, приаттачив клиента докера к STDIN/STDOUT контейнера и отправляю в фон.
Затем проверяю что php конфиг считывает:
```
~$ docker start -a php7 &
[1] 10183
[27-Aug-2015 14:56:26] NOTICE: fpm is running, pid 1
[27-Aug-2015 14:56:26] NOTICE: ready to handle connections
```
Теперь можно проверить что php видит мой конфиг.
```
~$ docker exec php7 php -i |grep php.ini
Configuration File (php.ini) Path => /usr/local/etc/php
Loaded Configuration File => /usr/local/etc/php/php.ini
```
При желании можно отредактировать и заменить в контейнере конфиги fpm.

(tbc)