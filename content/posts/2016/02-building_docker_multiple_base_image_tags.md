+++
title = "Automatically Building Docker Containers With Different Base Image Tags"
slug = "automatically-building-docker-containers-with-different-base-image-tags"
aliases = ['/articles/automatically-building-docker-containers-with-different-base-image-tags']
date = 2016-04-20
lastmod = 2024-09-07
draft = false
tags = ["docker", "jenkins", "bash"]
+++

{{< notice warning >}}
This post was written in 2016 and the ecosystem has matured much since then.

Proceed with caution!
{{< /notice >}}

At work I recently added a Dockerfile to one of our accessory tools, and set it up with Jenkins to build on a nightly basis. This build occurs nightly as opposed to on new master commits to ensure that the resulting container always has the latest versions of all its dependencies (including node).

I wrote the Dockerfile such that we were building off of the [official node containers](https://hub.docker.com/_/node). Specifically, off of the latest tag. Here is what I mean:

```Dockerfile
FROM node:latest

RUN some-stuff
```

This would ensure that I was always using the latest node version every night, which is exactly what I wanted. Perfect!

A few weeks later as news of the container made its way around the devs, I was asked to provide a build of the tool that uses the latest LTS node release instead of the latest-latest release. I should be able to do that, no problem. On Docker Hub there is a node tag named ``argon``, which corresponds to the latest build of the current LTS branch (currently node 4.4.3). So my challenge was to provide two identical containers with only the node version changed between them.

I first considered using the Docker ``--build-arg`` flag with a corresponding ``ARG`` line in the Dockerfile. Typically, this is used in the following way:

```Dockerfile
FROM ubuntu

ARG myvar

RUN apt-get install ${myvar}
```

And then built like so:

```shell
docker build --build-arg myvar=value -t container-name .
```

But of course, I wanted to use this build argument in the ``FROM`` line. So first I tried this:

```Dockerfile
ARG tag

FROM node:${tag}

RUN some-stuff
```


But that isn't going to work. The first line in every Dockerfile must be a ``FROM`` statement.

What I instead decided to do (and settled on) was to instead use a file I named ``Dockerfile-template``, and then use the unix tool ``sed`` to generate the actual Dockerfile one at a time. Here is what that looked like:

```Dockerfile
FROM node:TAG

RUN some-stuff
```

And the scrubbed bash script that was run by Jenkins:

```bash
#!/bin/bash
TAGS="latest:argon"

for tag in ${tags//:/ }; do
    sed 's/TAG/'"${tag}"'/' Dockerfile-template > Dockerfile
    docker build --no-cache -t container-name:${tag} .
    docker push container-name:${tag}
    docker rmi container-name:${tag}
done
```



So now I'm providing nightly rebuilds of the internal tool, one with tag ``latest`` and one with tag ``argon``, each corresponding to what version of node it was built with!
