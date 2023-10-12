# Authorization with Pundit

<div class="bg-blue-100 py-1 px-5" markdown="1">

If you want to experiment alongside this reading, then use your `industrial-auth-1` codespace. This reading builds on the work we did there to increase security.
</div>

Now that you've got a handle on, fundamentally, how authorization is done — with conditionals and redirects — for an application of any real size, it's probably worth using an authorization library. A good one will give us helper methods that will allow us to be more concise, and more importantly will help us avoid letting security holes slip through the cracks.

There are many Ruby authorization libraries, but my go-to is one called [Pundit](https://github.com/varvet/pundit). 

In a project:

  - add `gem "pundit"` to your `Gemfile` 
  - `bundle` 

Pundit revolves around the idea of **policies**. Create a folder within `app/` called `policies/`, and within it create a file called `photo_policy.rb`. This will be a plain ol' Ruby object (PORO):

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy
end
```

The idea is that we want to encapsulate all knowledge about who can do what with a photo inside instance methods within this "policy" class. Let's set up the class to accept a user and a photo when instantiated:

```ruby{4,6-9}
# app/policies/photo_policy.rb

class PhotoPolicy
  attr_reader :user, :photo

  def initialize(user, photo)
    @user = user
    @photo = photo
  end
end
```

And now, for example, to figure out who can see a photo, let's define a method called `show?` that will return `true` if the user is allowed to see the photo and `false` if not:

<aside markdown="1">
A question mark at the end of a method name doesn't do anything special, functionally; it's just another letter in the method name, as far as Ruby is concerned.

It's a convention among Rubyists to end the names of methods that return `true` or `false` with a question mark, so I'm following that convention here with `show?`.
</aside>

```ruby{11-18}
# app/policies/photo_policy.rb

class PhotoPolicy
  attr_reader :user, :photo

  def initialize(user, photo)
    @user = user
    @photo = photo
  end

  # Our policy is that a photo should only be seen by the owner or followers
  #   of the owner, unless the owner is not private in which case anyone can
  #   see it
  def show?
    user == photo.owner ||
      !photo.owner.private? ||
      photo.owner.followers.include?(user)
  end
end
```

Let's test it out in `rails console`.

First we'll get two users:

```ruby
[1] pry(main)> alice = User.first
=> #<User id: 15>
[2] pry(main)> bob = User.second
=> #<User id: 16>
```

Then we'll see if Bob follows Alice (Bob shouldn't, so unfollow Alice if `followers.include?` returns true), and if Alice is set to private (again, if this is not the case, then set Alice to private): 

```ruby
[3] pry(main)> alice.followers.include?(bob)
=> false
[4] pry(main)> alice.private?
=> true
```

Now we get get one of Alice's photos:

```ruby
[5] pry(main)> photo = alice.own_photos.first
=> #<Photo id: 157>
```

First, we instantiate a policy for Alice, `policy_a`, and based on our `.show?` method, we check the visibility:

```ruby
[6] pry(main)> policy_a = PhotoPolicy.new(alice, photo)
=> #<PhotoPolicy:0x00007fca9886d8e0>
[7] pry(main)> policy_a.show?
=> true
```

And we do the same for Bob:

```ruby
[8] pry(main)> policy_b = PhotoPolicy.new(bob, photo)
=> #<PhotoPolicy:0x00007fca5fa3a6e8>
[9] pry(main)> policy_b.show?
=> false
```

Great! Now we can replace the conditionals that we have scattered about the application with this method. 

Let's start by locking down access to the `Photos#show` action. 

### Using `before_action`

We can start by following a similar technique as before, and utilizing our new policy in a `before_action`:

```ruby{6,21-25}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  before_action :ensure_current_user_is_owner, only: [:destroy, :update, :edit]
  before_action :ensure_user_is_authorized, only: [:show]
  
  # ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_photo
      @photo = Photo.find(params[:id])
    end

    def ensure_current_user_is_owner
      if current_user != @photo.owner
        redirect_back fallback_location: root_url, alert: "You're not authorized for that."
      end
    end

    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        redirect_back fallback_location: root_url
      end
    end
  # ...
end
```

(Mind the `!` before `...show?`; we want to redirect if the user _can't_ view the photo.)

Now we can try to navigate to a private user's photo, and we should be bounced.

### Raising An Exception

Instead of redirecting, let's take a slightly different tack: we'll raise an exception if the `current_user` isn't authorized, so that we can get an error message. 

We want to get error messages, because soon you will be setting up error monitoring systems. These errors would get logged and tracked automatically for us, so we could keep track of them for debugging.

Pundit provides a specific error message class for exactly this purpose: `Pundit::NotAuthorizedError`.

```ruby{9}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  # ...
  before_action :ensure_user_is_authorized, only: [:show]
  # ...
    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        raise Pundit::NotAuthorizedError, "not allowed"
      end
    end
  # ...
end
```

Test it out by visiting a show page that you shouldn't be able to (you can remove the `/edit` from an edit page to find a URL.)

Great! But what if we don't want to show an error page? Redirecting with a flash message was pretty nice. Well, we can rescue that specific exception in `ApplicationController`:

```ruby{5,7,9-13}
# app/controllers/application_controller.rb

class ApplicationController
  # ...
  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

    def user_not_authorized
      flash[:alert] = "You are not authorized to perform this action."
      
      redirect_back fallback_location: root_url
    end
end
```

Now we have all of the same functionality back, but we're nicely set up to encapsulate all of our authorization logic in one place.

## Convention over configuration in controllers

So far, Pundit hasn't done anything for us at all, other than providing the exception class that we raised. We wrote all the Ruby ourselves. But now, let's use some some helper methods from Pundit that will allow us to be _very_ concise, if we follow conventional naming.

First, `include Pundit` in `ApplicationController` to gain access to the methods in all of the other controllers:

```ruby{4}
# app/controllers/application_controller.rb

class ApplicationController
  include Pundit
# ...
```

Now, we can use the `authorize` method. Instead of all this:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  # ...
  before_action :ensure_user_is_authorized, only: [:show]
  # ...
    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        raise Pundit::NotAuthorizedError, "not allowed"
      end
    end
  # ...
end
```

We can write directly in the `show` action, without any `before_action` step, just this:

```ruby{6}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  # ...
  def show
    authorize @photo
  end
  # ...
end
```

🤯 What just happened? When you pass the `authorize` method an instance of `Photo`:

 - It assumes there is a class called `PhotoPolicy` in `app/policies`.
 - It assumes there is a method called `current_user`.
 - It passes `current_user` as the first argument and whatever you pass to `authorize` as the second argument to a new instance of `PhotoPolicy`.
 - It calls a method named after the action with a `?` appended on the new policy instance.
 - If it gets back `false`, it raises `Pundit::NotAuthorizedError`.

## Views with Pundit

In view templates, we now have a `policy` helper method that will make it easier to conditionally hide and show things. For example, assuming we define a `show?` method in our user's policy, like so:

```ruby
# app/policies/user_policy.rb

class UserPolicy
  attr_reader :current_user, :user

  def initialize(current_user, user)
    @current_user = current_user
    @user = user
  end

  def show?
    user == current_user ||
     !user.private? || 
     user.followers.include?(current_user)
  end
end
```

Then we can head to the view template and change this conditional:

```erb{4}
<!-- app/views/users/show.html.erb -->

<!-- ... -->
<% if current_user == @user || !@user.private? || current_user.leaders.include?(@user) %>
  <!-- ... -->
<% end %>
```

to:

```erb{4}
<!-- app/views/users/show.html.erb -->

<!-- ... -->
<% if policy(@user).show? %>
  <!-- ... -->
<% end %>
```

## Just plain ol' Ruby

Since policies are just POROs, we can bring all our Ruby skills to bear: inheritance, [aliasing](https://medium.com/rubycademy/alias-in-ruby-bf89be245f69), etc.

To start with, we can run the generator 

```
rails g pundit:install
```

which creates a good starting point policy to inherit from in `app/policies/application_policy.rb`. Take a look and see what you think. If you like it, let's inherit from it:

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy < ApplicationPolicy
```

## Secure by default

So — that means that all we need to do from now on is:

 1. Remember to define a method in our policy that matches **every** action name (but we can alias).
 2. Remember to call `authorize` within **every** action.

If we've done #2, which will force us to do #1, that will ensure that we've at least thought about who can get into every action.

Try visiting `/follow_requests` right now — oops! Quite a big security hole, and one that's depressingly common to find left open after a resource has been generated with `scaffold`.

Pundit includes a method called `verify_authorized` that we can call in an `after_action` in the `ApplicationController` to help enforce the discipline of pruning our unused routes:

```ruby{6}
# app/controllers/application_controller.rb

class ApplicationController
  include Pundit
  
  after_action :verify_authorized
  # ...
```

[There's another method called `policy_scope`](https://github.com/varvet/pundit#scopes) similar to `authorize` that's used for collections rather than single objects. Usually, you'll want to ensure that either one or the other is called with something like the following in `ApplicationController`:

```ruby
# app/controllers/application_controller.rb
after_action :verify_authorized, except: :index
after_action :verify_policy_scoped, only: :index
```

<div class="bg-blue-100 py-1 px-5" markdown="1">

If your project is setup with Devise accounts, you may need to update the `verify_authorized` method to allow those controllers:

```ruby
after_action :verify_authorized, unless: :devise_controller?
after_action :verify_policy_scoped, only: :index, unless: :devise_controller?
```
</div>

Now try visiting `/follow_requests` or some other scaffolded route that was insecurely left reachable. You'll see that you can't.

If necessary, you can make the choice to `skip_before_action :verify_authorized` on a case-by-case basis, as we did for `:authenticate_user!`. We are now secure-by-default instead of insecure-by-default.

## Read more

There's more to Pundit [that you should read about in the README](https://github.com/varvet/pundit), but not a _ton_ more. That's what I like about it — it's relatively lightweight, but gets the job done well. The way that it is structured also plays great with [Rails I18n](https://guides.rubyonrails.org/i18n.html) and other libraries. Another powerful tool for your belt.
