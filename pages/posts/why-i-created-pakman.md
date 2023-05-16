---
title: Why I created the PAKman Build System
date: 2023/5/16
description: Why I created PAKman a build system based on Alpine packages using Elixir programming language.
tag: development, infrastructure
author: Zack Siri
---

# Why I created the PAKman Build System

PAKman is one of the 4 core modules that power [instellar.app](https://instellar.app). It's [open-sourced](https://github.com/upmaru/pakman) and builds your application using github actions into alpine packages that get delivered to an S3 compatible bucket you specify via instellar. Our platform then takes that built package and deploys the application on your infrastructure.

![basic workflow](/images/why-i-created-pakman/workflow-diagram.png)

## In the beginning

Back in 2018 I looked at using docker before I embarked on the journey to build my own build system. At that point I had been using docker for a long time. I was an early user of docker and one of the issues I constantly ran into was the following:

- Large build artifact (hundreds of MB)
- Needed a Registry
- Consumes bandwidth
- Slow deployments

At first I considered just using docker because it was the 'standard'. Everyone was using docker and docker swarm was in it's hey days, k8s was gaining steam. Most of the docker images were built using ubuntu as the base image, as you can imagine the built images were quite large. Alpine linux was gaining popularity and was starting to be used in the docker community to reduce image size. I often wondered, why the community didn't just build using alpine's native build system. So I tried it for myself. It took me a long time to work through the alpine build system, the documentation was scarce and I had to trial and error my way to understanding it. My little experiment made me realize that while the final output was amazing (built packages were ranging from few MB - 50MB depending on the application) it was extremely complex to use. I figured most people probably just ended up using docker due to simplicity and readily available documentation.

I ended up mastering building with alpine's package system and threw together some scripts that would automate and make things easy to build with alpine packages. There was however one problem, this meant not using docker for running the applications. With docker you build the app into a docker image and you run the entire image. You wouldn't just install the custom package in a docker container because that means the image would need a package manager and that would just make the final image even larger. This is where the concept of docker being an 'application' container hit hard.

I also explored kubernetes to see what it could do and figured that kubernetes was way too complex for most deployments. The conclusion I came to was k8s and docker would work together. If I wanted to use my alpine package build method I would need something else.

## Enter LXD

While doing my research I found LXD, it advertised itself as being a 'system' container this meant creating an LXC container would mean I had the entire OS running including the package manager. This was exactly what I was looking for and would fit with my build system like peas in a pod. LXD containers meant that all I had to do was expose the alpine package in a file system and add it as the repository inside the alpine linux container and I could run `apk update && apk add [package]` and be done with it. I hacked together a proof of concept with bash and terraform and amazingly it worked! I was actually able to just build my app and just ship my app to my lxc container and it was blazingly fast! Apps were being deployed in a matter of a few seconds! Upgrades were also handled by alpine packages by adding the `-u` flag. Upgrades were even faster than installing a fresh package.

![docker vs pakman](/images/why-i-created-pakman/docker-vs-pakman.png)

## A new Invention is needed!

While my proof of concept worked it was far from ready for primetime. I needed something which is robust, written in a language I'm familiar with (elixir), and most importantly worked with an existing infrastructure I didn't have to host. The first versions of PAKman I hacked together was a combination of building packer images using bash script that would run in a custom gitlab runner. While it worked, it was not elegant and was not flexible. In 2018 Github Action was released I explored github actions and realized that I could create my own custom action inside a docker container which meant I could use whatever programming language I wanted to create the build system.

I realized that I needed to create a simple solution for people and simply telling everyone to simply 'just use alpine's build system' would not work. I had an idea that I could essentially simplify everything down to a `.yml` file. I needed to develop an intermediary layer that would take the yaml file and convert it into files that the apkbuild system for alpine linux would understand. This is the birth of PAKman

![what pakman does](/images/why-i-created-pakman/what-pakman-does.png)

## Project Goal

While I still needed to use a docker container to create the final build since that's how github actions work. I realized that I can simply extract the artifact and ship it to an S3 compatible storage. This was the most simple design. Since once the package was built I could install and run it anywhere Alpine Linux ran. This would achieve the following goals:

- No need for custom infrastructure for building
- Packages need to be as small as possible
- Save on bandwidth costs
- Fast deployments (matter of seconds)

![ship small packages](/images/why-i-created-pakman/ship-small-package.png)

While many may challenge my decisions of saving bandwidth. I do have my reasons. I believe if something can be done well it should be done. In the big picture the goals of PAKman serves our mission for [instellar.app](https://instellar.app). Instellar enables anyone to run their own PaaS on their own infrastructure. This means it's important for us to keep the cost of ownership low. If we can save on bandwidth costs for our customers it's our duty to do it. Another valuable asset we save is time. Small packages mean deployments are fast! The update for the blog you are reading now was deployed in 6 seconds! You can see [PAKman in action](https://github.com/upmaru-stage/nimbus/actions).

![fast deployments](/images/why-i-created-pakman/upgrade-timestamp.png)

The final built artifact that gets shipped over the wire for this NextJS blog weighs in at 5.69 MB

![built artifact](/images/why-i-created-pakman/built-artifact.png)

Welcome to the future!

