+++
title = "Remember to Pull Your Base Docker Images Before Building"
slug = "remember-to-pull-your-base-docker-images-before-building"
aliases = ['/articles/remember-to-pull-your-base-docker-images-before-building']
date = 2016-04-22
draft = false
tags = ["docker", "jenkins"]
+++

{{< notice warning >}}
This post was written in 2016 and the ecosystem has matured much since then.

Proceed with caution!
{{< /notice >}}

At work I was recently debugging some docker containers that were being built nightly by Jenkins. These were actually the same images that I was building [in my last article]( {{< ref "posts/2016/02-building_docker_multiple_base_image_tags.md" >}}).

After making the changes that I mentioned in that article, I noticed that the resulting containers for ``argon`` were having no issues, but builds of our app using ``latest`` were having issues finding a package. So I built our app container locally and I dived in to see what was wrong.

```console
$ docker run --rm -i -t --entrypoint="/bin/bash" container-name
```

This let me get a prompt inside the container to take a look at what the problem was. First I looked inside the ``node_modules`` folder to see if the package was messed up. What I instead noticed was that the container was missing a bunch of our dependencies! Then I got suspicious...

```console
$ node --version
4.1.2
```

What the fuck. This should be `5.10.1`

So I re-triggered the Jenkins job, blew everything away locally and re-pulled the ``baseimage:latest`` that Jenkins just built.

```console
$ docker run --rm -i -t --entrypoint="/bin/bash" baseimage:latest

# node --version
4.1.2
```

FUCK

So I check the Jenkins build log. Yes, it correctly spits out `FROM node:latest` when building the image. It pushes everything correctly. No errors, no typos.

So I explicitly compare what I was running before in the Dockerfile to what I was running now. Before it was actually this:

```Dockerfile
FROM node:5
```

And now its:

```Dockerfile
FROM node:latest
```

Currently, those point to the same image on Docker hub. It was at this point though that I realised what was going on. At some point months ago someone on this Jenkins box did a `docker pull node`. Then at build time, when docker read the `FROM node:latest` line, it would check locally, see the image, and just use that.

This wasn't a problem before because `FROM node:5` would look locally, but not see anything tagged `node:5`. It would then pull it from Docker Hub, but not save the tag on the server! Thus, every time it ran it would re-pull the latest 5 release container.

So the solution? Be explicit in what your project needs in your automated builds. Pull your base images before attempting to use them.

```bash
#!/bin/bash
TAGS="latest:argon"

for tag in ${TAGS//:/ }; do
    sed 's/TAG/'"${tag}"'/' Dockerfile-template > Dockerfile
    docker pull node:${tag}
    docker build --no-cache -t container-name:${tag} .
    docker push container-name:${tag}
    docker rmi container-name:${tag}
done
```
