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
- 2018-02-27-dockerized-workflow-ruby-on-rails-auth0-part2
---

**TL;DR:** Workflow to bootstrap a full-stack database driven application using Ruby on Rails with external identify management. This article will make it easy to convert ideas into testable MVPs ([Minimum Viable Products](https://en.wikipedia.org/wiki/Minimum_viable_product)). 

You will be able to build a small shelf and fill them with books in the process. You can have a look at the app [here](https://bookstall.herokuapp.com/) and  this [repository on GitHub](https://github.com/vijayabharathib/auth0_rails_docker) has the code and configuration of the finished product.

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

If you already know what docker is and how is it helping developers, skip over to the section **Finding The Right Docker Image**. If not, there is a first time for everything. Follow along.

Here is the problem. See if you recognize it. You started working on a new project that needed a certain environmental setup. You followed the guides to the letter and ran into issues launching the local application. No one else seems to have got the issue. **It works on their machine, but not on yours**. You consulted [Stack Overflow](https://stackoverflow.com/) trying to figure out what went wrong. Apply sequence of helpful answers to narrow down on the issue and finally reach the moment of truth.

You don't recognize that? Lucky you. For the rest of us, Docker solves the problem by allowing us to bundle our runtime environment in an image and share it around.

Just like your git repository is a single version of truth for your code that you share with your team, a docker image is a repository of your runtime environment that you can share with your team. 

Sometimes, it just boils down to having a simple text file named `Dockerfile` with a set of commands to get the right environment. Docker images are solved problems for you to build your solution on top of.

Let's say you need Ruby runtime on Ubuntu. Pull an Ubuntu image and install Ruby on top. Wait, isn't that what we do already? Yes. So, here is one better. How about pulling an image that already has Ruby on top of Ubuntu. Now we are talking? How about pulling an image that has Ruby and NodeJS on top of a Debian flavor? Sweet. From there onwards, the same image can be used across multiple projects as separate containers.

The bottom-line, pull an image and get down to business. Your business could be, finally, building that app. Tweaking configurations may not be a productive way to spend a whole evening, thought that's exactly what you might end up doing in certain cases.

### Docker Vs Virtual OS

Now, think about other solutions that solve similar problem. You might have used a *dual-boot* PC that runs both Windows and Linux. You would have switched between them when required.

Another solution is the VirtualBox that allows you to run entirely full scale guest OS on top of your host OS.

These are not bad solutions. It is just that Docker is a leaner solution compared to these full scale OS images if you just want to have a uniform runtime environment. Moreover, there is no automated way to configure each OS the same way, unless you do it manually, which may lead to different configurations at scale.

Compare that against a set of written instructions that can be version controlled. **Docker uses the instructions to pull in the necessary OS libraries to construct the runtime on top of your existing operating system.** Not too different from using a `package.json` or `Gemfile` to keep track of project dependencies.

The image with Ruby and Node that we'll be using is close to 300MB. Imagine having to set up a virtual OS, usually takes up more space.

Enough selling Docker. Time to get an image up and running.

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

In English, create a folder named `auth0_rails_app`, `cd` into the directory and create a file named `Dockerfile`. I can hear you say, "that doesn't need a terminal!". But, it's good to get comfortable living in the terminal.

Now, open your favorite text editor. Fill in this content within the `Dockerfile`.

```Dockerfile
FROM starefossen/ruby-node:2-8-stretch
```

That command says "pull the `ruby-node` image tagged `2-8-stretch`".

**Tip**: Look into the [docker hub](https://hub.docker.com/r/starefossen/ruby-node/) and pick the right version from the list of tags that suits your needs.

...and back at the terminal. Time to build and run the container.

```bash
docker build -t auth0app .
docker run -it auth0app /bin/bash

```

The first command builds an image based on the instructions you've given in the `Dockerfile`. For now, `Dockerfile` just says, pull up the image from the repository. It would take minutes (or hours) depending on your connection. Total download size was close to 300MB for the first time.

Second command boots the image into a container, which is like giving life to the image. Can't help it, but it is identical to creating an object out of a class! `-it` flag gives you an interactive terminal access *inside* the container. 

That should take you to the shell prompt that shows something in the lines of `root@abc123a1234b:/# `. That means you are inside the container, **with root access**. *Just be careful what you command the prompt to do!*

Now run `ruby -v` inside the shell to get the ruby version. Did you get `ruby 2.5.0...`? What about `node -v`? Did you get `v8.10`? 

Great! That's your own runtime environment with Ruby and Node pre-installed. Well done so far. 

## Ruby on Rails Inside the Container

You'll shape different aspects of Rails project in this section. **This whole section is just one-off execution.** Once you are done with this, all else is normal application development workflow.

### Setting Up `Dockerfile`

Here are the additions to `Dockerfile`

```Dockerfile
FROM starefossen/ruby-node:2-8-stretch
RUN apt-get update -qq && \
    apt-get install -y nano build-essential libpq-dev && \
    gem install bundler
RUN mkdir /project
COPY Gemfile Gemfile.lock /project/
WORKDIR /project
RUN bundle install
COPY . /project
```

**Tip**: Each `RUN` command creates a layer. It is usually a best practice to put most of those commands together in one line to avoid layers within the image. For example, `RUN apt-get update -qq & apt-get install -y nano build-essential libpq-dev` and so on. 

Here is what's going on:

* Pull the ruby+node image from Docker hub.
* Update the libraries within the image
* Install nano editor, build tools and library for Postgres.
* Install Bundler which will update existing Bundler.
* Create a folder for your project
* Copy `Gemfile` and `Gemfile.lock` from host to app folder
* Set working directory to app folder
* Run `bundle install` inside the project folder. This will install necessary gems inside the container.
* Copy rest of the content from your host folder to container app folder.

Note that copying only `Gemfile` and `Gemfile.lock` to app folder before running `bundle install` has **tremendous advantage**. Changes to the other files within the application folder do not trigger `bundle install`.Only if the `Gemfile` or `Gemfile.lock` changes, `bundle install` will be triggered. 

If you think deeply enough, you'll understand that it will save hours. If thinking deeply hurts, just try moving the command `COPY . /project` line just above `bundle install`. Every time you make even a small change within app folder, the container has to be built with all the gems being installed from scratch. This hurts more!

### Stitch Services with `docker-compose.yml`

The `docker-compose.yml` file contains instructions that stitches multiple pieces together such as database container, application container, host folder where you store your application repository, environmental aspects such as volumes and ports.

This is going to be a **database driven application**. So, we need a way to persist data created in the environment. One way is to introduce a separate service for database layer that has its own volume (fancy name for storage space).

Create a `docker-compose.yml` file inside the folder. It looks like the one below:

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
      - .:/project
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

Start by building the containers. For that you need to add:

1. an empty `Gemfile`
2. an empty `Gemfile.lock`.

Add just two lines to the `Gemfile`. It should look like:

```Gemfile
source 'https://rubygems.org'
gem 'rails', '~>5.1.5'
```

 **Once those empty files are in place**, run this on your terminal:

```
docker-compose up --build
```

You will see that `step 1/nn: FROM starefossen/ruby-node:2-8-stretch` is available **instantly** as a layer (that is if you have already tried the initial `docker run` with single line of `Dockerfile`).

That's Docker reusing the ruby image downloaded earlier. The whole process that ran last time is saved for you. So it is only from second line of code within the `Dockerfile` that matters now.

You'll see all the steps within the `Dockerfile` executed dutifully during the build process. Each of them creates a **layer** that you can recognize by their hash, that looks like `1adb3ee3e245`. 

Once build steps are over, you'll see `db` service being started first. And then the `app` service starts, but **stops** when trying to run `bundle exec rails s`, as we do not have `rails` gem within the environment yet. 

But the log on the terminal shows `man` page for Rails and shows you how to get started. Run this command to create a new Rails project. 

```bash
docker-compose run app rails new . --force --database=postgresql --skip-bundle
```

That should scaffold a new rails application in all its initial glory. As you can see, `postgresql` is the database of choice. Also, instructs the command to skip running `bundle install` after creating the project. 

Since the command creates all new `Gemfile` with necessary gem dependencies detailed, you need to build the image again. So, there is no point in running automatic `bundle install` on the container, as the gems downloaded will not be persisted.

Since the new rails project has updated `Gemfile`, you need to stop the running services and build the image agian. The following commands should get you back up online: 

```bash
docker-compose down
docker-compose up --build
```

You'd have noticed that Docker reuses most of the layers created. It does not install updates or nano editor or build tools. Because there is no change in those layers. Very smart!

Docker build picks up from where changes happened. The only change is on `Gemfile`, hence, the line `COPY` within the `Dockerfile` kicks in and build starts from there on top of existing layers.

**Explore:** Docker starts to download and install all gems once again. There are ways to cache gems locally so that they do not need to be downloaded again. Some of the online solutions are a bit old and broken. You'll have to let me know when you find a solution. 

But the good news is, **Yay! You are on rails!**. Open your favorite browser and point it at [http://localhost:3000] and you should see the whole world rejoicing at the sight of `Rails 5.1.5` and `Ruby 2.5`.

**Remember to re-build the image again if you make changes to the `Gemfile`**.

### Own The Files

All the files generated will be owned by `root`. Run `ls -la` within the terminal to look at who owns the files. Another way is to try and edit one of the generated files. If they are owned by a user other than yourself, you will have to go down the `sudo` route to get them `chown`ed. Run this command on your terminal within the application folder:

```bash
sudo chown -R weeuser:weeuser .
``` 

Apparently, use your id instead of `weeuser` in the command above.

## Git It Up

A working configuration of a brand new Rails project is usually very easy. As you saw, it just took a simple `rails new` command within the container. But it has done so much work for you, so much that it warrants a git commit.

```bash
git init
git add --all
git commit -am "first working copy"
```

That initiates git repository within the current folder. Adds all the files to staging area and commits the changes with a readable comment.

If you look closely, Rails automatically initiated the repository. It has also generated a `.gitignore` file for you with pre-populated instructions as to which folders to ignore. This file ensures unnecessary folders and files, such as log files, are kept out of the repository.

Now create a new repository within [GitHub]. Just create an empty one, no need to create any `README.md` or `.gitignore` files on the GitHub server. They are already there on your local machine.

GitHub automatically shows these instructions. Run these on your terminal to push your changes to GitHub. 

```bash
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

## Hot Reloading

### Precious Gems

[Guard](https://github.com/guard/guard) gem helps watching for file changes and run appropriate commands based on which file has changed. There are plugins that allow you to take different actions for different files. [guard-minitest](https://github.com/guard/guard-minitest) allows you to run tests when test files or app files change. [guard-livereload](https://github.com/guard/guard-livereload) refreshes the page automatically when `css`, `js` or `erb` files change.

Ensure the `Gemfile` reflects guard gems necessary for watching file changes. This will help hot reloading and automated testing.

```Gemfile
group :development, :test do
  # ...
  gem 'guard', '~>2.14.2'
  gem 'guard-livereload','~>2.5.2'
  gem 'guard-minitest', '~>2.4.6'
```

Ensure you **do not remove** any existing gems in the process. Remeber the usual drill to build the image to include these new gems. 

```bash
docker-compose up --build
```

### Initiate a Guardfile 

You need to set up `guard` to watch for file changes. Run the following command to get started.

```bash
docker-compose run --user $(id -u):$(id -g) app guard init livereload
```

That should create a `Guardfile` with instructions to watch for a list of extensions. Have a read through and you'll understand that the instructions target files that affect the rendered page such as `css`, `js` or `erb`. 

**Note**: Remember the drill to `chown` the files. Ensure you have edit access to the files. An alternative is to run docker commands as current user instead of allowing it to use root by default. That is achieved by `--user $(id -u):$(id -g)` portion of the command.

Now that the `Guardfile` is ready, you can boot it up with this command.

```bash
docker-compose run -p 35729:35729 app bundle exec guard
```

That should show `guard is now watching at /app`. It should also show that it is **waiting for browser to connect**. You'll get there in a minute. But it is **not necessary to open up a new terminal** and run that command every time. If you want, you can set it up as a docker service in itself. 

Exit out of that `guard>` prompt for now.

### Guard Docker Service 

Setup a guard service in your `docker-compose.yml` file.  You can take a copy of the `app` service we had earlier and amend it to look like this (within the same `docker-compose.yml` file of course).

```yml
  db:
  ...
  ...
  app:
  ...
  ...
  guard:
    build:
      context: .
      dockerfile: Dockerfile
    command: bundle exec guard -i
    volumes:
      - .:/project
    ports:
      - "35729:35729"
```

Livereload server uses port `35729`. You just exposed it outside of the container. This will ensure the livereload browser plugin can talk to the server. 

Bring the service down using `docker-compose down` and reboot it through `docker-compose up`. This should open up livereload in a separate service.

### Livereload Browser Plugin

On to browser now. Install livereload extension on the browser from [here](http://livereload.com/extensions/#installing-sections). It has extensions for major browsers.

Once you install the extension, you should be able to get the extension as an icon on the toolbar to make it accessible. When you click on the extension button, the container running services from `docker-compose.yml` should show `INFO - Browser connected.`. That means, **all pieces of the puzzle are now in place**.

![guard-disconnected](https://auth0.com/blog/rails/image_f31.png)
![guard-connected](https://auth0.com/blog/rails/image_f32.png)

Time to test it out. Also, time to get your own page on screen.

### First Rails Controller

Back on the terminal, run the following command to create a Controller along with an action named 'show'.

```bash
docker-compose run --user $(id -u):$(id -g) app rails g controller Home show
```

Have a good look at that default Ruby on Rails page. Because, **Yay! You are on Rails** is going bye bye and you'll replace it with most useful and comfy home page the internet has ever seen.

You can see Rails has added `get 'home/show'` by default within `config/routes.rb` file. Replace it with this now.

```
root 'home#show'
```

Now if you visit [http://localhost:3000] on your browser, or reload it once, it should show the default HTML page created when you generated the controller.

Ensure that your container is up and running with all services. Especially the Guard one. Also, ensure browser is connected to the livereload server.

Final test if Guard is really worth all the effort. Open `app/views/home/show.html.erb`, edit it to add your heart's content and save it.

**Viola!** By the time you go back to the browser, it should already have the new content.

![GIF of LiveReload]

## Guard To Automate Tests

Guard gem can also help you run tests when you change your application files. In fact, it is smart enough to run only those tests that need to be executed based on the files you change. 

You have already included necessary `guard-minitest` gem to automate running tests upon file changes, in **Precious Gems** section. You just need to get the test environment ready.

### Setup MiniTest Environment

Add this configuration to `config/database.yml`.

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  user: postgres
  port: 5432
  password:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %> 
```

This is to ensure the `db` service created within the docker container is used as database server.

And it is better to create a database instance upfront to avoid unnecessary error messages from Guard Minitest when it tries to run tests.

```bash
docker-compose run app rails db:create RAILS_ENV=test
```

### Initiate Guard Minitest 

Just like what you've done to get livereload working, guard needs to be initiated watch for file changes to run tests when necessary.

```bash
docker-compose run app guard init minitest
```

That should add additional instructions to `Guardfile`. Open the `Guardfile` in your editor and do these changes under `guard :minitest do` group:

1. **Comment** the default instructions 
2. **Un-comment** instructions generated for `Rails 4`.
3. **Change** the block to read like `guard "minitest", spring: "bin/rails test" do`. 

Using spring is optional, spring is a preloader that starts tests faster. So the tests should run without point 3 as well. Despite being in Rails 5, those commands should work just ok. 

You should be able to see **guard running all tests** when it starts. If it doesn't, stop the services using `docker-compose down` and reboot them via `docker-compose up`.

You should be able to see a failed test, if you followed on to the instructions above.

```bash
guard_1  | 1 runs, 0 assertions, 0 failures, 1 errors, 0 skips

```

Remember changing the `config/routes.rb` file to point root at `home#show` action? That's what causing the issue now. Update the file `test/controllers/home_controller_test.rb` to reflect the changes made to the `config/routes.rb` file earlier. 

The test would have auto-generated line `get home_show_url`, just change it to `get root_url`. 

```rb
  #...
  test "should get show" do
    get root_url
    assert_response :success
  end
  #...
```

Save the file and you should be able to see your tests being run by Guard. And this time, it should pass.

```bash
guard_1  | 1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

Now that you are guarded against your own inadvertent changes, you can confidently change the application as long as you have tests to cover your changes.

**Remember to commit and push** changes.

```
git add --all
git commit -m "add livereload and automated test"
git push
```

Your code is now safe in the hands of [GitHub], you should be able to see `staging` branch was used by default. You need to leave `master` branch intact. In part 2, you will use `master` branch for production copy, where you'll raise pull requests from `staging` to `master`.

## Useful Commands

Docker grows on you pretty quickly, doesn't it? You may have had loving thoughts about running all future projects in docker or run none at all. While that is all good, docker also grows on your disk space.


I refer to [this](https://lebkowski.name/docker-volumes/) post for some clean up work. Be careful when you use the commands. It is better to go step by step.

Command | Description
------- | -----------
`docker system df` | List disk usage by docker
`docker ps -a` | List all containers
`docker images` | List docker images 
`docker rmi image_name` | Remove image by repository name
`docker rmi -f abcdef` | Remove image by ID
`docker rm name` | Remove docker container by name
`docker rm abcdef` | Remove container by ID

But one last handy tip that will save a lot of keystrokes for you. Add an alias to `docker-compose` in your bash profile. You can do that by adding this line `alias dc='docker-compose'` as the last line in the file `~/.bashrc`. That allows you to run commands like this:

```bash
dc run --rm app /bin/bash
```

## Conclusion

That brings us to the end of Part 1, where you have a **dockerized workflow that gives you both live-reloading and automated testing**. 

In part 2, you'll build core application functionality on top of this workflow with identity management. You'll also extend your workflow to enable **Continuous Deployment** involving [GitHub], [Travis] and [Heroku]. You'll end it with an application running in production.


[GitHub]: https://github.com/
[Travis]: https://travis-ci.org/
[Heroku]: https://dashboard.heroku.com
[Auth0]: http://auth0.com/ 
