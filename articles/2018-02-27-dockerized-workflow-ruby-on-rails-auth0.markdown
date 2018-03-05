---
layout: post
title: "Killer Dockerized Workflow for Ruby on Rails With Auth0 - Part 1"
description: "Skip the entire installation and configuration business. Dive right into building products."
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

**TL;DR:** If you are looking to bootstrap a full-scale database driven application using Ruby on Rails with password-less authentication, look no further. This article will make it easy to convert ideas into testable [MVPs](https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMinimum_viable_product) (minimum viable products). We are going to build a small shelf. We'll fill them with books in the process. You can have a look at the app [here](https://bookstall.herokuapp.com/) and [repository on GitHub](https://github.com/vijayabharathib/auth0_rails_docker) has the code and configuration to start with.

## Development Workflow

The objective is to optimize for developer happiness. Ruby on Rails fits the bill. We'll figure out a workflow that is easy to setup, test, debug and deploy.  We will enter the world of [continuous deployment](https://en.wikipedia.org/wiki/Continuous_delivery) where commits are deployed automatically on successful test run.

### Tech Stack

**1. Docker**

Get [docker installed](https://docs.docker.com/install/). Use `docker -v` and verify you have a working version of docker. The one I have is `Docker version 17.12.0-ce, build c97c6d6`. We'll talk more about docker in a minute (or in an hour if you have to go through all the setup here).

**2. Docker Compose**

[Docker compose](https://docs.docker.com/compose/install/) will allow us to piece multiple services together. Try `docker-compose -v` to ensure a working version. Here is the result from my terminal `docker-compose version 1.16.1, build 6d1ac21`

**3. Free Tier Login Accounts**

  * [GitHub](https://github.com/)
  * [Travis](https://travis-ci.org/)
  * [Heroku](https://dashboard.heroku.com)
  * [Auth0](http://auth0.com/) 

None of them should cost you a dime for following this workflow!

**4. Git CLI** (Seriously?)

Let's not leave anything to chance. If you have not been using [Git CLI](https://git-scm.com/downloads), it's time already. Start [here](https://git-scm.com/book/en/v2/Getting-Started-The-Command-Line).

**5. What about Rails?**

That's the point. You *don't* need to install Ruby, Rails or Build Tools in your local environment. We'll instruct docker to handle that for us.

## Docker - Up and Running
TL;DR: Just like your git repository is a single version of truth that you share with your team, a docker image is a repository of your runtime you can share with your team. 

Here is the problem. See if you recognize it. You started work on a new project that needed a certain environmental setup or wanted to try out a new technology. You followed the guide to the letter and ran into issues launching the local application. No one else seems to have got the issue. **It works on their machine, but not on yours**. You consulted *stackoverflow* trying to figure out what went wrong. Apply sequence of helpful answers to narrow down on the issue and finally reach the moment of truth.

You don't recognize that? Lucky you. For the rest of us, `docker` solves the problem by allowing us to bundle our runtime environment in an image and share it around, just like a `git` repository.

Sometimes, it just boils down to having a simple text file named `Dockerfile` with a set of commands to get the right environment.

When I say environment, it is everything required to run a particular software. Docker images are solved problems for you to build your solution on top of.

Let's say you need `ruby` runtime on `ubuntu`. Pull an `ubuntu` image and install `ruby` on top. Wait, isn't that what we do already? Yes. So, here is one better. How about pulling an image that already has `ruby` on top of `ubuntu`. Now we are talking? How about pulling an image that has `ruby` and `nodejs` on top of a `debian` flavor? Sweet.

The bottom-line, pull an image and get down to business. Your business could be, finally, building that app. Tweaking configurations may not be a productive way to spend a whole evening, thought that's exactly what we might end up doing in certain cases.

### Docker Vs Virtual OS

Now, think about other solutions that solve similar problem. I know I've used a `dual-boot` PC that runs both Windows and Linux. I switch between them when required, but wouldn't be able to run both at the same time.

Another solution is the virtual box that allows you to run entirely full scale guest OS on top of your host OS.

These are not bad solutions. It is just that `docker` is a leaner solution compared to these full scale OS if you just want to have a uniform runtime environment.

**Docker just pulls in the necessary libraries to construct the runtime using your existing operating system.**

The image with `ruby` and `node` that we'll be using is close to 300MB. Imagine having to set up a virtual OS, usually takes up more space.

Enough selling `docker`. Let's get one up and running.

### Finding The Right Docker Image

Like I said, the right runtime environment for you may have been solved already. You just need to find it. If you need only ruby, the [official ruby image](https://store.docker.com/images/ruby) will do. If you need both `ruby` and `node`, I found this [image by starefossen](https://hub.docker.com/r/starefossen/ruby-node/) useful.
This image uses official ruby image and builds `node` on top of it using commands from the official `node` image. So you get best of both worlds.

**Just a word of caution**, `docker` is not a silver bullet. Our objective is to start a runtime as quickly as possible and move on to building our app. But if the repository is not maintained or updated to reflect latest ways to install `node`, then it migth be broken. Look through the builds to ensure you have a working image.

### Giving Life to A Container
Hurray! Time to step into a terminal.

```bash
mkdir auth0_rails_app
cd auth0app
touch Dockerfile
```

In English, create a folder named `auth0_rails_app`, `cd` into the directory and create a file named `Dockerfile`.

Now, on to filling in some contents within the `Dockerfile`.

```Dockerfile
FROM ruby:2-stretch

```
...and back at the terminal. Let's build and run the container.

```bash
docker build -t auth0app .
docker run -it auth0app

```

The first command builds an image based on the instructions you've given in the `Dockerfile`. For now, `Dockerfile` just says, pull up the image tagged `2-stretch` from official `ruby` repository. It would take minutes (or hours) depending on your connection. Total download size was close to 300MB.

Second command boots the image into a container, which is like giving life to the image. Can't help it, but it is identical to creating an object out of a class! `-it` flag gives you a terminal access. If that leaves you with `irb>` prompt, the interactive shell for `ruby`, then you are all set. Well done so far. 

