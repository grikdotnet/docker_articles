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

