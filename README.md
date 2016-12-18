# Rails 5 + Devise + Omniauth

## Setting up Devise

**Background:** Devise is an all-in-one authentication gem for Rails. It allows you to quickly get a robust authentication system up and running. 

First, add devise to your `Gemfile`.
```ruby
gem 'devise'
```

After running a `bundle install`, run the devise installation generator below to copy over views from devise into your app.
```
$ rails generate devise:install
```

In order to get devise to work you have to add the following line into your `config/development.rb`:
```ruby
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

Next, change the default Devise logout HTTP verb from `DELETE` to `GET`. Change `config.sign_out_via = :delete` to `config.sign_out_via = :get` in `config/initializers/devise.rb`.

You also need to make sure that you have flash messages displaying since Devise uses them to indicate errors such as invalid email/password or login/logout messages. If you don't have something like this already, add this to your `application.html.erb`:

```ruby
<% flash.each do |name, message| %>
  <div class="alert alert-<%= name %> alert-dismissible" role="alert">
    <button type="button" class="close" data-dismiss="alert"><span aria-hidden="true">&times;</span><span class="sr-only">Close</span></button>
    <%= message %>
  </div>
<% end %>
```

Next, run the devise model generator. You can replace `User` with the name of the model that you'll be adding login to (it's probably called User). If the model doesn't exist yet, devise will create it. If it already exists, devise will simply add what it needs to the existing model. 

```
$ rails generate devise User
```

```
$ rails db:migrate
```

Now, if you go to `localhost:3000/users/sign_in` or `localhost:3000/users/sign_up`, you'll see the forms created by devise. At this point, basic signup/login/logout functionality should be working. If it isn't - make sure you did everything above correctly.

_Optional: if you'd like to make your login/logout routes have simpler names, you can edit your `routes.rb` to this: `devise_for :users, path: '', path_names: { sign_in: 'login', sign_out: 'logout'}`. Then you can simply use the `/login` and `/logout` paths._


## Setting up Omniauth

**Background:** Omniauth is a ruby gem that implements the OAuth protocol, which is a standardized protocol that allows web applications to access information from other web applications. In short, it allows you to authenticate a user via a provider (such as facebook or twitter).


We'll be implementing facebook login in our example, so add the `omniauth-facebook` gem to your `Gemfile`.
```ruby
gem 'omniauth-facebook'
```

Next, we need to add a `provider` and `uid` to our Users table. They will be used to store the name of the site which we're using to login (the provider is facebook in our case), and the unique id of the user on that site (the uid). 
```
rails g migration AddOmniauthToUsers provider:string uid:string
rake db:migrate
```

Next, you have to go to [Facebook for Developers](http://developers.facebook.com "Facebook for Developers") and register an account. Once you have done that, create a new app on the developer dashboard. You will be given an app_id and a secret key - you'll need to put these into `config/initializers/devise.rb`:
```ruby
config.omniauth :facebook, ENV['FACEBOOK_APP_ID'], ENV['FACEBOOK_SECRET']
```
Note that I'm use the dotenv-rails gem to store my facebook API credentials. You don't want to be storing your API keys in your code itself, as anyone can see it when you push it up to github. 

Next, we have to make our User model omniauthable - simply add the following to your `user.rb`:
```ruby
devise :omniauthable, :omniauth_providers => [:facebook]
```

We'll be creating a new controller for our omniauth callbacks, so set up the route for it first in `routes.rb`:
```ruby
devise_for :users, :controllers => { :omniauth_callbacks => "users/omniauth_callbacks" }
```

Next, create the corresponding controller `app/controllers/users/omniauth_callbacks_controller.rb`, and add the following code:

```ruby
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController

  def facebook
    # You need to implement the method below in your model (e.g. app/models/user.rb)
    @user = User.from_omniauth(request.env["omniauth.auth"])

    if @user.persisted?
      sign_in_and_redirect @user, :event => :authentication #this will throw if @user is not activated
      set_flash_message(:notice, :success, :kind => "Facebook") if is_navigational_format?
    else
      session["devise.facebook_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url
    end
  end

  def failure
    redirect_to root_path
  end
end
```

Finally, you need to add methods to your User model file (`app/models/user.rb`). The `from_omniauth` method runs whenever a user logs in or signs up - you should notice that we are only saving the email and password for the user. Later on you can use the same syntax to grab more information available from the Facebook API. For now, it should look like so:

```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable,
         :omniauthable, :omniauth_providers => [:facebook]

  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0,20]
      # user.name = auth.info.name   # assuming the user model has a name
      # user.image = auth.info.image # assuming the user model has an image
      # If you are using confirmable and the provider(s) you use validate emails, 
      # uncomment the line below to skip the confirmation emails.
      # user.skip_confirmation!
    end
  end

  def self.new_with_session(params, session)
    super.tap do |user|
      if data = session["devise.facebook_data"] && session["devise.facebook_data"]["extra"]["raw_info"]
        user.email = data["email"] if user.email.blank?
      end
    end
  end

end
```

That's it! You should now have facebook signup/login links on their respective Devise-generated forms. You can also paste in the link anywhere else you may like: 
```
<%= link_to "Sign in with Facebook", user_facebook_omniauth_authorize_path %>`
```
