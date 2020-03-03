---
layout: post
title:  Bare essentials of running Java applications with Docker
date:   2020-02-29 12:00:00 +0530
categories: spark
description: Using Docker containers to run your Java / JVM Applications
comments: false
--- 


Hopefully you are here because you are convinced that you want to use Docker to run your Java / JVM application. In this post I will go over the most important concepts you need to know in order to run your Java applications within Docker containers in a production environment.

## Prerequisites
 - You have Docker installed and setup.
 - Familiarity with packaging Java applications as an executable uber/fat jar.
 - Familiarity with basic Docker commands and concepts.

## Outline
- [Building our executable jar](#building-our-executable-jar)
- [Choosing a base container image](#choosing-a-base-container-image)
- [Official Docker Image](#official-docker-image)
- [Alpine Linux Image](#alpine-linux-image)
- [Building our Dockerfile](#building-our-dockerfile)
- [Building our Docker image](#building-our-docker-image)
- [Running the Docker Container](#running-the-docker-container)
- [Running a Spring Boot REST Application with Docker](#running-a-spring-boot-rest-application-with-docker)

## [Building our executable jar](#building-our-executable-jar)
I have packaged my simple java application using the Maven shade plugin to build an executable jar. The entire source code may be found [here][java-docker-git]. The only thing this simple application does is print "Hello Docker!!".

## [Choosing a base container image](#choose-base-container)
The official Docker documentation describes a container image as _"a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings."_

One of the most important concepts you need to understand with docker images is the concept of layering. For our simple java application here, we could start off with Linux as a base image, then install java and the related dependencies onto it. Each step we use to install custom software / utilities will create another layer. Alternately, I could save myself the work of manually installing Java and use a Java base image to run my application. This Java base image will automatically include Linux, much like the way Maven dependencies work.

So now, how do you choose? One rule of thumb I use is to keep my `Dockerfile` as lean as possible and find an official base image as close to what I require to run my application. But what is an __"Official"__ docker image?

## [Official Docker Image](#official-docker-image)
If you do a quick search on Docker Hub for a container image, your search may throw up a couple of results. Anyone is allowed to register with Docker Hub and publish images. If you are a seasoned veteran of Docker, you could technically use any image you wish, provided you are adept at inspecting the `Dockerfile` and validating that you aren't getting any Easter eggs :-D.

For most people I'd highly recommend sticking to Official images. As mentioned on Docker Hub, ["Official Images"][official-images] are a curated set of repositories, they provide a base OS , follow best practices for `Dockerfile` and contain the latest security updates and patches amongst other things.

If for some reason you are unable to find a relevant Official image, you could consider starting out with ["Certified Images"][certified-images]

I would highly recommend reading the `Dockerfile` of the image you choose, it may be hard to understand when you first get started but it's an essential skill to acquire.

## [Alpine Linux Image](#alpine-linux-image)
[Alpine Linux Images][alpine-image] are minimalist Docker images that are only 5 MB in size. In a production environment you need code to be deployed quickly. Here is where Alpine Linux based images come in handy, as they contain only the most essential Linux packages. A note of caution however- since Alpine build are based on musl libc and BusyBox, if you have any dependency that only works with traditional GNU/Linux distributions, Alpine containers may not work for you. This is a very rare situation.

## [Building our Dockerfile](#building-our-dockerfile)
To run our simple Java application, we will choose an Apline based openjdk image. I found this image by searching for "OpenJdk" on Docker Hub and selected the [OpenJdk Official Image][openjdk-official] that showed up.

``` 
FROM openjdk:8-alpine
COPY target/hellodocker-1.0-SNAPSHOT-shaded.jar /usr/local/app/
WORKDIR /usr/local/app/
ENTRYPOINT ["java", "-jar", "hellodocker-1.0-SNAPSHOT-shaded.jar"]
```
Let's understand each step in this `Dockerfile`
 - Select the OpenJDK 8 Alpine image.
 - Copy our jar file which is generated on our local environment after running the `mvn clean package` command into a directory within our  image container. Command format `COPY <src> <dest>`.
 - Set that directory as the current directory, sort of like `cd /usr/local/app`
 - Execute the jar file.

These are all the steps we need to create a Docker image what will run our Java application.

## [Building our Docker image](#building-our-docker-image)
We'll build our Docker image using the command `docker build -t <image-name> PATH`. We'll give it a name so that it's easy to refer to it later, using the `-t` flag and following the naming convention `username/imagename:tag`. The `tag` is optional and we will not use it at this point.

`docker build -t oliver/simplejavaapp .`

Once the image is built you can run `docker image ls` to list the images on your system. Here is the output from my system
```
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
oliver/simplejavaapp   latest              2f0abbdfc557        53 minutes ago      105MB
<none>                 <none>              99007a07897e        55 minutes ago      105MB
openjdk                8-alpine            a3562aa0b991        9 months ago        105MB
```


## [Running the Docker Container](#running-the-docker-container)
Now let's run our container using the `docker run <image-name>` command. It should output "Hello Docker!!"

```
docker run oliver/simplejavaapp 

Output :> Hello Docker!!
```

## [Running a Spring Boot REST Application with Docker](#running-a-spring-boot-rest-application-with-docker)
Now that we covered the basics, it's time for something a bit more real world. I've forked a sample [Spring Boot REST application][spring-boot-docker-git]. This application is packaged as an executable jar and runs with an embedded server which listens on port `8080` by default.
As with the previous example, we'll create a Docker image and execute the application. The one extra step will be exposing the port the application listens on to the outside world. Here is the `Dockerfile`.

```
FROM openjdk:8-alpine
COPY target/rest-service-0.0.1-SNAPSHOT.jar /usr/local/app/
WORKDIR /usr/local/app/
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "rest-service-0.0.1-SNAPSHOT.jar"]
```
Let's understand each step in this `Dockerfile`
 - Select the OpenJDK 8 Alpine image.
- Copy our jar file which is generated on our local environment after running the `mvn clean package` command into a directory within our  image container. Command format `COPY <src> <dest>`.
 - Set that directory as the current directory, sort of like `cd /usr/local/app`
 - __Expose the port 8080 which is the default port for Spring Boot Web Applications.__
 - Execute the jar file.

The `EXPOSE` instruction is important since we need to inform Docker on which port the container is listening on at runtime.

The build command remains the same.

`docker build -t oliver/springrest .`

When running the container we specify an additional parameter `-p` to publish the port from the host in which your container is running, in order for the outside world to communicate with the container via the host. Here to keep it simple we map the same port 8080.

`docker run -p 8080:8080 oliver/springrest`

Now you will be able to `curl localhost:8080/greeting` and get a response.


## References
1. [Docker development best practices][best-practices]
2. [Docker Certified Images][certified-images]
3. [OpenJDK Official Docker Image][openjdk-official]
4. [How to build Docker images with Dockerfile][building-docker-images]
5. [Source code for the simple Java application][java-docker-git]
6. [Source code for the RESTful Spring Boot Application][spring-boot-docker-git]

 

[openjdk-official]: https://hub.docker.com/_/openjdk
[alpine-image]: https://hub.docker.com/_/alpine/
[java-docker-git]: https://github.com/oliversavio/java-docker-part-1
[spring-boot-docker-git]: https://github.com/oliversavio/gs-rest-service/tree/master/complete
[official-images]: https://docs.docker.com/docker-hub/official_images/
[certified-images]: https://docs.docker.com/docker-hub/publish/certify-images/
[building-docker-images]: https://linuxize.com/post/how-to-build-docker-images-with-dockerfile/
[best-practices]: https://docs.docker.com/develop/dev-best-practices/

