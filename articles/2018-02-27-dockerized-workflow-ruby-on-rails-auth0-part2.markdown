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
- 2018-02-27-dockerized-workflow-ruby-on-rails-auth0
---

Your workflow must have a solid foundation based on part 1. You'll connect your app to Auth0. You'll establish a pipeline to automatically deploy your changes to [heroku]. You'll also use [Travis CI] as a quality gate to run tests before deployment.

## Cloud Authentication by Auth0

It is better to have Authentication from the start. It will force you to think about modeling records accordingly. This is where you can use the free-tier login given by [Auth0] to try it out.

### Auth0 Tenant and Client

If you have not done so already, sign up to [Auth0] and follow the instructions:

1. Sign up / Login to Auth0 
2. Create a domain/tenant (tugboat)
3. Create a client (secondstall)

I've created a tenant named *tugboat* and a client named *secondstall*. 

While creating the client, select *Regular Web Application* as type. Once the client is created, within the list of technologies shown, select **Ruby On Rails**. [Auth0] shows the [Quick Start Guide](https://auth0.com/docs/quickstart/webapp/rails/01-login) for rails by default. 

A cool feature within the guide is that, the code samples show your client name and other details if you are logged in. You can reuse the code from the guide in most of the instructions used in this post, as the guide will show you code customized for your app and name you have chosen. 

It also has instructions for some of the most **commonly faced issues**. It will come in handy if you run into any such issues. The focus, for now, is on getting the authentication working and moving on to building the app.

[Auth0] reaches our application through callback URLs. So it is **important** to tell Auth0 client what the callback URL will be. Under settings section, find a text box that asks for **Allowed Callback URLs** and add this:

```bash
http://localhost:3000/auth/oauth2/callback
```

If you try to reach this URL on a browser now, you'll be betting a routing error. 

Time to Fix it.

## Updates at Rails End

The following are the changes at the back-end.
1. Add gem dependencies
2. Add middleware
3. Add controller for authentication
4. Add secrets
5. Add Landing Pages


### Gem Dependencies

Add these two gems to your `Gemfile`. `omniauth` provides basic authentication capabilities while `omniauth-auth0` implements a strategy that allows authentication from multiple providers.

```Gemfile
# ...
gem 'omniauth', '~> 1.8.1'
gem 'omniauth-auth0', '~> 2.0.0'
```

Remember the usual drill to build the container. Run this from terminal:

```bash
docker-compose up --build
```


### Secrets Need To Stay So

The **client secret** key from Auth0 has to be available for the Rails app, but that has to be a secret. One way is to add it as a plain environment variable. But all the secrets as environment variables is [not so safe](https://www.engineyard.com/blog/encrypted-rails-secrets-on-rails-5.1) either. Rails 5.1 gives an option to encrypt secrets, store them in a file and commit them to version control. 

Rails 5.2 goes a [step further](https://www.engineyard.com/blog/rails-encrypted-credentials-on-rails-5.2), but that has to wait, unless you are from the future where 5.2 is production ready.

For now, encryption from rails 5.1 will do. Following commands should create an encryption file and a key file.

```bash
docker-compose run --user $(id -u):$(id -g) app /bin/bash
```

That takes you to the terminal within the container. Then you'll have a chance to setup secret files.

And for the secrets themselves, they come from below areas:
1. `auth0_client_secret` is the value from your [Auth0] client.
2. `secret_key_base` is something you can generate by using: `docker-compose run app rails secret`. If you are already inside a terminal within the container, you just have to run `rails secret`

When you are in the terminal within the container, run these:

```bash
rails secrets:setup
EDITOR=nano rails secrets:edit
```

That'll open up the file in an editor within the terminal. Set it up to look like the one below:

```yml
# ....
default: &default
  auth0_client_secret: fed4ac99...

development: 
 <<: *default
 secret_key_base: fa23ade...

test: 
 <<: *default
 secret_key_base: ee84ada...

production: 
 <<: *default
 secret_key_base: ddcc4ae...

```

**Warning:** the problem with using an editor in the terminal is, if you leave any syntax errors, it is difficult to open it again. `rails:edit` throws error while trying to fix the very syntax error that is causing issue. I had to delete the encryption files and start over once to get this done. Be cautious about the syntax.

Nano editor by default shows the keys, but just in case:

* `Ctrl + o` will save the changes 
* `Ctrl + x` will close the editor

You might have noticed that the `rails secrets:setup` command asked you to set up the flag `config.read_encrypted_secrets = true`. That's the instruction asking rails to load secrets from the encrypted file. You need to add it to add three environment files:

 * `config/environments/production.rb`
 * `config/environments/test.rb`
 * `config/environments/development.rb`

You will also setup the `RAILS_MASTER_KEY` on Heroku environment. But that can wait, as the local environment has the master key in the file `secrets.yml.key` (and that **should not** be included in version control). Rails by default adds the file to `.gitignore` for you.

Now you can safely delete the plain old `secrets.yml` file.

### Add Middleware Strategy

**TK** - the middleware strategy allows auth0 client to interact with your rails application.

```rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider(
    :auth0,
    'u1kYusNsyW397MxD2z5oF6rWU4Jdrc4q',
    Rails.application.secrets.auth0_client_secret,
    'tugboat.auth0.com',
    callback_path: "/auth/oauth2/callback",
    authorize_params: {
      scope: 'openid email profile',
      audience: 'https://tugboat.auth0.com/userinfo'
    }
  )
end
```

**Note**: It is important to have `email` within the scope section for you to get email as part of the information sent from Auth0.

The hash `u1kYusNsyW397MxD2z5oF6rWU4Jdrc4q` is the client ID. You need to replace it with your client ID from the [Auth0] site. The `auth0_client_secret` you've setup in the previous section is used here in the middleware.

### Controller To Handle Auth0 CallBack

Add a controller within rails app `app/controllers/auth0_controller.rb`:

```rb
class Auth0Controller < ApplicationController
  def callback
    session[:userinfo] = request.env['omniauth.auth']
    redirect_to '/'
  end

  def failure
    @error_msg = request.params['message']
  end
end

```

And add the actions to `config/routes.rb`:

```rb
Rails.application.routes.draw do

  root "home#show"
  get "/auth/oauth2/callback" => "auth0#callback"
  get "/auth/failure" => "auth0#failure"

end
```

Bring the container down if it is already running and boot it up again to ensure middleware changes take effect.

```bash
docker-compose down
docker-compose up
```

If that one test you had is passing within the automatic test run initiated by Guard, then you should be good to go.

### The Front Page Comes Alive

You are now ready to add that much awaited Login button.

```erb
<!-- app/views/home/show.html.erb -->

<section class="jumbotron text-center">
  <h1>Bookshelf</h1>
  <h2>Built on Ruby on Rails secured with Auth0</h2>
  <a href="/auth/auth0">Login</a>
  <%= debug session[:userinfo] %>
</section>
```

If you had the container up and running with Guard, and your browser plugin is connected to livereload server, you should see the changes on the browser immediately.

Apparently, there is no `userinfo` within session yet.

Click on that login button and it should take you to this beautiful page provided by [Auth0]. Just use the sign in option provided by google (or any provider you've switched on within Auth0).

Once you give permission, it should redirect back to the same home page on your app.

But this time, the `userinfo` from the session is printed for you on the page, along with email and name.  

**Congratulations!** You have effectively setup a trustable cloud authentication system that you can build upon. 

![image of session info]()

## Travis CI Integration

Next up is to get help from a nice bot. [Travis] allows you to test the application when you commit new changes to [GitHub] repository or when you create pull requests and even when you merge pull requests.

Once you setup a login within Travis-CI, you should be able to add your git repository to the list.

Click on that small **+** button just beside **My repositories**. It should list down your git repositories. If you cannot find it, try **Sync Account** once.

You can click on that small cog that denotes settings and you should be able to select when do you want to build.

Switch on **Build only if .travis.yml is present** option. 

and add a file named `.travis.yml` to the project root folder with this content:

```yml
language: ruby
cache: bundler
rvm:
  - 2.5.0
services:
  - postgresql
before_script:
  - cp config/database.travis.yml config/database.yml
  - psql -c 'create database auth0app_test;' -U postgres
script:
  - bin/rake db:migrate RAILS_ENV=test
  - bin/rake 
```

Translating that, you are instructing travis to use `ruby`. `bundle install` is the one that'll take a long time, so you are caching that for future use. You are asking for a `postgresql` service to enable database.


The actual `script` section enables database migration. You do not have any database yet, but worth making it future proof. 

Before anything runs, you've instructed travis to use the new `database.travis.yml` and the default `database.yml`. You've also created a database for test region. This is required as the original one is adapted to run tests locally in your docker container test environment:

```yml
test:
  adapter: postgresql
  database: auth0app_test
```

**Commit and push** your changes to `staging` branch. You should see travis coming alive once the changes are pushed.

The first build **failed!**. Get comfortable reading through the error messages on the Travis build log.

You can see that missing `RAILS_MASTER_KEY` is the reason. You can set it up on Travis repository settings page. You can access it under 'more options' menu.

Under **Environment Variables** section, add a variable named `RAILS_MASTER_KEY` and add the value from your key file stored at `config/secrets.yml.key`. Ensure you disable `Display value in build log`, that would defeat the whole purpose of keeping secrets.

Go back to the **Current** tab and use the option **Restart Build**.

You should see a **green and happy** badge showing the test was successful. 

It took over 3 minutes originally and you should see it coming down next time onwards when the cache is used.

Now that your own quality gate is ready, you can move on to publishing your application.

## Dev Workflow

### Git Workflow
* maintain a staging branch in github
* commits to staging branch shoult be tested 
* commits that are passing through tests should be deployed to heroku
* pull requests from staging to master 

### Debugging
* show how to use `logger.debug`
* show how to use `inspect` within views
* show how to use `byebug`
* show how to open up `rails console` within docker

### Live Reload
* Introduce live reloading with guard

## Continuous Testing on Travis CI

* add .travis.yml
* create a separate database.yml file for travis (as the original one uses local test environment on docker, host name wouldn't match)
* postgres should be added as a service to travis
* before script can be used to create database
* db:migrate is required as first script before test runs (to create a working database copy)
* it is better to let travis build a new environment, but look into caching when necessary

## Deploy to Heroku

### New Heroku Pipeline Architecture
* Create a new heroku account
* Explain the concept of pipelines in Heroku
* Create a new pipeline

### Setting Up Staging
* create a staging app within the pipeline
* Create a postgres addon
* That creates a DATABASE_URL config var
* while creating it, connect to github repository
* enable automatic deploy
* deploy manually once
* Use that "DATABASE_URL" var within database.yml
* run "rails db:migrate" within console given in heroku
* Change the code and commit
* It should trigger a build
* Use the log menu under options to check logs

### Setting Up Production 
* Repeat heroku instructions above for production app within the pipeline.
* Link production to `master` branch of git repository.
* Change the app and deploy to staging
* Create a pull request in master branch
* Check automatic deployment to heroku

## Conclusion
Takes us to the end of workflow where changes are automatically tested locally. Commits to staging is tested and deployed to staging environment. Merged pull requests in master trigger another deployment to production.

Leave users with topics to explore further. Leave questions that users can add value by answering.


[GitHub]: https://github.com/
[Travis]: https://travis-ci.org/
[Heroku]: https://dashboard.heroku.com
[Auth0]: http://auth0.com/ 
