Je suis Docker.
========

They are talking about Docker all the time. I know what you answer: "It's just for fun", "One can prepare an image for a cloud and launch it the same way", "You can just set up an LXC, chroot or AppArmor". You know you don't need it. One more trendy toy. Too lazy to study, at last. But curious! Ok, this article is for you.

If you never heard about containers in Linux, here is a list of pages you need to read to understand what's this all about:
- https://en.wikipedia.org/wiki/LXC
- https://en.wikipedia.org/wiki/UnionFS
- http://habrahabr.ru/post/253877/
- https://www.docker.com/whatisdocker

Set up Docker, it's not large. For Windows and Mac you can set up a Toolbox: https://www.docker.com/toolbox.
It is better to create and set up the virtual machine using command line rather then a GUI tool.
Notes for MAC OS are [here](./docker_mac.md).
Take a few lessons from a manual. This articles contains what the documentation doesn't.

**Docker is not a virtualization.**

Here's my Linux:
```
Welcome to Ubuntu 15.04 (GNU/Linux 3.19.0-15-generic x86_64)

Last login: Tue Aug 18 00:43:50 2015 from 192.168.48.1
gri@ubuntu:~$ uname -a
Linux ubuntu 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/                                       Linux
gri@ubuntu:~$ free -h
             total       used       free     shared    buffers     cached
Mem:          976M       866M       109M        11M       110M       514M
-/+ buffers/cache:       241M       735M
Swap:         1.0G       1.0M       1.0G
```
Starting CentOS
```
gri@ubuntu:~$ docker run -ti centos
[root@301fc721eeb9 /]# uname -a
Linux 301fc721eeb9 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
[root@301fc721eeb9 /]# cat /etc/redhat-release
CentOS Linux release 7.1.1503 (Core)
[root@301fc721eeb9 /]# free -h
              total        used        free      shared  buff/cache   available
Mem:           976M         85M        100M         12M        790M        677M
Swap:          1.0G        1.0M        1.0G

```

Docker - is not a chroot, their functionality partially match. It's not a security system like AppArmor. Docker uses same containers as LXC, but it's interesting not because of containers. Docker is nothing I thought about it before I read documentation.

Same kernel, memory, filesystem, but distributives, libraries, processes and users are different. Though, they can be same as well, if one whishes.

**Docker is an object oriented design tool for an infrastructure.**

A common moot point is whether Nginx configuration files are a part of a web application. Architects fly in UML dreams, hitting dependencies disputes with system administrators right before launch. Conflicts of this kind throw the project down to missed deadlines and significant losses.
Here come devops guys, replacing conventional procedural bash routines with an OOP design applied to the whole infrastructure.
Docker provides encapsulation, inheritance and polymorphism to system components, such as database or data. You can decompose a whole information system: split up an application, a web server, a database, system libraries and data to the independent components. You may inject dependencies from configs and make it work in a group, identically on different servers.

Такой подход можно использовать, чтобы снизить потери рабочего времени дорогих front-end разработчиков на настройку базы данных и Nginx.
Чтобы уйти от vendor lock-in. Не обломаться когда openssl на сервере не поддерживает cipher, используемый в API госучреждения.
Чтобы приложение работало независимо от версии PHP или Python на сервере заказчика.
Создавать open source не только в виде кода, но и настройкой пакетов из нескольких приложений, написанных на разных языках, работающих на разных слоях OSI.

**Начало**

Итак, я открыл https://docs.docker.com/mac/started/, поставил Docker, выполнил несколько упражнений, и почувствовал, что меня держат за дурачка-двоечника, которого боятся перегрузить информацией.
Первый вопрос: куда это чертов докер поставился, где лежат его файлы, в каком формате, как оно все устроено?
Ответы здесь: http://blog.thoward37.me/articles/where-are-docker-images-stored/

