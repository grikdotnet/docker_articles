Docker myths and receipts. Mac troubles
========

Tired to type `$ docker full_command` every time?
Set up the [bash-completion с дополнением docker](http://stackoverflow.com/questions/26132451) and enjoy.

If you have a general experience in unixes, it is recommended to install a VirtualBox [test build](https://www.virtualbox.org/wiki/Testbuilds) and update it periodically rather then using the one from the Docker Toolbox.

Docker-machine is a nice tool. Use it to create a Docker virtual machine and open a terminal.

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

If you see an error
>Error creating machine: Get https://api.github.com/repos/boot2docker/boot2docker/releases: dial tcp: lookup api.github.com: no DNS servers

add a resolving server to your /etc/resolv.conf, like "`nameserver 8.8.8.8`"
Of course, OSX will overwrite it on next reboot, but you don't need it too often.

