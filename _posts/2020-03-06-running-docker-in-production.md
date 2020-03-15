---
layout: post
title: Running Docker in Production
tags: [docker, devops , java]
subtitle: Things you should know when running Docker in production.
--- 

If you haven't read about the bare essentials of running Java applications with Docker, you can find it [here][Part1]. In this post we'll dive deeper into a few advanced concepts that need to be understood when dealing with docker application is a production environment.

## Outline
- Reading application logs.
- Understanding memory limits.
- Monitoring your 

## Reading application logs
By default, all files written by your application running inside a Docker container will be written to a writeable layer. As we've previously understood with layers, this is not good thing. This will increase the size of your container in addition to being hard to access. Another downside is the fact that this data will not be persisted once the container is stopped.

If there's one thing you need to get right when running applications in production it's good application logging. There will be times when logs are your only hope in reproducing or tracking down application issues. This make is essential to have easily accessible log files. The two main approaches we'll focus on today are bind mounts and volumes.

![Types of mounts]({{ site.url }}/img/types-of-mounts.png)

_[Image Source][mount-img-src]_

### Bind Mounts
Using a bind mount is the simplest option available, it mounts a file or directory on the host system into the container. This is ideal during development, however ina production environment, some caution will need to be exercised. Using a bind mount will let you directly modify file or directories on the host system, if use incorrectly it could have undesirable consequences.

To get started we'll use our sample [Spring Boot REST application][spring-boot-docker-git] from part 1. In order to log to a file I've added a logback configuration file.

{% highlight xml linenos %}
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>/tmp/app/testFile.log</file>
        <append>true</append>
        <!-- set immediateFlush to false for much higher logging throughput -->
        <immediateFlush>true</immediateFlush>
        <!-- encoders are assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
        <encoder>
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="DEBUG">
        <appender-ref ref="FILE" />
    </root>
</configuration>
{% endhighlight %}

This will redirect all log statements to `/tmp/app/testFile.log` within the container, we will then mount a folder on the host OS in order to view the log file.

We'll use the `--mount` flag since the latest Docker document recommends using over the older `-v` flag. It's also a lot more verbose and easier to understand.

_Note: Unlike the `-v` flag `--mount` will not create a directory if it doesn't exist on the host system and will throw and exception. Hence when using the `--mount` option you need to ensure the path on the host exists._


{% highlight bash linenos %}
# Build New Container with Tag vlogs
docker build -t oliver/restapplogs:vlogs .

# Run Container with --mount flag
docker run \
--mount type=bind,source=/tmp/docker/logs/,target=/tmp/app/ \
--name restapp \
--publish 8080:8080 oliver/restapplogs:vlogs

{% endhighlight %}

Now you should be able to navigate to `/tmp/docker/logs/` and view the log file.

You can also use the `docker inspect` command and verify everything is configured correctly by inspecting the Mount section.

Let's run this for the container we've just run: `docker inspect restapp`

```
"Mounts": [
            {
                "Type": "bind",
                "Source": "/tmp/docker/logs",
                "Destination": "/tmp/app",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
```



### Volumes
Next, let's explore Volumes, they are managed by Docker, unlike bind mounts you cannot mount any file or directory. Here are some of the advantages of using volumes over bind mounts listed in the Docker documentation.

- Volumes are easier to back up or migrate than bind mounts.
- You can manage volumes using Docker CLI commands or the Docker API.
- Volumes work on both Linux and Windows containers.
- Volumes can be more safely shared among multiple containers.
- Volume drivers let you store volumes on remote hosts or cloud providers, to encrypt the contents of volumes, or to add other functionality.
- New volumes can have their content pre-populated by a container.

You may choose to create a volume independent of containers using the `docker volume create <volume-name>` command or create one on the fly when running the container. I'll stick to the latter for now. Once again we'll use the `--mount` flag albite with slightly different options.

{% highlight bash linenos %}
# Run Container with --mount flag
docker run \
--mount type=volume,source=rest-app-vol,target=/tmp/app/ \
--name restapp-volumes \
--publish 8080:8080 oliver/restapplogs:vlogs
{% endhighlight %}

Once the container has started, let's check if the volume is created.

{% highlight bash linenos %}
docker volume ls
# Output
DRIVER              VOLUME NAME
local               rest-app-vol
{% endhighlight %}

As you can see we have a volume named `rest-app-vol`, now let's inspect the details of that volume.
{% highlight bash linenos %}
docker volume inspect rest-app-vol
# Output
[
    {
        "CreatedAt": "2020-03-15T13:30:08Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/rest-app-vol/_data",
        "Name": "rest-app-vol",
        "Options": null,
        "Scope": "local"
    }
]
{% endhighlight %}

Docker has mounted the folder `/var/lib/docker/volumes/rest-app-vol/_data` to the container, you may also inspect the container and look at the Monunt section as we did in the previous section.

_Note: If you are running Docker for Mac as I am, you will not find anything at this location. This is because Docker for Mac runs withing a VM, in order to get to the volume you'll have to run `screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty` now you are inside the vm and should be able to navigate to to `/var/lib/docker/volumes/rest-app-vol/_data`. On a side note since you are starting a `screen` I'd highly recommend you learn the basics of you using `screen`. `Crtl+a` `d` to exit._


## Refernces
1. [Manage Data in Docker][docker-storage]



[Part1]:{% post_url 2020-02-29-running-java-apps-with-docker %}
[docker-storage]: https://docs.docker.com/storage/
[mount-img-src]: https://docs.docker.com/storage/images/types-of-mounts.png
[spring-boot-docker-git]: https://github.com/oliversavio/gs-rest-service/tree/master/complete