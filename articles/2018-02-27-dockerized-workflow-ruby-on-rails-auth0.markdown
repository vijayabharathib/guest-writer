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

  * [GitHub]
  * [Travis]
  * [Heroku]
  * [Auth0]

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
FROM starefossen/ruby-node:2-8-stretch
```

**Tip**: Look into the [docker hub](https://hub.docker.com/r/starefossen/ruby-node/) and pick the right version from the list of tags that suits your needs.

...and back at the terminal. Time to build and run the container.

```bash
docker build -t auth0app .
docker run -it auth0app /bin/bash

```

The first command builds an image based on the instructions you've given in the `Dockerfile`. For now, `Dockerfile` just says, pull up the image from the repository. It would take minutes (or hours) depending on your connection. Total download size was close to 300MB for the first time.

Second command boots the image into a container, which is like giving life to the image. Can't help it, but it is identical to creating an object out of a class! `-it` flag gives you an interactive terminal access *inside* the container. 

That should take you to the shell prompt that shows something in the lines of `root@abc123a1234b:/# `. That means you are inside the container, **with root access**. Just be careful what you command the prompt to do!

Now run `ruby -v` inside the shell to get the ruby version. Did you get `ruby 2.5.0...`? Great!

Well done so far. 

## Ruby on Rails Inside the Container

This is where the Rails project takes shape. **This whole section is just one-off execution.** Once you are done with this, all else is normal application development workflow.

### Setting Up `Dockerfile` 
Here are the additions to `Dockerfile`

```Dockerfile
FROM starefossen/ruby-node:2-8-stretch
RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev && \
    gem install bundler
RUN mkdir /app
COPY Gemfile Gemfile.lock /app/
WORKDIR /app
RUN bundle install
COPY . /app
```

**Tip**: Each `RUN` command creates a layer. It is usually a best practice to put most of those commands together in one line to avoid layers within the image. For example, `RUN apt-get update -qq & apt-get install -y build-essential libpq-dev` and so on. 

Here is what's going on:

* Pull the ruby+node image from Docker hub.
* Update the libraries within the image
* Install build tools and library for Postgres.
* Install Bundler which will update existing Bundler.
* Create a folder for app
* Copy `Gemfile` and `Gemfile.lock` to app folder
* Set working directory to app folder
* Run `bundle install` inside the app folder
* Copy rest of the content from your host folder to container app folder.

Note that copying only `Gemfile` and `Gemfile.lock` to app folder before running `bundle install` has tremendous advantage. Changes to the other files within the application folder do not trigger `bundle install`. 

Only if the `Gemfile` or `Gemfile.lock` changes, `bundle install` will be triggered. 

If you think deeply enough, you'll understand that it will save hours. If thinking deeply hurts, just try moving the command `COPY . /app` line just above `bundle install`. Every time you make even a small change within app folder, the container has to be built with all the gems being installed from scratch. This hurts more!

### Stitch Services with `docker-compose.yml`

The `docker-compose.yml` file that stitches multiple pieces together such as database container, application container, host folder where you store your application repository, environmental aspects such as volumes and ports.

This is going to be a **database driven application**. So, we need a way to persist data created in the environment. One way is to introduce a separate service for database layer that has its own volume (fancy name for storage space).

It looks like the one below:

```yml
version: '3'
volumes: 
  postgres-data:
services:
  db:
    image: postgres
    volumes: 
      - postgres-data:/var/lib/postgresql/data
  app:
    build:
      context: .
      dockerfile: Dockerfile
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
```

All right, another set of descriptions for instructions within the `docker-compose.yml` file:

1. The version tells Docker about the possible tokens used inside this YAML file. Be cautious using flags from other versions, they may not work in version 3.
2. A standalone database container launched from official Postgres image.
3. Another service named `app` which is built based on the `Dockerfile` in the same folder.
4. The final command that need to be run when launching the container.
5. A volume that maps local folder to corresponding folder inside the container.
6. Instruction to open port 3000 and map it to port 3000 on host.
7. An instruction to launch `db` service first as a dependency before launching `app` service.

### All New Rails Project

Start by building the containers. For that you need to add an empty `Gemfile` and another empty `Gemfile.lock`. **Once those empty files are in place**, run this on your terminal:

```
docker-compose up --build
```

You will see that `step 1/nn: FROM starefossen/ruby-node:2-8-stretch` is immediately available as a layer (that is if you have already tried the initial `docker run` with single line of `Dockerfile`).

That's Docker reusing the ruby image downloaded earlier. The whole process that ran last time is saved for you. So it is only from second line of code within the `Dockerfile` that matters now.

You'll see all the steps within the `Dockerfile` executed dutifully during the build process. Each of them creates a **layer** that you can recognize by their hash, that looks like `1adb3ee3e245`. 

Once build steps are over, you'll see `db` service being started first. And then the `app` service starts, but fails when trying to run `bundle exec rails s`, as we do not have `rails` gem within the environment yet.

Get into another terminal and run the following command to **stop the services**:

```bash
docker-compose down
```

Add just two lines to the `Gemfile`. It should look like:

```Gemfile
source 'https://rubygems.org'
gem 'rails', '~>5.1.5'
```

