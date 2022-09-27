---
title: How to setup Nextcloud AIO docker image with traefik the correct way
date: 2022-09-26
#categories: ["flutter tricks"]
tags: [nextcloud, traefik, docker]     # TAG names should always be lowercase
---
As a specialist in distributed services, I can say that... jk, I'm an app developer and the title was clickbait! Well, not entirely. I actually wanted to setup Nextcloud on my home NAS and I wanted to keep it behind my reverse proxy solution Traefik, because it sets up a Let's Encrypt certificate for my automatically and everything. The simplest solution seemed to be the Nexcloud AIO docker image.

## Nextcloud AIO
AIO stands for "All In One" and is a clever solution with one (1) docker image which spins up all needed services in other docker containers once started and configured. You give it access to `/var/run/docker.sock`{: .filepath} by mounting it as a volume and by some clever docker-ception it starts containers for services, databases and frontends when it's configured. That's just it though, when it's confirgured. I had quite a bit of problem with that small gotcha.

## Traefik
Traefik is a great reverse proxy that can make SSL termination for all of your services a simple endeavor. You can run it in a docker container and also give it access to `/var/run/docker.sock`{: .filepath} and then it will react to new containers and create routers for them automatically depending on config that you can put in labels of the new containers. If you want to know more and configure it for your self, I recommend [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04)

## The Problem
The root of the problem is that the helpful, albeit incomplete, [documentation](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#traefik-2) kind of assumes that you are running the docker container with `--network host` mode. I do not, however, and don't want to run my traefik instance in host mode. "Just replace localhost with the IP" says the instructions so I did. 

When the AIO container spins up it starts a web UI where you can configure some things and also test your port forwarding (and reverse proxy) and... it didn't work. I spent so long time trying to debug the proxy and forwarding and communication to and from containers