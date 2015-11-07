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
Here come devops guys, replacing conventional procedural bash calls with an OOP design applied to the whole infrastructure.
Docker provides encapsulation, inheritance and polymorphism to system components, such as database or data. You can decompose a whole information system. An application, web server, database, system libraries and data can be independent components of a whole. You may inject dependencies from configs and make it work in a group, identically on different servers.

One can use this approach to reduce costs on expensive frontend developers setting up a web server and a database. To avoid vendor lock-in. To get the application working when an openssl on a production server does not support a cipher used by a government API. To make your application work independently from a PHP or Python version on a customer server.
Create open source not just as a code, but as a composition of pre-configured packages of different application, written in different languages, working in different OSI layers.

**Beginning**

So, I opened https://docs.docker.com/mac/started/, installed Docker, executed several exercises, and felt myself as a looser, who authors don't want to overload with information.
First questions: where this damn docker got installed, what is the location of files, what format used, how is it arranged?
The answers are here: http://blog.thoward37.me/articles/where-are-docker-images-stored/

In short, to work with file system Docker can use one of its drivers. Usually it's [AUFS](https://en.wikipedia.org/wiki/Aufs), and files for all containers are in /var/lib/docker/aufs/diff/.
/var/lib/docker/containers/ contains service information, not the containers themselves.

Images are like classes. Containers are like objects, created from classes. The major difference is that a container can be committed and form an image.
Images consist of the so called layers. Layers are in fact folders in /var/lib/docker/aufs/diff/. Most of images with applications inherit from some ready-made official system images. When Docker downloads an image, it needs just the missing layers.

E.g. I download an image https://hub.docker.com/r/library/nginx/tags/
```
docker@dev:~$ docker pull nginx
latest: Pulling from nginx
aface2a79f55: Pull complete
72b67c8ad0ca: Downloading [=============>                                     ] 883.6 kB/3.386 MB
9108e25be489: Download complete
902b87aaaec9: Already exists
9a61b6b1315e: Already exists
```
They write nginx 1.9.4 image is 52 Mb, but in fact I am downloading just 3Mb. This is because nginx is built on `debian:jessie` that "Already exists" in my storage.
There is a lot of images based on Ubuntu as well. Of course, it makes sense to build all images of an application stack with the same ancestor image.

**Docker does not execute containers, but manages them**

Containers are executed by the kernel feature called Cgroups(https://en.wikipedia.org/wiki/Cgroups).
The `docker` service starts container by a command received from a client application, like `docker`, and waits until the container releases the standard i/o stream. That's why docker config for Nginx [you can see](https://hub.docker.com/_/nginx/): 

> Be sure to include daemon off; in your custom configuration to ensure that Nginx stays in the foreground so that Docker can track the process properly (otherwise your container will stop immediately after starting)!

When the container finishes execution, it is not deleted, if it is not defined explicitely. Each container run with a command `$ docker run image_name` without paramenters `--name` or `--rm` creates a new container with a unique ID. Such container stays in a system until deleted. Docker is a system prone to littering.
Container names are unique within a system. I recommend naming each permanent container. The ones that need not store any data, I recommend rnning with --rm parameter.
Containers are created with commands `docker run` and `docker create`. You can see all existing containers with `docker ps -a`.

**Docker is a client-server system service**

And as such, it can freeze. If you orser to download an image, the only way to interrupt the process is to restart the service. Authors discuss how to solve it for two years already, but no solution in sight.

For example, there is a bug in 1.8.1:
```
docker@dev:~$ docker pull debian
Using default tag: latest
latest: Pulling from library/debian
2c49f83e0b13: Downloading [===================>                               ] 19.89 MB/51.37 MB
```
Press Ctrl-C, then start the download again.
```
docker@dev:~$ docker pull debian
Using default tag: latest
```
Here we are, frozen. Restart the daemon.
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
Sometimes docker does not want to die and does not release the port, and the init script does not process the boundary cases yet.
Well, just don't forget to check `sudo /etc/init.d/docker status`, `sudo netstat -ntpl` and go dance with it.

One more important notice. The order of parameters for the `docker` command is significant. If you write `docker create nginx --name=nginx`, the --name=nginx parameter is considered as a command to execute in a container, not a container name.

Now it will be easier for you to understand the officiall documentation.

