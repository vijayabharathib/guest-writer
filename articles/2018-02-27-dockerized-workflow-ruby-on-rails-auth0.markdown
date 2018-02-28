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

**TL;DR:** If you are looking to bootstrap a full-scale database driven application using Ruby on Rails with passwordless authentication, look no further. This article will make it easy to convert ideas into testable [MVPs](https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMinimum_viable_product) (minimum viable products). We are going to build a small shelf. We'll fill them with books in the process. You can have a look at the app [here](https://bookstall.herokuapp.com/) and [repository on GitHub](https://github.com/vijayabharathib/auth0_rails_docker) has the code and configuration to start with.

## Development Workflow

The objective is to optimize for developer happiness. Ruby on Rails fits the bill. We'll figure out a workflow that is easy to setup, test, debug and deploy.  We will enter the world of [continuous deployment](https://en.wikipedia.org/wiki/Continuous_delivery) where commits are deployed automatically on successful test run.

### Tech Stack

**1. Docker**

Get [docker installed](https://docs.docker.com/install/). Use `docker -v` and verify you have a working version of docker. The one I have is `Docker version 17.12.0-ce, build c97c6d6`. We'll talk more about docker in a minute (or in an hour if you have to go through all the setup here).

**2. Docker Compose**

[Docker compose](https://docs.docker.com/compose/install/) will allow us to piece multiple services together. Try `docker-compse -v` to ensure a working version. Here is the result from my terminal `docker-compose version 1.16.1, build 6d1ac21`

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