Если вкратце, для работы с файловой системой Docker может использовать один из нескольких драйверов, обычно это [AUFS](https://en.wikipedia.org/wiki/Aufs), и все файлы контейнеров лежат в /var/lib/docker/aufs/diff/.
В /var/lib/docker/containers/ служебная информация, а не сами файлы контейнеров.

Образы - это как классы в коде. Контейнеры - как объекты, созданные из классов. Основное отличие - контейнер можно закомитить и сделать из него образ. 
Образы состоят из так называемых слоев, слои - это папки, которые лежат в /var/lib/docker/aufs/diff/. Обычно образы приложений наследуют какие-то готовые официальные системные образы. Когда Docker скачивает образ, ему нужны только те слои, которых у него нет.

Например, скачаю я официальный образ nginx: https://hub.docker.com/r/library/nginx/tags/
```
docker@dev:~$ docker pull nginx
latest: Pulling from nginx
aface2a79f55: Pull complete
72b67c8ad0ca: Downloading [=============>                                     ] 883.6 kB/3.386 MB
9108e25be489: Download complete
902b87aaaec9: Already exists
9a61b6b1315e: Already exists
```
Пишут, что образ nginx 1.9.4 размером 52 мб, а по факту, у меня скачается всего 3 мб. Это потому, что nginx собран на образе `debian:jessie`, который у меня "Already exists".
Есть много образов на базе Ubuntu. Конечно, стоит собирать свою систему из образов с одним предком.

**Docker не исполняет контейнеры, а управляет ими**

Контейнеры исполняются механизмом ядра под названием Cgroups(https://en.wikipedia.org/wiki/Cgroups).
Служба `docker` запускает контейнер по команде, полученной, от клиентского приложения (например, `docker`) и останавливает его когда в контейнере освобождается поток стандартного ввода-вывода. Поэтому в конфигурации Nginx для Docker [пишут](https://hub.docker.com/_/nginx/): 

> Be sure to include daemon off; in your custom configuration to ensure that Nginx stays in the foreground so that Docker can track the process properly (otherwise your container will stop immediately after starting)!

Когда работа контейнера заканчивается, он не удаляется, если это не указать специально. Каждый запуск контейнера командой `$ docker run image_name` без параметров `--name` или `--rm` создает новый контейнер с уникальным идентификатором, который остается в системе до удаления. Так что Docker - система, склонная к замусориванию.
Имена контейнеров в системе уникальны. Рекомендую присваивать имя каждому создаваемому постоянному контейнеру,  а контейнеры, в которых не нужно сохранять данные, рекомендую создавать с параметром --rm.
Контейнеры создаются командами `docker run` и `docker create`. Посмотреть список всех существующих в системе контейнеров можно командой `docker ps -a`.

**Docker - это клиент-серверная системная служба**

Соответственно, Docker может и зависнуть. Если вы дали команду скачать образ, единственный способ прервать процесс скачивания - перезапустить службу. Авторы уже давно обсуждают что с этим делать, но воз и ныне там.

Например, в версии 1.8.1 есть воспроизводимая ошибка:
```
docker@dev:~$ docker pull debian
Using default tag: latest
latest: Pulling from library/debian
2c49f83e0b13: Downloading [===================>                               ] 19.89 MB/51.37 MB
```
Нажимаю Ctrl-C, затем сразу запускаю скачивание повторно.
```
docker@dev:~$ docker pull debian
Using default tag: latest
```
Картина Репина "Приплыли". То есть, зависли. Надо перезапустить демон.
```
docker@dev:~$ sudo /etc/init.d/docker restart
Need TLS certs for dev,127.0.0.1,10.0.2.15,192.168.99.104
-------------------
docker@dev:~$ sudo /etc/init.d/docker status
Docker daemon is running
docker@dev:~$ docker pull debian
Using default tag: latest
latest: Pulling from library/debian
...
Status: Downloaded newer image for debian:latest
```
Бывает, что демон docker не хочет умирать самостоятельно и не освобождает порт, а init-скрипт пограничные случаи еще не отрабатывает.
Так что не забывайте проверять `sudo /etc/init.d/docker status`, `sudo netstat -ntpl`, доставайте бубен и танцуйте.

Еще надо помнить, что порядок операторов для команды docker имеет значение. Если написать `docker create nginx --name=nginx`, --name=nginx будет считаться командой, которую надо выполнить в контейнере, а не именем контейнера.

Теперь вам будет проще разбираться с официальной документацией.

