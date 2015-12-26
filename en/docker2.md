Semi-automatic self-homing gun
========

**How not to use Docker.**

There is always a difference between examples from manual and real life usage. When you open a page of andy official software like [Nginx](https://hub.docker.com/_/nginx/), you will find the "How to use this image" section.
We are offered to create the image based on the official one, copying our files into it, set up the port mapping out to the world, and mount the folder with configs.
```
FROM ...
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
```
Same advice is given in [PHP](https://hub.docker.com/_/php/), [Pyhon](https://github.com/docker-library/docs/blob/master/python/README.md), Ruby, and other images. 

They don't write what happends after the prince married the cinderella, usually. After creating an image it is supposed to be stored in a registry. You can set up your own [Docker Registry](https://www.docker.com/docker-registry) or pay for a private registry in a docker hub. Then we deploy our application to our servers and workstations right from the registry. Pretty straighforward, but there is a problem in a long-run.

Speaking the OOP language, we are offered to inherit the Model from the View in a single god class. Can the principles of OOP design be applied to the components of the system as a whole? Yes, because we face the same consequences. If we create a star object, it becomes hard to support.

A registry is a SPOF, and if it goes down, the work stops. So, you wrap the registry itself into an image and deploy several instances, setting up rsync to keep copies up to date, make backups and monitor availability with a failover switch. This is ok, but now it's a bit more complex then 3 lines in a Dockerfile.
You stick to a docker protocol and tools to deploy your application, getting a vendor lock-in that the Docker was suppose to help with.
At last, you have to rebuild, redeploy and restart the whole large image when changing an application, updating PHP, Nginx, or any library.

Fortunately, the solution is well-known same long as the problem. Gang of four, Java creator James Gosling and Martin Fowler are harping on for 20 years: use composition instead of inheritance.
That's why teams are putting beside containers with different services, create an adapter, a data transfer object, and bind them with a config.

I use inheritance to extend the features of existing images, keep incapsulation and taking into accounting the [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle). For example, to add extensions to php I inherit from an official php image just like the docker hub suggests.

**Monkeypatch**

Let's make a web stack with Nginx and PHP 7.
```
~$ docker pull nginx
...
~$ docker pull php:7-fpm
Status: Downloaded newer image for php:7-fpm
```

Now I've got two other people classes and need to bind them with dependency injection. The easiest way to add dependencies to the other people code is for sure, [monkeypatching](https://ru.wikipedia.org/wiki/Monkey_patch)!
At first I create containers. Remember the [second hard thing in computer science](http://martinfowler.com/bliki/TwoHardThings.html) - give containers intelligible names, it will be of use to make containers interact with each other.
```
~$ docker create --name=php7 php:7-fpm
3d1b737edfcc3f1102fa54c91f9120da4b86d8cbba3092b6f80156c0e31b4d8f
~$ docker create --name=nginx nginx
80be81b27e012fd061ff4b682f0b7b8803500bc38a4b9f787f91661603b2d4b7
```

### PHP

Starting with PHP. One can see the location of config files in [Dockerfile](https://github.com/docker-library/php/blob/f5e091ac3815dce80ca496298e0cb94638844b10/7.0/fpm/Dockerfile):

```Dockerfile
	ENV PHP_INI_DIR /usr/local/etc/php
	   --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
	WORKDIR /var/www/html
	COPY php-fpm.conf /usr/local/etc/

```

Copying the contents of the folder with php configuration files.
```
~$ mkdir monkeypatch
~$ cd monkeypatch/
$ docker cp php7:/usr/local/etc localetc
$ ls localetc/
pear.conf		php			php-fpm.conf		php-fpm.conf.default	php-fpm.d
$ ls localetc/php
conf.d
```
Maintainers placed the file php-fpm.conf, but did not include the default php.ini. We'll have to take it from the php sources.

	$ docker cp "$PHP7:/usr/src/php/php.ini-development" localetc/php/php.ini

Edit the config according to your requirements. What folder does PHP search extensions in?
One can find out by running php in a temporary container.
```
$ docker run --rm php:7-fpm php -i |grep extension_dir
extension_dir => /usr/local/lib/php/extensions/no-debug-non-zts-20141001 => /usr/local/lib/php/extensions/no-debug-non-zts-20141001
$ docker run --rm php:7-fpm ls /usr/local/lib/php/extensions/no-debug-non-zts-20141001
opcache.a
opcache.so
```
The only extension included is opcache. To enable it, add options to php.ini
```
$ echo extension_dir = "/usr/local/lib/php/extensions/no-debug-non-zts-20141001" >>  localetc/php/php.ini
$ echo zend_extension = opcache.so >> localetc/php/php.ini
```
Adding more extensions is described in the docker hub page for an image.

To use my config I recreate the php container from the image and mount config folders. Path to the mounted folder should be defined from server root - the docker service does not know what folder is the docker client called from.
```
$ docker rm php7
php7
$ docker run -v "$(pwd)/localetc:/usr/local/etc" --name=php7 php:7-fpm php -i |grep Configuration
Configuration File (php.ini) Path => /usr/local/etc/php
Loaded Configuration File => /usr/local/etc/php/php.ini
```
Now I can recreate the php7 container with a test application. Image maintainers did not prepare php-fpm to run as a daemon, so we'll have to run it in a background not releasing standard i/o streams.
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
Meanwhile for debug I leave output from php-fpm to my console.

### NGINX

Starting Nginx is pretty straightforward. Copying the config folder to my disk:
```
$ docker cp nginx:/etc/nginx .
```
Edit nginx.conf from the `nginx/` folder of a container, fastcgi_params to taste, and create a config file for your web site `nginx/conf.d/`.
The most important to link nginx with php is to define the php container name as a host for fastcgi_pass, and pathes in the root and SCRIPT_FILENAME directives should resolve in the php7 container.
```Nginx
    location ~ \.php$ {
        fastcgi_pass   php7:9000;
        fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
```

Mounting configs inside nginx and starting with 80 port mapping to the local 8080. 

```
$ docker rm nginx
$ docker run -v "$(pwd)/nginx:/etc/nginx" -p 8080:80 --name=nginx nginx &
$ curl 127.0.0.1:8080/test.php
172.17.0.65 -  29/Aug/2015:15:50:29 +0000 "GET /test.php" 200
Hello world! 7.0.0RC1
```
Rock'n'Roll!

In version 1.7 the `docker run` command required the --link parameter to resolve the name of another container. In 1.8 it works without this parameter.
