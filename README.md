# Docker Best Practices
![Docker logo](https://upload.wikimedia.org/wikipedia/commons/7/79/Docker_(container_engine)_logo.png)

There are so much useful tips out there, how to use docker. This great tips are part of blog posts, talks or within a documentation. I tought it might be useful, to collect all this docker best pratices, which are distributed all over these resources.

I would be very happy to receive some Pull Request to let the list grow.

## Table of contents
* [Docker image](#docker-image)
* [Docker container](#docker-container)
* [Docker security](#docker-security)
* [Application running within docker](#application-running-within-docker)

## Docker image
### Minizing layer size
Some installations create data, which are not be needed. Try to remove this data in the same layer.

```
RUN yum install -y epel-release && \
    rpmkeys --import file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 && \
    yum install -y --setopt=tsflags=nodocs bind-utils gettext iproute\
    v8314 mongodb24-mongodb mongodb24 && \
    yum -y clean all
```

For more detailed information see [Container best practices](http://docs.projectatomic.io/container-best-practices/#_clear_packaging_caches_and_temporary_package_downloads).

### Minizing number of layers
Try to recude the number of layers, which will be created in your Dockerfile. Most Dockerfile instructions will add a new layer on top of the current image and commit the results. 

For more detailed information see [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#minimize-the-number-of-layers).

### Tagging
Use tags to reference specific versions of your image.

Tags could be used to denote a specific Docker container image. Hence, the tagging strategy must include leveraging a unique counter like `build id` from a CI Server (eg: Jenkins) to help with identifying the right Image. 

For more detailed information see [The tag command](https://docs.docker.com/engine/reference/commandline/tag/).


### Log Rotation

Use `--log-opt` to allow log rotation if the containers you are creating are too verbose and are created too often thanks to a continuous deployment process. 

For more detailed information see [the log driver options](https://docs.docker.com/engine/admin/logging/overview/#/json-file-options).


## Docker container
### Containers should be disposable/ephemeral
The creation and startup time of a container should be as small as possible. In addition a container should shut down gracefully when the container receive a SIGTERM. This makes it easier to scale up or down. It also makes it easier to remove unhealthy containers and spawn new ones.

For more detailed information see [The Tweleve-Factor App](https://12factor.net/disposability) and [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#containers-should-be-ephemeral).

### Containers should have its PID1 to be a Zombie reaper
Container processes may not be responding to an inmmediate command of `docker stop`
If our main proccess determinated by our CMD or execution in the `entrypoint` it is uncapable to manage Reaping, our `docker stop` won't work as no `SIGINT` would be able to reach the appropriate process. We should whenever posible use container images that are extend Reaping management.

So, the question is does the process you exec in your entrypoint registering signal handlers? A good way to figure this out might be to check whether your process responds properly `docker stop` (or if it waits for 10 seconds before exiting). In this last case tools like [tini](https://github.com/krallin/tini) fix this problem.

[Complete explanation by Krallin, creator of Tini](https://github.com/krallin/tini/issues/8)

### One Container - One Responsibility - One process
If a container only has one responsibility, which should in almost all cases one process, it makes it much easier to scale horizontally or reuse the container in general.

[Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#run-only-one-process-per-container).

### Store share data in volumes.
If you services need to share data, use shared volumes. Please make sure that your services are designed for concurrency data access (read and write).

For more detailed information see [Manage data in containers](https://docs.docker.com/engine/tutorials/dockervolumes/).

## Docker registry
### Garbage collection
Remove unnecessary layers in your registry.

For more detailed information see [Garbage Collection](https://github.com/docker/distribution/blob/master/docs/garbage-collection.md).

## Docker security
### Limit access from network
Only open the ports, which are needed in prodution and containers should only be accessible by other containers that need them. So groups of containers should use their own network.

```
$ docker network create --driver bridge isolated_nw
$ docker run --network=isolated_nw --name=container busybox
```

### You can't trust anyone
> This may seem like a harsh thing to say - but in a climate where even baby monitors and lightbulbs can be taken over to participate in DDOS attacks we need to get smart.

#### Don't use an image unless it's official
For ARM, there are no truly official images for now, but resin/rpi-raspbian is used by thousands of devices and curated by resin.io, you can use it as your base image.

#### Don't run any binaries you that didn't compile yourself
It's way better to compile our binary than relying on a tar.gz provided by someone you know nothing about on the internet.

> Docker Security by Adrian Mouat coins the term poison image for an image tainted with malware.

If you want to create a Docker image for software such as Prometheus.io, Node.js or Golang then head over to their download page and locate the official binary package - then add it into one of the base images we covered above.

If no binary exists then take the time to rebuild from source and don't take any risks. Google the build instructions if you run into issues - they can often be found from a 5-minute search.

Docker Captain Alex Ellis have provided a set of Dockerfiles for ARM on Github for common software such as Node.js, Python, Consul and Nginx: [alexellis/docker-arm](http:s//github.com/alexellis/docker-arm)

For more detailed information see [5 things about Docker on Raspberry Pi](http://blog.alexellis.io/5-things-docker-rpi/) by Docker Captain Alex Ellis @alexellisuk

### Limit access to filesystem
If not necessary start your container in a read-only mode with `--read-only`. You also should do the same with volumnes with adding `:ro`. This makes it harder for attackers to corrupt your container.

```
$ docker run --read-only ...
$ docker run -v /my/data:/data:ro ...
```

### Limit memory
You should limit the memory resource of your container against DoS attacks or/and against applications with memory leaks. This protects your host system and other containers. You should use `-m` and `--memory-swap` to limit memory.

```
$ docker run -m 128m --memory-swap 128m ...
```

### Security scanning
Use a security scanner for your images for a static analysis of vulnerabilities.

For more detailed information see [Docker Security Scanning](https://docs.docker.com/docker-cloud/builds/image-scan/) or [Clair](https://github.com/coreos/clair).

### Swtich to non-root-user
If your service do not need root privileges, then do not run it with it. Create a new user and switch the user with `USER`.

```
RUN groupadd -r myapp && useradd -r -g myapp myapp
USER myapp
```

For more detailed information see [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#user).

## Application running within docker
### Logging to stdout
To handle logs in your service easily, write all your logs to `stdout`. This uniform process makes it easy for docker deamon to grab this stream.

For more detailed information see [The Tweleve-Factor App](https://12factor.net/logs) and [Configure logging drivers](https://docs.docker.com/engine/admin/logging/overview/).
