Docker myths and receipts. Mac OS X
========

* VMWare Fusion and Parallels Desktop are better then VirtualBox in performance and stability.

* At the moment of writing the Kinematic app sometimes can't do the work. If you have general experience in unix and want to use Virtual Box, I recommended installing a [test build](https://www.virtualbox.org/wiki/Testbuilds) and update it periodically rather than using a VirtualBox installed with the Docker Toolbox. Maybe it will get fixed later.

* Tired to type `$ docker full_command` every time?
Set up the [bash-completion with docker extension](http://stackoverflow.com/a/26132452) and enjoy.

* Docker-machine is a nice tool. Use it to create a Docker virtual machine and open a terminal.

```
airgri:~ gri$ docker-machine create --driver virtualbox dev
No default boot2docker iso found locally, downloading the latest release...
Downloading https://github.com/boot2docker/boot2docker/releases/download/v1.8.1/boot2docker.iso to /Users/gri/.docker/machine/cache/boot2docker.iso...
Creating VirtualBox VM...
Creating SSH key...
Starting VirtualBox VM...
Starting VM...
To see how to connect Docker to this machine, run: docker-machine env dev

airgri:~ gri$ docker-machine ssh dev
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.8.1, build master : 7f12e95 - Thu Aug 13 03:24:56 UTC 2015
Docker version 1.8.1, build d12ea79
docker@dev:~$
```

* If you see an error

> `Error creating machine: Get https://api.github.com/repos/boot2docker/boot2docker/releases: dial tcp: lookup api.github.com: no DNS servers`

add a resolving server to your /etc/resolv.conf, like "`nameserver 8.8.8.8`"


Of course, OSX will overwrite it on next reboot, but you don't need it too often.

* docker-machine (as well as boot2docker) mounts the /Users folder from a host Mac OS to the /Users folder of a virtual machine. Not to a docker container.

```
	docker@dev:~$ mount |grep Users
	none on /Users type vboxsf (rw,nodev,relatime)
```
