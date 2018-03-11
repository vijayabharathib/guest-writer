---
layout: post
title: "Ruby on Rails - Killer Workflow with Docker (Part 1)"
description: "Using docker reduces time spent on installation and configuration. You just dive right into building products."
date: "2018-02-27 08:30"
category: Ruby On Rails, Docker, Auth0, Travis CI, CI/CD
author:
  name: "Vijayabharathi Balasubramanian"
  url: "vijayabharathib"
  mail: "yajiv.vijay@gmail.com"
  avatar: "https://twitter.com/vijayabharathib/profile_image?size=original"
related:
- 2017-11-15-an-example-of-all-possible-elements
---

**TL;DR:** If you are looking to bootstrap a full-scale database driven application using Ruby on Rails with external identify management, look no further. This article will make it easy to convert ideas into testable MVPs ([Minimum Viable Products](https://en.wikipedia.org/wiki/Minimum_viable_product)). You will be able to build a small shelf and fill them with books in the process. You can have a look at the app [here](https://bookstall.herokuapp.com/) and  this [repository on GitHub](https://github.com/vijayabharathib/auth0_rails_docker) has the code and configuration of the finished product.

## Development Workflow

The objective is to optimize for developer happiness. Ruby on Rails fits the bill. You'll setup a workflow that makes it easy to configure, test, debug and deploy.  Also, you will enter the world of [continuous deployment](https://en.wikipedia.org/wiki/Continuous_delivery) where commits are deployed automatically on successful test runs.

### Tech Stack

**1. Docker**

You need to get [docker installed](https://docs.docker.com/install/). Use `docker -v` to verify you have a working version of docker. The one I have is `Docker version 17.12.0-ce, build c97c6d6`. You'll know more about docker in a minute (or in an hour if you have to go through all the setup here).

**2. Docker Compose**

Next one is to install [Docker compose](https://docs.docker.com/compose/install/). It will allow you to piece multiple services together. Try `docker-compose -v` to ensure a working version. Here is the result from my terminal: `docker-compose version 1.16.1, build 6d1ac21`.

**3. Free Tier Login Accounts**

  * [GitHub](https://github.com/)
  * [Travis](https://travis-ci.org/)
  * [Heroku](https://dashboard.heroku.com)
  * [Auth0](http://auth0.com/) 

None of them should cost you a dime for following this workflow!

**4. Git CLI** (Seriously?)

If you have not been using [Git CLI](https://git-scm.com/downloads), it's time already. Start [here](https://git-scm.com/book/en/v2/Getting-Started-The-Command-Line).

**5. What about Rails?**

That's the point. You *don't* need to install Ruby, Rails or Build Tools in your local environment. You'll instruct docker to handle that.

## Docker - Up and Running

Just like your git repository is a single version of truth that you share with your team, a docker image is a repository of your runtime you can share with your team. 

Here is the problem. See if you recognize it. You started working on a new project that needed a certain environmental setup or wanted to try out a new technology. You followed the guide to the letter and ran into issues launching the local application. No one else seems to have got the issue. **It works on their machine, but not on yours**. You consulted *Stack Overflow* trying to figure out what went wrong. Apply sequence of helpful answers to narrow down on the issue and finally reach the moment of truth.

You don't recognize that? Lucky you. For the rest of us, Docker solves the problem by allowing us to bundle our runtime environment in an image and share it around, just like a Git repository.

Sometimes, it just boils down to having a simple text file named `Dockerfile` with a set of commands to get the right environment.

The environment mentioned above is everything required to run a particular software. Docker images are solved problems for you to build your solution on top of.

Let's say you need Ruby runtime on Ubuntu. Pull an Ubuntu image and install Ruby on top. Wait, isn't that what we do already? Yes. So, here is one better. How about pulling an image that already has Ruby on top of Ubuntu. Now we are talking? How about pulling an image that has Ruby and NodeJS on top of a Debian flavor? Sweet.

The bottom-line, pull an image and get down to business. Your business could be, finally, building that app. Tweaking configurations may not be a productive way to spend a whole evening, thought that's exactly what you might end up doing in certain cases.

### Docker Vs Virtual OS

Now, think about other solutions that solve similar problem. You might have used a *dual-boot* PC that runs both Windows and Linux. You would have switched between them when required.

Another solution is the VirtualBox that allows you to run entirely full scale guest OS on top of your host OS.

These are not bad solutions. It is just that Docker is a leaner solution compared to these full scale OS images if you just want to have a uniform runtime environment. Moreover, there is no automated way to configure each OS the same way, unless you do it manually, which may lead to different configurations at scale.

Compare that against a set of written instructions that can be version controlled. **Docker uses the instructions to pull in the necessary OS libraries to construct the runtime on top of your existing operating system.** Not too different from using a `package.json` or `Gemfile` to keep track of project dependencies.

The image with Ruby and Node that we'll be using is close to 300MB. Imagine having to set up a virtual OS, usually takes up more space.

Enough selling Docker. Let's get one up and running.

### Finding The Right Docker Image

The right runtime environment for you may have been solved already. You just need to find it. If you need only ruby, the [official ruby image](https://store.docker.com/images/ruby) will do. If you need both Ruby and NodeJS, I found this [image by starefossen](https://hub.docker.com/r/starefossen/ruby-node/) useful.

This image uses official ruby image and builds NodeJS on top of it using commands from the official NodeJS image. So you get best of both worlds.

**Just a word of caution**, Docker is not a silver bullet. Our objective is to start a runtime as quickly as possible and move on to building our app. But if the repository is not maintained or updated to reflect latest ways to install NodeJS, then it might be broken. Look through the builds to ensure you have a working image.

### Giving Life to A Container
Hurray! Time to step into a terminal.

```bash
mkdir auth0_rails_app
cd auth0_rails_app
touch Dockerfile
```

In English, create a folder named `auth0_rails_app`, `cd` into the directory and create a file named `Dockerfile`.

Now, on to filling in some contents within the `Dockerfile`.

```Dockerfile
FROM ruby:2-stretch

```
**Tip**: Look into the [docker store](https://store.docker.com/images/ruby) and pick the right version. You can even narrow down to `2.5.0-stretch` tag, that gives you more control over upgrades.

...and back at the terminal. Time to build and run the container.

```bash
docker build -t auth0app .
docker run -it auth0app

```

The first command builds an image based on the instructions you've given in the `Dockerfile`. For now, `Dockerfile` just says, pull up the image tagged `2-stretch` from official Ruby repository. It would take minutes (or hours) depending on your connection. Total download size was close to 300MB.

Second command boots the image into a container, which is like giving life to the image. Can't help it, but it is identical to creating an object out of a class! `-it` flag gives you a terminal access *inside* the container. If that leaves you with `irb>` prompt, the interactive shell for Ruby, then you are all set. 

If you need to verify Ruby version, run following commands:

```bash
docker run -it auth0app /bin/bash
```

That should take you to the shell prompt that shows something in the lines of `root@abc123a1234b:/# `. That means you are inside the container, **with root access**. Just be careful what you command the prompt to do!

Now run `ruby -v` inside the shell to get the ruby version. Did you get `ruby 2.5.0...`? 

Well done so far. 

