# Docker

## Settings

```shell
docker version
```
Check your versions and that docker is working

```shell
docker info
```
Shows most configurations values of the engine

---

## Containers

```shell
docker container run
docker container run --publish 80:80 nginx
```
Start a new container from an image:
1. Downloaded image 'nginx' from Docker Hub
2. Started a new container from that image
3. Opened port 80 on the host IP
4. Routes that traffic to the container IP, port 80

```shell
docker container run --publish 80:80 --detach nginx
```
Docker runs the container in the background (get back the container id)

```shell
docker container ls
docker ps
```
List running containers
```shell
docker container stop <container_id>
```
Stop a running a container
```shell
docker container ls -a
docker ps -a
```
List all containers
```shell
docker container run -p 80:80 -d --name webhost nginx
```
Naming containers
```shell
docker container logs <container_name / container_id>
docker container top
```
Check logs from running containers in the background
```shell
docker container rm <container_id>
```
Remove (delete) one or more containers

```shell
docker container rm -f <container_id>
```
Force stop and remove (delete) one or more running containers

#### What happens in 'docker container run'?

1. Looks for that image locally in image chache, doesn't find anything
2. Then looks in remote image repository (defaults to Docker Hub)
3. Downloads the latest version (nginx:latest by default) and stored in image chache
4. Creates new container based on that image and prepares to start
5. Gives it a virtual IP on a private network inside docker engine
6. Opens up port 80 on host and forwards to port 80 in container
7. Starts container by using the CMD in the image Dockerfile

#### Assignment

* Run a `nginx`, `mysql`and `httpd` (apache) server
* Run all of them `--deatch` and name them
* nginx should listen on `80:80`, httpd on `8080:80`and mysql on `3306:3306`
* When running `mysql` use the `--env`option to pass in `MYSQL_RANDOM_ROOT_PASSWORD=yes`
* Use `docker container logs` on mysql to find the random password

```shell
# HOST:CONTAINER
docker container run -d --name db -p 3306:3306 -e MYSQL_RANDOM_ROOT_PASSWORD=yes mysql
docker container logs db
docker container run -d --name webserver -p 8080:80 httpd
docker container run -d --name proxy -p 80:80 nginx
docker container ls
curl localhost
curl localhost:8080
docker container stop db webserver proxy
docker container container ls -a
docker container rm db webserver proxy
```
#### What's going on in containers?

* `docker container top` - process list in one container
* `docker container inspect` - details of one container config
* `docker container stats` - performance stats for all Containers

#### Getting shell inside containers

Containers only run as long as the command CMD that it run in the start up runs. If we run `bash`in container it will exit as soon as the container is created and runs the `bash` command.

```shell
docker container run -it --name proxy nginx bash
```

* `-t` (pseudo -tty): simulates a real terminal, like what SSH does
* `-i` (interactive): keep session open to receive terminal input

If you run a `ubuntu` image, the default start command is `bash`. Once you exit the shell in the docker container run above, the container will stop.

```shell
docker container run -it --name ubuntu ubuntu
# inside the shell
apt-get update
apt-get install -y curl
curl google.com
exit
```
When we run `exit`, the container stops. But if we run this container again, we will have `curl` installed in it. A new container from `ubuntu` will not have `curl` installed.

```shell
docker container start -ai ubuntu
```

#### Run additional process in running container

What if we want to actually see the shell inside a running container that's already running something?

```shell
docker container exec -it mysql #!/usr/bin/env bash
# inside the shell
ps aux
exist
# mysql will continue running (this is just a side process)
```

#### Alpine

Alpine is a super light security-focused linux distribution.

```shell
docker pull alpine
docker container run -it alpine bash
# alpine does not have bash
docker container run -it alpine ssh
```

#### Docker networks

```shell
docker container run -p 80:80 --name webhost -d nginx
```

Remember publishing ports is always in HOST:CONTAINER format. Port 80 from HOST forwards traffic to port 80 in the container.

```shell
docker container port webhost
```

By default, it is not true that the IP address of the host is shared with the container. Getting the IP address of the container

```shell
# IP container
docker container inspect --format '{{ .NetworkSettings.IPAddress }}' webhost
# IP host (inet)
ifconfig en0
```

##### CLI Management

* `docker network ls` - show networks
* `docker network inspect` - inspect a network
* `docker network create --driver` - create a network
* `docker network connect` - attach a network to container
* `docker network disconnect` - detach network from container

`--network bridge` is the default docker virtual network,  which is NAT'ed behind the Host IP.

`--network host` it gains performance by skipping virtual networks but sacrifices security of container model

`--network none` removes eth0 and only leaves you with localhost interface in container

Note: lessons 28 - 33 (content continues)

---

## Container images

#### What's in an image

* App binaries and dependencies
* Metadata about the image data and how to run the image
* Not a complete OS. No kernel, kernel modules (e.g. drivers). The host provides the kernel.
* Small as one file (your app binary) like a golang static binary

Image tags normally refer to the image version.

#### Image layers

Images are designed by building layers.

```shell
docker image history <image_id>:<image_tag>
```
Shows layers of changes made in image. Every image starts with a blank layer (scratch). We don't need to download layers that we already have cached.

We are never storing the entire stack of image layers more than once if there are really the same layers.

```shell
docker image inspect
```
---

## Dockerfile

The Dockerfile is a recipe.

```shell
docker build -f some-dockerfile
```
Each one of these following sentences is an actual layer in our docker image. So the order of them actually matter, because it does work top down.

* `FROM` is the base image.

