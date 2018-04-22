---
layout: post
title: "Ruby on Rails - Killer Workflow with Docker (Part 2)"
description: "Learn how to set up a killer dockerized workflow that will raise your productivity while developing Ruby on Rails applications."
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

By now, your development workflow must have a solid foundation based on part 1. You have continuous testing and live reloading set up already. In this second and final part, you'll connect your app to Auth0. You'll establish a pipeline to automatically deploy your changes to [Heroku]. You'll also use [Travis CI] as a quality gate to run tests before deployment.

## Cloud Authentication by Auth0

It is better to have Authentication from the start. It will force you to think about modeling records accordingly. This is where you can use the free-tier login given by [Auth0] to try it out.

### Auth0 Tenant and Client

If you have not done so already, sign up to [Auth0] and follow the instructions:

1. Sign up / Login to Auth0 
2. Create a domain/tenant (tugboat)
3. Create a client (secondstall)

I've created a tenant named *tugboat* and a client named *secondstall*. 

While creating the client, select *Regular Web Application* as type. Once the client is created, within the list of technologies shown, select **Ruby On Rails**. [Auth0] shows the [Quick Start Guide](https://auth0.com/docs/quickstart/webapp/rails/01-login) for rails by default. 

A cool feature of the guide is that the code samples show your client name and other details if you are logged in. You can reuse the code from the guide in most of the instructions used in this post, as the guide will show you code customized for your app and name you have chosen. 

It also has instructions for some of the most **commonly faced issues**. It will come in handy if you run into any such issues. The focus, for now, is on getting the authentication working and moving on to building the app.

[Auth0] reaches our application through callback URLs. So it is important to tell Auth0 client what the callback URL will be. Under settings section, find a text box that asks for **Allowed Callback URLs** and add this:

```bash
http://localhost:3000/auth/oauth2/callback
```

If you try to reach this URL on a browser now, you'll be betting a routing error. 

Time to Fix it.

## Updates at Rails End

The following are the changes at the back-end.
1. Add gem dependencies
2. Add middleware
3. Add a controller for authentication
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

The **client secret** key from Auth0 has to be available for the Rails app, but that has to be a secret. One way is to add it as a plain environment variable. But all the secrets as environment variables are [not so safe](https://www.engineyard.com/blog/encrypted-rails-secrets-on-rails-5.1) either. Rails 5.1 gives an option to encrypt secrets, store them in a file and commit them to version control. 

Rails 5.2 goes a [step further](https://www.engineyard.com/blog/rails-encrypted-credentials-on-rails-5.2), but that has to wait unless you are from the future where 5.2 is out and has gone through a few minor releases (i.e production ready).

For now, encryption from rails 5.1 will do. Following commands should create an encryption file and a key file.

```bash
docker-compose exec --user $(id -u):$(id -g) app /bin/bash
```

That takes you to the terminal within the container. Then you'll have a chance to set up secret files.

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

**Warning:** the problem with using an editor in the terminal is, if you leave any syntax errors, it is difficult to open it again. `rails secrets:edit` throws error while trying to fix the very syntax error that is causing the issue. I had to delete the encryption files and start over once to get this done. Be cautious about the syntax.

Nano editor by default shows the keys, but just in case:

* `Ctrl + o` will save the changes 
* `Ctrl + x` will close the editor

You might have noticed that the `rails secrets:setup` command asked you to set up the flag `config.read_encrypted_secrets = true`. That's the instruction asking rails to load secrets from the encrypted file. You need to add it to all three environment files:

 * `config/environments/production.rb`
 * `config/environments/test.rb`
 * `config/environments/development.rb`

You will also set up the `RAILS_MASTER_KEY` on Heroku environment. But that can wait, as the local environment has the master key in the file `secrets.yml.key` (and that **should not** be included in version control). Rails by default adds the file to `.gitignore` for you.

Now you can safely delete the plain old `secrets.yml` file.

### Add Middleware Strategy

Omniauth has many implementations called strategies. This one from Auth0 allows auth0 client to interact with your rails application.

```rb
# config/initializers/auth0.rb

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

**Note**: It is important to have `email` within the scope section for you to get the email as part of the information sent from Auth0. Read more about [scopes](https://auth0.com/docs/scopes/current) from Auth0.

The hash `u1kYusNsyW397MxD2z5oF6rWU4Jdrc4q` is the client ID. You need to replace it with your client ID from the [Auth0] site. The `auth0_client_secret` you've set up in the previous section is used here in the middleware.

### Controller To Handle Auth0 CallBack

Add a controller within rails app to handle callback and failure from Auth0. You'll change it later to add user management and logout functionalities. For now, this will do.

```rb
# app/controllers/auth0_controller.rb
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

And add relevant routes so that Rails can hand over calls to respective controller actions.

```rb
# config/routes.rb
Rails.application.routes.draw do
  root "home#show"
  get "/auth/oauth2/callback" => "auth0#callback"
  get "/auth/failure" => "auth0#failure"

end
```

This will grow as you build the application. 

Bring the container down if it is already running and boot it up again to ensure middleware changes take effect.

```bash
docker-compose down
docker-compose up
```

If that one test you had is passing the automatic test run initiated by Guard, then you should be good to go.

### The Front Page Comes Alive

You are now ready to add that much-awaited Login button.

{% highlight html %}
<!-- app/views/home/show.html.erb -->
<section class="welcome card">
  <h1>Bookshelf</h1>
  <h2>Built on Ruby on Rails secured with Auth0</h2>
  <a class="btn green" href="/auth/auth0">Login</a>
</section>
<% if Rails.env.development? %>
 <%= debug session[:userinfo] %>
<% end %>
{% endhighlight %}

If you had the container up and running with Guard, and the browser is connected to the livereload server, you should see the changes on the browser immediately.

The debug section will be blank for now, as there is no `userinfo` within session yet.

Click on that login link and it should take you to a beautiful page provided by [Auth0]. Just use the sign in option provided by Google (or any provider you've switched on within Auth0).

Once you give permission, it should redirect back to the same home page on your app.

But this time, the `userinfo` from the session is printed for you on the page, along with email and name.  

**Congratulations!** You have effectively set up a trustable cloud authentication system that you can build upon. 

![image of session info]()

## Travis CI Integration

Next up is to get help from a nice bot. [Travis] allows you to test the application when you commit new changes to [GitHub] repository or when you create pull requests and even when you merge pull requests.

Once you set up a login within Travis-CI, you should be able to add your git repository to the list.

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

Translating that, you are instructing Travis to use `ruby`. `bundle install` is the one that'll take a long time, so you are caching that for future use. You are asking for a `postgresql` service to enable database.


The actual `script` section enables database migration. You do not have any database yet, but worth making it future proof. 

Before anything runs, you've instructed Travis to use the new `database.travis.yml` and the default `database.yml`. You've also created a database for test region. This is required as the original one is adapted to run tests locally in your docker container test environment:

```yml
test:
  adapter: postgresql
  database: auth0app_test
```

**Commit and push** your changes to `staging` branch. You should see Travis coming alive once the changes are pushed.

The first build **failed!**. Get comfortable reading through the error messages on the Travis build log.

You can see that missing `RAILS_MASTER_KEY` is the reason. You can set it up on Travis repository settings page. You can access it under 'more options' menu.

Under **Environment Variables** section, add a variable named `RAILS_MASTER_KEY` and add the value from your key file stored at `config/secrets.yml.key`. Ensure you disable `Display value in build log`, that would defeat the whole purpose of keeping secrets.

Go back to the **Current** tab and use the option **Restart Build**.

You should see a **green and happy** badge showing the test was successful. 

It took over 3 minutes originally and you should see it coming down next time onwards when the cache is used.

Now that your own quality gate is ready, you can move on to publishing your application.

## Go Live With Heroku

Heroku allows you to create apps for staging and production environment. It also allows automatic deployment from different branches. Free tier allows you to run your app in production mode and test it out.

Sign up to [Heroku] first and set up a link to your GitHub profile. You can do this within **Applications** tab in **Account Settings**.

Once a link to GitHub profile is in place, create a new pipeline. While creating the pipeline, you'll have an option to connect to a GitHub repository. Connect to the repository where you've stored the app.

### Setup Staging App

Within the pipeline, **create app** under staging area. Once you create the app, you'll have an option to **configure automatic deployment**. Select `staging` branch for automatic deployment and check the option to **wait for CI**.

Now that your staging app is ready, click through to the detailed app page. There you'll have fine-grained control over all aspects of the application. 

For now, add a **Postgres** addon under **Resources** tab. 

Your next stop should be **Settings** tab where you can **Reveal Config Vars**. You should be able to see a `DATABASE_URL` in there, which was added by default when you added **Heroku Postgres** addon. But where does this environment variable go? That would be the `config/database.yml` file.

Change the `production` section to look like this:

```yml
# ...
production:
  url: <%= ENV['DATABASE_URL'] %>
```

That should also remind you about the `RAILS_MASTER_KEY`. The same **Config Vars** section allows you to add that as a key and the value for the key will be from `config/secrets.yml.key`.

Commit the change to `database.yml` file and push it to staging branch. 

This is probably the highest point of the movie. You should be able to see a lot of things coming together. You can watch these live:

* Travis tests the new changes
* Heroku Pipeline page shows *One check pending*
* Heroku starts to build the app once Travis tests are completed.
* Heroku shows a hash for the deployed version.

**You are almost there!** Now you should be able to open the app in the browser. Heroku shows that as an option on the menu. 

![Heroku automatic deployment](https://cdn.auth0.com/blog/docker-ruby/show_heroku_deployment_after_travis_test.gif)

Just one problem. The *Login* link on the home page is broken. But a helpful message **Callback URL mismatch** is shown by Auth0. That's the hint.

Go back to your Auth0 client and add the callback URL shown on the error page. Now the **Allowed Callback URLs** box should look like:

```
http://localhost:3000/auth/oauth2/callback,https://shelf3stage.herokuapp.com/auth/oauth2/callback
```

**Remember to save** the changes to settings. That covers both local and staging environment. 

If you go back to the app on the browser and click on **Login** link now, you should see the familiar Auth0 login page. If you try and log in, it should come back to the app with the home page showing plain text details of the login.

That's it. 

### Setup Production Region

There is nothing new here. You have already done all that in the staging area. You'll have to repeat the steps.

* Create another application under **production** section. 
* Use **Configure automatic deployment** option to select deployment from `master` branch this time. 
* Remember to enable **wait for CI**.
* Add a **Heroku Postgres** addon.
* **No need** to touch `database.yml`. You are already covered there.
* Add `RAILS_MASTER_KEY` config var.

But how do you deploy something to production? That's when you go back to a very well known workflow on GitHub.

You create a [Pull Request](https://help.github.com/articles/creating-a-pull-request/) on your repository. Helpfully, GitHub shows a message that shows recently published branches, along with an option to **Compare and Pull Request**.  Click on that and it should take you to a new pull request page.

Give it a good title and description. Watch out for the message **Able to merge** with a green tick. That's a sign that `master` branch can receive changes from your `staging` branch. This will be helpful when more than one person gets to push changes.

Once you create the pull request, the detailed PR page shows **One check pending** and starts to build and test the project via Travis CI.

Note: testing the pull request takes time as the bundler cache is not used from staging branch.

Once all Travis Tests are over, you'll see **All checks** have passed with the merge button turning bright green. You can now safely merge the pull request to `master` branch.

Once the pull request is merged, Travis starts the final test. You can **watch the magic** as it unfolds within the Heroku pipeline page.

Now, use the **Open app in browser** option from Heroku app. You should see the familiar page. And if you click on the **Login** link, you should see familiar failure!

Auth0 callback URL mismatch. Take that URL shown on that error page and add it to **Allowed Callback URLs** section within Auth0 client.

It looks like this now:

```
http://localhost:3000/auth/oauth2/callback,https://shelf3stage.herokuapp.com/auth/oauth2/callback,https://shelf3prod.herokuapp.com/auth/oauth2/callback
```

Back at the browser, if you load the production application and try **Login**, you should get the user details back on home screen.

So much for a full-scale workflow. You are done. From here on out, building your app is where you'd spend your time.

**Troubleshooting**: 
It will be useful for you to be aware of possible issues that you may run into. Lookout for these:

1. In case you get **Cookie Overflow** error, consult this [troubleshooting page](https://auth0.com/docs/quickstart/webapp/rails) from Auth0. Rails has a separate block for *enabling/disabling* caching within the `development.rb`. You might need to enable caching and change `config.cache_store = :null_store` to `:memory_store`

 2. You may also run into **CSRF-Detected** error while trying to log in if session store is not configured properly. Check [this issue](https://github.com/omniauth/omniauth-oauth2/issues/58) and [this one](https://github.com/mkdynamic/omniauth-facebook/issues/73) to see if you have run into one of those scenarios. Setting domain to `:all` while configuring `config/initializers/session_store.rb` helped in development, but introduced the same issue on production. You might find alternative methods in [this wiki page](https://github.com/plataformatec/devise/wiki/How-To:-Use-subdomains).

### To The Explorer In You

As you navigate through Heroku pages, you'll see an option for **Review Application**. That'll help you create apps for each pull request, just to see how your application will look like once the pull request is merged.

There are also options to create containers within Heroku and use the local docker image you have created. Explore and remember to let me know if you run into something interesting.

## Developing the App (WorkInProgress)

Take in another cup of your favorite drink. You are going put the authentication to good use in this last mile of the run.

The idea is to allow users to move books between shelves. 

### Models and DB

Start by creating necessary models with the usual commands. You'd just need three models:

* A model for Users
* A model for Books
* Another one for Shelf

While it is possible to run commands from the host terminal by creating a temporary container each time, it may not be the quickest solution. It is better to get into a bash prompt in the terminal within the container.

Run this while you are in the app folder:

```bash
docker-compose exec --user $(id -u):$(id -g) app /bin/bash
```

That should take you to a terminal within the container. 

As an aside, you might have used bash alias from part 1 to shrink docker commands. The one I use for this in my .bashrc looks like:

```bash
# ~/.bashrc or ~/.zshrc
# suffixed to the end of the file 
alias de='docker-compose exec --user $(id -u):$(id -g)'
```

You give up flexibility to add other flags, but this alias shrinks the previous command by 42 characters and looks like this:

```bash
de app /bin/bash
```

Now that you are on a terminal within the container, you can run rails commands as usual. Go ahead and create three models.

```bash
rails g model User email:string
rails g model Book name:string
rails g model Shelf place:integer user:references book:references
```

That should create database migration files, models, and tests. Before you apply the migrations to the database, **there is one important addition** to the shelves migration file.  Add an index to mark the combination of book and user as a unique combination.

Note: `rails g scaffold Book name:string` will create all the routes, controller, and actions along with tests and helper files. Try it and see if you'd like it.

```rb
# db/migrate/2018...._create_shelves.rb
class CreateShelves < ActiveRecord::Migration[5.1]
  def change
    create_table :shelves do |t|
     # ...
    end

    add_index :shelves, [:book_id,:user_id], unique: true
  end
end
```

Of course, you might want to think about a proper indexing strategy for other models.

Time to apply the database changes. Within the same terminal that runs inside the container, you can run `rails db:create` and `rails db:migrate` to apply changes.


The `Shelf` model represents the relationship between models. You'll recognize those from earlier `rails g model` statement. You also need to introduce an enum for different types of shelves.

```rb
# app/models/shelf.rb
class Shelf < ApplicationRecord
  enum place: [:wishlist, :bought, :reading, :done]
  belongs_to :user
  belongs_to :book
  validates :user_id, uniqueness: {scope: :book_id}  
end
```

The model for books looks like this. 

```rb
# app/models/book.rb
class Book < ApplicationRecord
  has_many :shelves, dependent: :destroy
  has_many :users, through: :shelves
  scope :within_shelf, ->(place,user) {
    joins(:shelves,:users) 
    .where(shelves: {place: place, users: {id: user.id}})
  }
end
```

You need to set up a scope that allows you to pull out books from a particular shelf. You'll use that scope within controllers in a minute.

Finally, the `User` model. It's not huge. In fact, you are not even going to store the name and image attributes to the database. That will be available when the user logs in from Auth0. The only thing you'll store within `User` model is the email.

```rb
class User < ApplicationRecord
    attr_accessor :name
    attr_accessor :image
    has_many :shelves, dependent: :destroy
    has_many :books, through: :shelves
end
```

Just to hydrate the database, you can create a seed file with a list of books like this:

```rb
# Shelf.delete_all
# Book.delete_all
books=Book.create([
    {name: 'book1', author: 'author1'},
    {name: 'book2', author: 'author1'},
    {name: 'book3', author: 'author1'},
    {name: 'book4', author: 'author2'},
    {name: 'book5', author: 'author3'},
])
```
Run the seeding command on a shell within the container.

```bash
docker-compose exec app /bin/bash
rails db:seed
```

This will fill the database with a list of books. In case you want to have a fresh start, you can un-comment the first two lines. Since shelf depends on book model, that one needs to be deleted first. Then all the existing books can be deleted, leaving only newly created books. Needless to say **this can cost you** dearly if you run it in production.


Now that the database is ready, you can tell your app that such a model exists and how to respond to users when they ask for it.

### Authentication Helpers

The first one is to set up `Auth0Helper` to allow authentication when necessary.

```rb
# app/helpers/auth0_helper.rb
module Auth0Helper
  private 
    def user_signed_in?
        session[:userinfo].present?
    end
    
    def authenticate_user!
        if user_signed_in?
            @current_user=build_user
        else
            redirect_to root_path
        end
    end
    
    def current_user
        @current_user
    end

    def build_user
        user_info=session[:userinfo]["info"]
        email_id=user_info["email"]
        user=User.find_by(email: email_id)
        user.name=user_info["name"]
        user.image=user_info["image"]
        user
    end
end
```

`session[:userinfo]` is constructed when Auth0 login is successful and callback method is invoked. That's the indication that user is signed in.

`authenticate_user!` is a helper method that allows you to restrict actions to users who have logged in. 

`build_user` uses email ID to fetch the user from DB. Also adds their name and avatar image URL to `user`.

Add this helper to application controller to make it available to other controllers.

```rb
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  include Auth0Helper
end

```

Another helper is required manage state from omniauth.

```rb
# app/helpers/session_helper.rb

module SessionHelper
  def get_state
    state = SecureRandom.hex(24)
    session['omniauth.state'] = state
    state
  end
end
```

One more helper to finally log out of this application.

```rb
module LogoutHelper
  def logout_url
    domain = 'tugboat.auth0.com'
    request_params = {
        returnTo: root_url,
        client_id: 'u1kYusNsyW397MxD2z5oF6rWU4Jdrc4q'
    }

    URI::HTTPS.build(
      host: domain, 
      path: '/v2/logout',
      query: to_query(request_params))
  end

  private

  def to_query(hash)
    hash.map { |k, v| "#{k}=#{URI.escape(v)}" unless v.nil? }.reject(&:nil?).join('&')
  end
end

```

This appears to be a complex one, but in reality, all it is doing is just constructing the URL for logout with different parameters. Especially, if you piece domain, path, and query, you'd get something like `tugboat.auth0.com/v2/logout?returnTo=http://localhost:3000/&client_id=u1k...`. 

### Controllers

Controllers bridge the model and view with the user. You would need controllers to handle users, books, and shelves.

```rb
# app/controllers/auth0_controller.rb
class Auth0Controller < ApplicationController
  include LogoutHelper
  def logout
      reset_session
      redirect_to logout_url.to_s
  end

  def callback
    session[:userinfo] = request.env['omniauth.auth']
    find_or_create_user!
    redirect_to '/books'
  end

  def failure
    @error_msg = request.params['message']
  end

  private

    def find_or_create_user!
        email_id=session[:userinfo]["info"]["email"]
        User.find_or_create_by!(email: email_id)
    end
end

```

This controller is something you have set up earlier. It is now ready to create users on the successful callback from Auth0.

This controller uses the email ID returned to the callback to find an existing user or create a new user when no such email is present in the database. 

Plain and simple use of the information sent from Auth0. There are loads of other information returned along, which you can find when you inspect `session[:userinfo]`. In fact, the `<%= debug session[:userinfo] %>` introduced to the view was already showing all that information that came along. You can find good use of it to design your user model.

As you can see, `logout` action has also been added here and it uses the `logout_url` helper function you've set up earlier.

Books need a controller. That's what you'd do next.

```rb
# app/controllers/books_controller.rb
class BooksController < ApplicationController
  before_action :authenticate_user!
  before_action :set_book, only: [:show, :edit, :update, :destroy]

  def index
    if(params[:place].nil?)
      @books=Book.all
    else 
      @books=Book.within_shelf(
        params[:place],
        current_user)
    end
  end

  def show
    current_shelf=@book.shelves.by_user(current_user).first
    if current_shelf.blank?
      @shelf=Shelf.new
    else 
      @shelf=current_shelf
    end 
  end

  def new
    @book = Book.new
  end

  def create
    @book = Book.new(book_params)
    if @book.save
      redirect_to @book
    else
      render :new
    end
  end

  private
    def set_book
      @book = Book.find(params[:id])
    end

    def book_params
      params.require(:book).permit(:name, :author)
    end
end

```

That's a slimmed down version of the controller to make it easy to understand. Rails uses REST architecture and you can see the controller is ready to handle the usual CRUD (Create, Read, Update, Delete) operations. `before_action :authenticate_user!` takes care of authentication before any user can create/amend a book. `authenticate_user` comes from the `Auth0Helper` you've set up earlier.

`book_params` method is also an important one that protects you from any malicious extra parameters sent in by the aliens.

*Note*: Just like scaffolding, you can run `rails g resources Books name:string` to generate boilerplate for necessary Model and Controllers. You'll get both HTML and JSON templates generated for you to play around. But writing your own actions helps you think more about each of the actions. Playing with both scaffolding and writing code on your own can help you learn Rails internals.

Go ahead and create another controller for shelves. This controller binds books to users. 

```rb
# app/controllers/shelves_controller.rb
class ShelvesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_shelf, only: [:update]

  def create
    @shelf=Shelf.find_or_create_by(
      user_id: current_user.id, 
      book_id: shelf_params[:book_id],
      place: shelf_params[:place]
    )
    redirect_to @shelf.book
  end

  def update
      @shelf.update(shelf_params)
      redirect_to @shelf.book
  end

  private

    def set_shelf
      @shelf = Shelf.find(params[:id])
    end
    def shelf_params
      params.require(:shelf).permit(:place, :book_id)
    end
end

```
To simplify things, you'll set up create and update actions. While creating the shelf, you'll have to pass in the user along with other params. But while updating, the user is already set up, so you just need to pass rest of the params.

You need to give one last visit to the home controller.

```rb
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  
  before_action :authenticate_user!, only: [:profile]
  
  def show
  end

  def profile
  end
end
```

As you'll remember, `show` action is the home page when the user is not logged in. You need to add `profile` action and make it available only when the user is logged in. This is achieved through the use of `authenticate_user`.

You are now ready to show off your actions to users with views. 

### Views

Views present information from controllers coming from models in HTML format (and also JSON if you are using scaffolding along with `jbuilder` gem). 

Start from the top. `application.html.erb` is the base template for all of your Rails controllers.

{% highlight html %}
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>
  <!-- leave everything within head tag intact -->
  <!-- unless you know what you are doing -->
  </head>
  <body>
    <header><%= render 'layouts/navbar' %></header>
    <main>
    <%= yield %>
    </main>
  </body>
</html>

{% endhighlight %}

You need to add a partial for navigation. Include that partial within this `application.html.erb` and that'll show up on all pages. By the way, that small `<%= yield %>` at the bottom is the placeholder for rest of your pages.

On to the navigation partial now.

{% highlight html %}
<!-- app/views/layouts/_navbar.html.erb-->
<nav>
    <div> 
        <%= image_tag "book.svg",class: "logo" %>
        <h1>
        <%= link_to "BookShelf", books_path%>
        </h1>
    </div>
    <ul>
        <% if user_signed_in? %>
            <li><%= link_to "Profile",profile_path %></li>
            <li><%= link_to "Logout",auth_logout_path %></li> 
        <% end %>
    </ul>
</nav>
{% endhighlight %}

That's just an image and a title on the left. On the right, you will add links to profile and log out.

Few more views to handle books, and you'll be done.

Start with a view to creating new books.

{% highlight html %}
<!-- app/views/books/new.html.erb-->
  <div class="new_book">
  <h1>New Book</h1>
  <%= render 'form', book: @book %>
  <%= link_to 'Back', books_path %>
</div>
{% endhighlight %}

The view names are usually derived from controller actions. That's how Rails know which view to render.

The corresponding `new` action from controllers gives context under `@book`. That'll be used in the form partial to create a book. That's up next.

{% highlight html %}
<!-- app/views/books/_form.html.erb-->
<%= form_with(model: book, local: true) do |form| %>

  <div class="field">
    <%= form.label :name %>
    <%= form.text_field :name, id: :book_name %>
  </div>

  <div class="field">
    <%= form.label :author %>
    <%= form.text_field :author, id: :book_author %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>

<% end %>

{% endhighlight %}

Rails takes care of creating a form that can send a POST request to controller action `create` when this form is submitted.

Now that you can create books, next is to show already created book. That's easy.

{% highlight html %}
<article class="book">
  <h1><%= @book.name %></h1>
  <p>
    <strong>Author:</strong>
    <%= @book.author %>
  </p>
  <%= render 'shelf_form', shelf: @shelf %>
</article>
{% endhighlight %}

Except for the last line, everything else is just business as usual. Use the `@book` instance variable from the controller to get book details and render them. The last line shows a form for the current shelf for the book so that the user can easily change the shelf if required. 

The groundwork for this activity was already done within books controller's `show` action. If you review that action, you'll see `@shelf` was prepared for this moment.

You'll use that to render a small form that has nothing but a drop-down showing current shelf, and an option to move it to another shelf.

This form lives in a partial.

{% highlight html %}
<!-- app/views/books/_shelf_form.html.erb-->
<%= form_with(model: shelf, local: true) do |form| %>
  <div class="field">
    <%= form.label :place,"Shelf"%>
    <%= form.select(:place,
        options_for_select(Shelf.places.keys,shelf.place),
        :include_blank=>true,
        id: :shelf_place) 
    %>
  </div>
  <%= form.hidden_field :book_id, :value=>params[:id] %>
  <div class="actions">
    <%= form.submit "Move to shelf" %>
  </div>
<% end %>
{% endhighlight %}

The variable `shelf` was passed by the `show` template. This partial makes use of it to render a select drop-down with all `Shelf` places from keys. The move to shelf button invokes `create` action if `shelf` is empty. It'll invoke `update` action if `shelf` already has valid values.

Final sprint. Index of all books and by their place in the shelf. This again takes two files. One to show index and another to show a filter for shelves as top menu.

{% highlight html %}
<!-- app/views/books/index.html.erb -->
<div class="books">
<%= render 'shelf_list' %>
<%= link_to "Add New Book", new_book_path , class: 'new_book' %>
<section class="book_list">
    <% @books.each do |book| %>
      <article class="book_entry">
        <h2><%= link_to book.name.capitalize, book %></h2>
        <h3><%= book.author.capitalize %></h3 >
      </article>
    <% end %>
</section>
</div>

{% endhighlight %}

That view takes care of listing all the books sent from `index` action within books controller.

In addition, it has a link to add new books at the bottom. But there is a partial at the top that lists option to filter books by shelves.

The `shelf_list` partial gives a list of links that will filter books.

{% highlight html %}
<!-- app/views/books/_shelf_list.html.erb -->
<section class="shelves">
    <%= link_to "All",books_path %>
    <% Shelf.places.keys.each do |shelf| %>
        <%= link_to shelf.capitalize,books_path(place: shelf) %>
    <% end %>
</section>
{% endhighlight %}

That partial adds links to each type of shelf. Passes each shelf as a parameter to books controller. If you look backward, the `index` action from books controller used a parameter `params[:place]. This is where it is coming from.

The profile view is the one outstanding and a small one to finish views.

{% highlight html %}
<!-- app/views/home/profile.html.erb -->
<section class="profile card">
  <h2><img src='<%= current_user.image %>' /></h2>
  <h1><%= current_user.name %></h1>
  <%= link_to "Logout",auth_logout_path, class: "btn red" %>
</section>


{% endhighlight %}

Now, go ahead and tell Rails about the URLs you are prepared to answer. That'll be done through routes.

### Routes

Routes allow you to lay down list or URLs that the app will respond to. First stop is to **set up the routes** for books, shelves, profile and log out. Add routes to the file `config/routes.rb`. The file looks like this:

```rb
# config/routes.rb
Rails.application.routes.draw do
  root 'home#show'
  get "/auth/oauth2/callback" => "auth0#callback"
  get "/auth/failure" => "auth0#failure"
  get "/profile" => "home#profile"
  get "/auth/logout" => "auth0#logout"
  resources :books
  resources :shelves
end

```

You already had some of those routes. In case you are curious, try loading `http://localhost:3000/books` now. It should redirect you to the login page. Once you log in, it should take you to book list.

### Styles

The application views may not be very impressive at first sight. But you should be able to add styles that'll breathe life into them.

`app/assets/stylesheets/` folder holds stylesheets for the application. `application.css` builds all stylesheets into one asset. You are free to add as many stylesheets as you want.

You can also use [Sass] files as Rails comes with built-in support. 

For example, add a new file `home.scss`. Add following style rules. 

```scss
/* app/assets/stylesheets/home.scss */

.card {
    max-width: 18em;
    padding: 1em;
    border-radius: 3px;
    box-shadow: 0 2px 4px grey;
    text-align: center;
    margin: 0 auto;
    background: rgba(255,255,255,.5);
}

```

That immediately center aligns text with box shadow and moves the whole section to the center of the screen.

Another example, the top navigation.

```scss
/* app/assets/stylesheets/header.scss */
nav {
    background: rgba(100,250,100,.5);
    display: flex;
    border-radius: 2px;
    margin-bottom: 10px;
    justify-content: space-between;
    div { display: flex; }
    .logo { width: 3em; }
    h1 { margin: auto .1em; }
    ul {
      list-style: none;
      display: flex;
      justify-content: space-around;
      padding: 0;
    }
    li { padding: .5em; }
}

```

Have a look at [CSS reference at MDN](https://developer.mozilla.org/en-US/docs/Web/CSS) if you are new to stylesheets. Also, worth checking [Sass](https://sass-lang.com/). Truly, CSS with superpowers. Apart from nesting selector styles, it has several other features. A lot of them are and will come into native CSS. 

### Deploy

You've done a lot of work. In fact, too many files that they should already be on Git. Commit and Push your changes. 

* Watch tests as they run on [Travis CI]. Check.
* Watch [Heroku] deploy the app to staging. Check.
* Open staging app and create a book. Uh ho!

There is something missing. That is, you need to migrate the database schema to Heroku. Remember running `rails db:migrate`? You need to tell Heroku to run that command whenever you deploy. Heroku has a release feature mentioned in [this post](https://mentalized.net/journal/2017/04/22/run-rails-migrations-on-heroku-deploy/) will help.

You need to set up a `Procfile` within the project root directory.

```rb
web: bundle exec puma -C config/puma.rb
release: rails db:migrate
```

The first one `web` declares the web server. The second one `release` is the one that will be executed once build is completed.

**Logs:**
There are three logs on Heroku that you'll find useful.
1. If you open staging pipeline on Heroku, you'll be able to see **log generated during build** under **Activity** tab. This shows the build activities and their status.
2. The same activity tab will also show **Release log** generated while running the `release` activity in `Procfile`. In this case, it would show database migrations.
3. You'll also be able to see general application runtime log using **More** options menu at the top right corner.

Use these to find out what's happening when you run into an issue.

The same **More Options** menu gives you control over a console within Heroku. You can use that console to run rails commands such as `rails db:rollback` and `rails db:migrate` when required. 

Open staging environment in the browser. You can do this from Heroku page itself. Try and create a book. Things should work like they've worked in development now.

The reason? `release` tag in the `Procfile` migrated the database schema. So users and books can be created without any trouble.

Great! Take a break, you deserve it!


## Debugging

You need a few more blades in your swiss army knife to debug the application. You can use these 4 options depending on what you are debugging.

### Debug Information on Views

This one is the easiest one. Just include a `<% debug %>` tag to the views (`erb` files) and that shows up on the pages. 

You have already done that on the `app/views/home/show.html.erb` file. You have included `<%= debug session[:userinfo] %>` to look into what was returned from [Auth0].

**Warning:** This tag will be rendered on staging and production as well. To make it available for development region, you can change that line to show the following:

{% highlight html %}
<% if Rails.env.development? %>
 <%= debug session[:userinfo] %>
<% end %>
{% endhighlight %}

But really, you should think about removing that line altogether before you commit and push the code. You don't want rails to check for the environment in production, each time someone visits that home page, do you? 

### Logging to Console

Logging to console is a serious business. But, that's exactly what you are going to do.

Let's say you tried setting up a secret key base, stored the Auth0 client secret encrypted and one MASTER_KEY to rule them all. But things are not working, you want to see if the decryption works ok.

Adding that information to a view will be disastrous right? Everyone will see that information on the screen.

**But adding to the console is not safe either.** If someone gains access to your server logs, they will be able to figure out a lot of things from the logs alone. You don't want to hand them the key to your whole app.

Try that here. Add this line to `show` method within `app/controllers/home_controller.rb` file.

```rb
    logger.debug Rails.application.secrets.auth0_client_secret
```

That's single, but a long line. Once you save the file, go to http://localhost:3000 on your browser and check your console log. There it is, bright as day, your encrypted Auth0 client secret.

**But that's about what you shouldn't do.** You can use logger for printing out information that is harmless. You can remove it after debugging. You can leave it if it would add value to production. You take a call.

### Rails Console On Terminal

This is quite straightforward. On-demand access to Rails console on the terminal. You'll get into a shell within the container and then run `rails c`. Start the container using `docker-compose up` and then issue these commands:

```bash
docker-compose exec app /bin/bash
rails console
```

The console gives you access to Rails application. You can interact with ActiveRecord and general Rails functionalities. Issue a command like `User.all` and `Book.all` to find out what's happening.

More on [Rails console on the guides.](http://guides.rubyonrails.org/command_line.html#rails-console)

### Console Within Views

This one is an interesting option. A console right there in the views on the browser. Your commands are sent to the server and response is sent back to the browser. 

Very powerful. **Very risky**, as it can talk to the server. This is enabled by `web-console` gem in the `Gemfile`. Have a look at the grouping. The gem should be inside `group :development do` section. Which means, this is available only in development environment. Not in test or production environment.

How do you use it? 

Place this line in your views: `<%= console %>`. If you want it on all pages, better put that right under `<% yield %>` statement on `app/views/layouts/application.html.erb` file.

Now, go ahead and revisit the home page. Do you see the console at the bottom? You should (see the troubleshooting section below if you don't).

You get a black section with a prompt `>>`. You can run any command that you run within a Rails console. For example, run `User.all` and you'll see a list of accounts with their email IDs (You should think of that as spammers treasure trove). This works even if you log out and load the home page. That's a big red-flag. **This console will work even without authentication**, as it is directly talking to rails database.

**Troubleshooting:** In case you don't, the terminal log running local server may have an answer. If you see something like `Cannot render console from 172.19.0.1!` that would be due to the container's lack of access. This can be fixed by white-listing container IPs in the development environment:

```rb
# config/environments/development.rb
# ...
config.web_console.whitelisted_ips = ['172.19.0.1']
# ... just leave all else intact
```
You need to restart the server and access the page again.

**TK**; wouldn't this break for other developers if the IP changes? 

Remember to remove the console. Otherwise, add console conditionally for development region alone. Use the same strategy you've used for `<%= debug %>`.

### Bye Bug!

`byebug` is a gem that allows you to dig into the runtime at any spot of the application. You already have it in your `Gemfile`.

For example, add `byebug` to `show` method in Home controller.

```rb
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  # ...
  def show
    byebug if Rails.env.development?
  end
  # ...
end

```

But that would need interaction on the terminal. `docker-compose up` was not built to handle inputs. It just logs the output. See this [issue](https://github.com/docker/compose/issues/4677).

You need to open up `stdin_open` and `tty` within `docker-compose.yml`. Do not remove anything from the file. Just add the last two flags.

```yml
#...
services:
  db:
    #...
  app:
    build:
    #...
    tty: true
    stdin_open: true

```

While debugging with `byebug`, you need to use `docker-compose run --service-ports app` instead of `docker-compose up`. 

Once the server is ready, reload the home page. If you notice, the home page keeps loading for a long time. That is because byebug caught up the execution and waiting for you on the terminal.

Go ahead and have a look at the terminal. `byebug` shows the line where it paused execution. You can type `help` or `var` and hit `Enter` to get some output.

`continue` lets you proceed further. `next` takes you to the next step. Learn more about `byebug` [here](https://github.com/deivid-rodriguez/byebug).

Did you notice *anything strange*? Did the characters you typed on the terminal never came up, but the output did? It did for me. I'm typing in without looking at the characters. If you know how to solve this, please do let me know!

More on debugging in the [Rails Guides](http://guides.rubyonrails.org/debugging_rails_applications.html)


## Conclusion

Thank you so much for staying with me so far. I know this is a dense article and it's great you've come this far. Hope this helped you think about creating that one app you had at the back of your mind. Once you get comfortable with the workflow, it encourages you to reuse most of it. Such as the `Gemfile` and `docker-compose.yml`, which makes it easy to spring up new applications.

But that's not all, your journey ends only when you tweak the workflow to improvise it and make it stick to support your habits. 

I invite you to share your thoughts. Anything that can be improved or anything that could speed things up? Share that in the discussion, it will benefit all of us. See you there.

Finally, thanks to [Bruno Krebs](https://twitter.com/brunoskrebs) for his excellent insights during the review. 

[GitHub]: https://github.com/
[Travis]: https://travis-ci.org/
[Heroku]: https://dashboard.heroku.com
[Auth0]: http://auth0.com/ 

