![Docker logo](https://upload.wikimedia.org/wikipedia/commons/7/79/Docker_(container_engine)_logo.png)
# Docker Best Practices

There are hundreds of useful tips around on how to use Docker. Unfortunately, the most useful bits of information are often buried in blog posts, talks or within long-winded documentation. I thought it might be useful to collect all these Docker tips and consolidate them in one place.

Please don't hesitate to submit a Pull Request to help the list grow!

## Table of contents
* [Docker image](#docker-image)
* [Docker container](#docker-container)
* [Docker security](#docker-security)
* [Application running within docker](#application-running-within-docker)

## Docker images

### Minimizing number of layers
Try to reduce the number of layers that will be created in your Dockerfile. Most Dockerfile instructions will add a new layer on top of the current image and commit the results.

#### Example : 
Suppose you need to get a zip file and extract it and remove the zip file. There are two possible ways to do this. 
```
COPY <filename>.zip <copy_directory>
RUN unzip <filename>.zip
RUN rm <filename>.zip
```

or in one `RUN` block:

```
RUN curl <file_download_url> -O <copy_directory> \
&& unzip <copy_directory>/<filename>.zip -d <copy_directory> \
&& rm <copy_directory>/<filename>.zip
```

The first method will create three layers and will also contain the unwanted <filename>.zip in the image which will increase the image size as well. However, the second method only creates a single layer and is thus preferred as the optimum method, as long as minimizing the number of layers is the highest priority. It has the drawback, however, that changes to any one of the instructions will cause all instructions to execute again -- something the `docker build` cache mechanism will avoid. Choose the strategy that works best for your situation.

For more detailed information see [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#minimize-the-number-of-layers).

However, sometimes executing all commands in one `RUN` block can make the script more opaque, especially when trying to mix and match `&&` and `||` statements. An alterative syntax is to use line continuation as you normally would, but explicitly switch on the shell's "exit on error" mode.
```
RUN set -e ;\
    echo 'successful!' ;\
    echo 'but the next line will exit: ' ;\
    false ;\
    causing this line not to run
# now you can use traditional shell flow of control without worry:
RUN set -e ;\
    echo 'next line will take evasive action' ;\
    if false; then \
      echo 'it seems that was false' >&2 ;\
    fi ;\
    echo 'and the script continues'
```

### Minimizing layer size
Some installations create data that isn't needed. Try to remove this unnecessary data within layers:

```
RUN yum install -y epel-release && \
    rpmkeys --import file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 && \
    yum install -y --setopt=tsflags=nodocs bind-utils gettext iproute\
    v8314 mongodb24-mongodb mongodb24 && \
    yum -y clean all
```

For more detailed information, see [container best practices](http://docs.projectatomic.io/container-best-practices/#_clear_packaging_caches_and_temporary_package_downloads).


### RUN-only environment variables

If one needs an environment variable set during a `RUN` block, but it is either unnecessary, or potentially disruptive to downstream images, then one can set the variable in the `RUN` block instead of using `ENV` to declare it globally in the image:

```
RUN export DEBIAN_FRONTEND=noninteractive ;\
    apt-get update ;\
    echo and so forth
```

### Tagging
Use tags to reference specific versions of your image.

Tags could be used to denote a specific Docker container image. Hence, the tagging strategy must include a unique counter like `build id` from a CI server (e.g. Jenkins) to help with identifying the correct image.

For more detailed information see [The tag command](https://docs.docker.com/engine/reference/commandline/tag/).


### Log Rotation
Use `--log-opt` to allow log rotation. This helps if the containers you are creating are too verbose and are created too often due to a continuous deployment process.

For more detailed information, see [the log driver options](https://docs.docker.com/engine/admin/logging/overview/#/json-file-options).


## Docker container
### Containers should be disposable/ephemeral
The creation and startup time of a container should be as small as possible. Furthermore, a container should shut down gracefully when a `SIGTERM` is received. This makes it easier to scale up or down. It also makes it easier to remove unhealthy containers and to spawn new ones.

For more detailed information, see [The Twelve-Factor App](https://12factor.net/disposability) and [best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#containers-should-be-ephemeral).

### Containers should have its PID1 to be a Zombie reaper
Container processes may not respond to an immediate command of `docker stop`. If our main process is determined by our command or execution to be in the `entrypoint` and is unable to manage Reaping, our `docker stop` won't work. This is because no `SIGINT` would be able to reach the appropriate process. We should, whenever possible, use container images that extend Reaping management.

So, the question is: does the process you execute in your entrypoint register signal handlers? A good way to figure this out might be to check whether your process responds properly to `docker stop` (or if it waits for ten seconds before exiting). In this case, tools like [tini](https://github.com/krallin/tini) can help fix this problem.

[Complete explanation by Krallin, creator of Tini](https://github.com/krallin/tini/issues/8)

### One container, One responsibility, One process
If a container only has one responsibility (which should in almost all cases involve one process), it makes it much easier to scale horizontally or reuse the container in general.

[Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#run-only-one-process-per-container).

### Store share data in volumes
If services need to share data, use shared volumes. Make sure that services are designed for concurrent data access (read and write).

For more detailed information see [Manage data in containers](https://docs.docker.com/engine/tutorials/dockervolumes/).

## Docker registry
### Garbage collection
Remove unnecessary layers in your registry. For more detailed information see [Garbage Collection](https://github.com/docker/docker.github.io/blob/master/registry/garbage-collection.md).

## Docker security
### Limit access from network
Only open ports which are needed in production; containers should only be accessible by other containers that need them. Groups of containers should use their own sub-network.

```
$ docker network create --driver bridge isolated_nw
$ docker run --network=isolated_nw --name=container busybox
```

### You can't trust anyone
> This may seem like a harsh thing to say, but in a climate where even baby monitors and light-bulbs can be taken over to participate in DDoS attacks, we need to get smart.

#### Don't use an image unless it's official
There are no official images for ARM at this time, but `resin/rpi-raspbian` is used by thousands of devices and curated by resin.io. You can use it as your base image.

#### Don't run any binaries that you didn't compile yourself
It's way better to compile your own binaries than relying on tar.gz files provided by someone you know nothing about.

> Docker Security by Adrian Mouat coins the term "poison image" for an image tainted with malware.

If you want to create a Docker image for software such as Prometheus.io, Node.js or Golang, head over to their download page and locate the official binary package, then add it into one of the base images we covered above.

If no binary exists, take the time to rebuild it from source and don't take any risks. Google search the build instructions if you run into issues; they can often be found quickly.

Docker Captain Alex Ellis has provided a set of Dockerfiles for ARM on Github for common software such as Node.js, Python, Consul and Nginx: [alexellis/docker-arm](https://github.com/alexellis/docker-arm)

For more detailed information see [5 things about Docker on Raspberry Pi](http://blog.alexellis.io/5-things-docker-rpi/) by Docker Captain Alex Ellis @alexellisuk.

### Limit access to filesystem
Start your container in a read-only mode with `--read-only`. You should also do this with volumes by adding `:ro`. This makes it harder for attackers to corrupt your container.

```
$ docker run --read-only ...
$ docker run -v /my/data:/data:ro ...
```

### Limit memory
You should limit the memory resources of your container against DoS attacks or/and against applications with memory leaks. This protects your host system and other containers. You should use `-m` and `--memory-swap` to limit memory.

```
$ docker run -m 128m --memory-swap 128m ...
```

### Vulnerability scanning
Use a security scanner for your images; this comprises a static analysis of software-level vulnerabilities.

For more detailed information see [Docker Security Scanning](https://docs.docker.com/docker-cloud/builds/image-scan/) or [Clair](https://github.com/coreos/clair).

### Switch to non-root-user
If your service does not need root privileges, **do not use root**! Consider the **Principle of Least Privilege** applied to Docker containers. Create a new user and switch the user with `USER`.

```
RUN groupadd -r myapp && useradd -r -g myapp myapp
USER myapp
```

For more detailed information see [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#user).

### Drop unused capabilities
Linux allows you to fine-tune the permissions of the application inside the container. By default, process are privileged and have the following capabilities: `chown`,` dac_override`,` fowner`,` fsetid`,` kill`,` setgid`,` setuid`,` setpcap`,` net_bind_service`,` net_raw`,` sys_chroot`,` mknod`,` audit_write`,` setfcap` (see `man capabilities`). Unused capabilities can be dropped the following way:

```
#  docker run -d --cap-drop=all --cap-add=setuid --cap-add=setgid <image>
```
For more detailed informations see [Secure Your Containers (rhelblog)](http://rhelblog.redhat.com/2016/10/17/secure-your-containers-with-this-one-weird-trick/).

### Passing credentials and secrets
Most applications have to handle with secrets and credentials. The most common way to pass secrets and credentials is to specitfy them as environment variables at container runtime. But if you do this, keep the following downside in mind:
* If you commit the container the image will contains the secrets
* With `docker inspect` the environment variables can be read out
* Although squashing removes intermediate layers from the final image, secrets from those layers will still be present in the build cache

For more detailed information, see [container best practices](http://docs.projectatomic.io/container-best-practices/#_passing_credentials_and_secrets).

## Application running within docker
### Logging to stdout
To handle logs in your service easily, write all your logs to `stdout`. This uniform process makes it easy for the docker daemon to grab this stream.

For more detailed information see [The Twelve-Factor App](https://12factor.net/logs) and [Configure logging drivers](https://docs.docker.com/engine/admin/logging/overview/).
