---
layout: post
title: "Ruby on Rails - Killer Workflow with Docker (Part 2)"
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