Package manager: PM's like apt and yum are one of the reasons to build containers `FROM` Debian, Ubuntu, Fedora or CentOs.

* `ENV` (environment variables) one reason they were chosen as preferred way to inject key/value is they work everywhere, on every OS and config.

* `RUN` run command can run shells scripts, any commands that you can run inside the container... Different commands can be chained together with (&&) and thus they all fit in the same layer.

Logging: docker already handles all the logging for us. All we need to make sure is that everything we want to be captured in the logs is spit to stdout and stderr. Docker will handle the rest.

```docker
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
```

* `EXPOSE` by default, no TCP or UDP ports are open inside a container. It doesn't expose anything from the container to a virtual network unless we list it here. For a web and proxy server, we would expose to `80` and `443`. This expose command does not mean these port are going to be opened automatically on our host. That's what the `-p` command is whenever we use `docker run`.

* `CMD` is a required parameter that is the final command that will be run every time you launch a new container from the image, or everytime you restart a stopped container.

#### Running Dockerfiles

The dockerfile needs to be built before run.

```shell
docker image build -t <image_name> .
```
When docker builds the image, all layers are built and identified by a hash, so it keeps all the layers in the build cache so that next time we build this thing, if that line hasn't change in the dockerfile, it's not going to rerun it.

This brings up the point about ordering of your lines in your Dockerfile. Because, if you get things out of order, for instace, if you copied the code in... let's say you're building a website. If you're copying the software code that you're creating at the very beginning of the file, then every time you change a source file and you rebuild, it's going to have to build the entire Dockerfile again. It is importat that you usually keep the things at the top of your Dockerfile that change the least and then the things that change the most at the bottom.


#### Persistent data

* Key concepts with conatiners: immutable, ephemeral
* Learning and using Data Volumes
* Learning and using Bind Mounts

__Volumes__ are a configuration option for a container that creates a special location outside of the container's union file system to store unique data. This preserves it accross container removals and allows us to attach it to whatever container we want and the container just sees it like a local file path.

__Bind Mounts__, which are simplu us sharing or mounting a host directoru, or file, into a container. It will just look like a local file path, or a directory path, to the container. It won't actually know that it is coming from the host.

#### Volumes

Example (mysql image Dockerfile):

```dockerfile
VOLUME /usr/lib/mysql
```

This is the default location of the MySQL databases. This image is programmed in a way to tell Docker that when we start a new container from it, to actually create a new volume location and assign it to this directory in the container. Which means any files that we put in there, in the container, will outlive the container until we manually delete the volume. Volumes need manual deletion, you can't clean them up by removing the container. There's an extra step. That's just for insurance really, because the whole point of a volume command is to say that this data is particularly important, at least much more important than the container itself.

```shell
docker container run -d --name mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=True mysql
docker container inspect mysql
```
We can check under "Mounts" the "Destination". What this is, is actually the running container getting its own unique location on the host, to store that data, and then it's in the background, ampped or mounted, to that location in the container, so that the location in the container actually just thinks it's writting in `/var/lib/mysql` and, in this case, we can see that the data is actually living in the "Source" location on the host `/var/lib/docker/volumes/<volume_id>/_data`.

```shell
docker volume ls
docker volume inspect <volume_id>
```
If you were on a Linux machine you could navigate to that path and find the data stored in that volume. Since we are on a Mac, remember that Docker for Mac is actually doing a little bit in the background that's hidden from us where it's actually creating a Linux VM and so this data is technically inside that Linux VM. (you're not going to be able to go to that path on your Mac machine and see the data). We will see with mounted volumes how we can get around such limitations as well.

If you do `docker container rm -f mysql` and then you check  `docker volume ls`, your volume is still there. Data is still safe. The databases outlive the executable. __Named volumes__ are a friendly way to assign volumes to containers.

```shell
docker container run -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=True -v /var/lib/mysql mysql
```
This is the same as the `VOLUME` command in the Dockerfile. Instead we can put a name in front of it:

```shell
# named volume
docker container run -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=True -v mysql-db:/var/lib/mysql mysql
docker volume ls
```

Now the new container is useng a new volume name `mysql-db`.

```shell
# required to do this before "docker run" to use custom drivers and labels
docker volume create
```
If we create them from a docker container run command at runtime, and we can create them by specifying in them in the Dockerfile, there's only a few cases where you'd want to create it ahead of time. Any driver options that we want to use the `-o` on. Sometimes, in special cases, you do need to create the Docker volume ahead of time, but usually for your local development purposes, just specifying it in a Dockerfile or at the run command is fine.

#### Bind Mount

A bind mounting is basically a mapping of the host file or directory to a container file or directory. In the background, it's basically just having the two locations point to the same physical locations on disk. If there are any files in the container that you map to the host files to, the host files win. It doesn't actually delete the files in the container that it overwrote, becautse it's not really overwriting. Because bind mounts are usually host specific, they need specific data to be on the hard drive of the host in order to work. You have to use them at runtime when you use the `docker container run` command (can't use in Dockerfile).

```shell
... run -v /Users/bret/stuff:path/container
```

It's really the `-v` that we used before, only on the left side of the colon, we are actually putting a full path rather than just a name. And the way Docker actually can tell the difference between the named volume and the bind mount, is the bind mount starts with a `/`.

As long as you have left colon and right sides, you can really map anything you want from the host into the container, and you can also specify things like read-only.

```shell
#  bind mounting
docker container run -d --name nginx -p 80:80 -v $(pwd):/usr/share/nginx/html nginx
docker container exec -it nginx #!/usr/bin/env bash
cd /usr/share/nginx/html
ls -al
```