Now if you build the services again using `docker-compose up --build`, you'll notice that Docker reuses most of the layers created. 

Docker build picks up from where changes happened. It does not pull the ruby image again, it does not create user or folders. The only change is on `Gemfile`, hence, the line `COPY` within the `Dockerfile` kicks in and build starts from there on top of existing layers.

The build process exits with a neat Rails `man` page showing how to initiate a new app. With the `rails` gem installed within the container, now you are in a position to initiate a new Rails project.

...back at the terminal:

```bash
docker-compose run app rails new . --force --database=postgresql
```

That should scaffold a new rails application in all its initial glory. As you can see, `postgresql` is the database of choice. 

All the files generated will be owned by `root`. Run `ls -la` within the terminal to look at who owns the files. Another way is to try and edit one of the generated files. If they are owned by a user other than yourself, you will have to go down the `sudo` route to get them `chown`ed. Run this command on your terminal within the application folder:

```bash
sudo chown -R weeuser:weeuser .
``` 

Apparently, use your id instead of `weeuser` in the command above.

Clean the `Gemfile` a bit. You might want to remove lines `jbuilder` and `coffee-rails` as you may not need them. You need to add `guard`, `guard-minitest` and `guard-livereload` to the `group :development, :test do` group within the `Gemfile`. 

```Gemfile
group :development, :test do
  # Leave buybug, capybara and selenium-webdriver
  # ...
  gem 'guard', '~>2.14.2'
  gem 'guard-livereload','~>2.5.2'
  gem 'guard-minitest', '~>2.4.6'
```

Stop the services with `docker-compose down` if they are running. To build them once again to take account of changes to the `Gemfile`, run this command:

```bash
docker-compose up --build
```

**Explore:** Docker starts to download and install all gems once again. There are ways to cache gems locally so that they do not need to be downloaded again. Some of the online solutions are a bit old and broken. You'll have to let me know when you find a solution. 

But the good news is, **Yay! You are on rails!**. Open your favorite browser and point it at `localhost:3000` and you should see the whole world rejoicing at the sight of `Rails 5.1.5` and `Ruby 2.5`.

## Git It Up

A working configuration of a brand new Rails project is usually very easy. As you saw, it just took a simple `rails new` command within the container. But it has done so much work for you, so much that it warrants a git commit.

```bash
git init
git add --all
git commit -am "first working copy"
```

That initiates git repository within the current folder. Adds all the files to staging area and commits the changes with a readable comment.

If you look closely, Rails generated a `.gitignore` file for you with pre-populated instructions as to which folders to ignore. This file ensures unnecessary folders and files, such as log files, are kept out of the repository.

Now create a new repository within [GitHub]. Just create an empty one, no need to create any `README.md` or `.gitignore` files on the GitHub server. They are already there on your local machine.

GitHub automatically shows these instructions. Run these on your terminal to push your changes to GitHub. 

```
git remote add origin git@github.com:user_name/repo_name.git
git push -u origin master
```

Once `push` is complete, refresh GitHub repository page to see the files committed. Now that they are on the cloud, we can say it is much more safe to play around with the local copy.

**From here onwards, we'll work on a branch named `staging` and move changes to `master` branch only when necessary.**

For that, follow these commands:

```bash
git checkout -b staging
git push -u origin staging
```
That creates a new branch and checks it out. Pushes the branch to `origin`, which is another name for the GitHub repository on the server side.

## Cloud Authentication by Auth0

It is better to have Authentication from the start. It will force you to think about modeling records accordingly. This is where you can use the free-tier login given by [Auth0] to try it out.

### Auth0 Tenant and Client

If you have not done so already, sign up to [Auth0] and follow the instructions:

1. Sign up / Login to Auth0 
2. Create a domain/tenant (tugboat)
3. Create a client (secondstall)

While creating the client, select *Regular Web Application* as type. Once the client is created, within the list of technologies shown, select **Ruby On Rails**. [Auth0] shows the [Quick Start Guide](https://auth0.com/docs/quickstart/webapp/rails/01-login) for rails by default. 

A cool feature within the guide is, it shows your client name and other details if you are logged in, unlike the generic guide linked above. You can reuse the code from the guide in most of the instructions used in this post, as the guide will show you code customized for your app and name you have chosen. 

It also has instructions for some of the most commonly faced issues. It will come in handy if you run into any such issues, but for this tutorial, you may not run into any of those. The focus, for now, is on getting the authentication working and moving on to building the app.

[Auth0] reaches our application through callback URLs. So it is **important** to tell Auth0 client what the callback URL will be. Under settings section, find a text box that asks for **Allowed Callback URLs** and add this:

```bash
http://localhost:3000/auth/oauth2/callback
```

### Updates at Rails End

4. Add env variables to docker
5. set callback URLs
6. install gem dependencies (dc up --build)
7. setup initializer middleware (auth0)
8. setup controllers/pages/URLs to handle login/logout
9. show how to persist user login


[GitHub]: https://github.com/
[Travis]: https://travis-ci.org/
[Heroku]: https://dashboard.heroku.com
[Auth0]: http://auth0.com/ 
