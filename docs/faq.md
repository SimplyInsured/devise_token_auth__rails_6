## FAQ

### I have missing headers or issues with batch requests

Try disabling `change_headers_on_each_request`, it's a nice to have security enhancement but not crucial. If you are curious, you can check how we [manage the tokens and batch requests](conceptual.md)

### Can I use this gem alongside standard Devise?

Yes! But you will need to enable the support of separate routes for standard Devise. So do something like this:

#### config/initializers/devise_token_auth.rb
~~~ruby
DeviseTokenAuth.setup do |config|
  config.enable_standard_devise_support = true
end
~~~

#### config/routes.rb
~~~ruby
Rails.application.routes.draw do

  # standard devise routes available at /users
  # NOTE: make sure this comes first!!!
  devise_for :users

  # token auth routes available at /api/v1/auth
  namespace :api do
    scope :v1 do
      mount_devise_token_auth_for 'User', at: 'auth'
    end
  end

end
~~~

### Another method for using this gem alongside standard Devise (updated May 2018)

Some users have been experiencing issues with using this gem alongside standard Devise, with the `config.enable_standard_devise_support = true` method.

Another method suggested by [jotolo](https://github.com/jotolo) is to have separate child `application_controller.rb` files that use either DeviseTokenAuth or standard Devise, which all inherit from a base `application_controller.rb` file. For example, you could have an `api/v1/application_controller.rb` file for the API of your app (which would use Devise Token Auth), and a `admin/application_controller.rb` file for the full stack part of your app (using standard Devise). The idea is to redirect each flow in your application to the appropriate child `application_controller.rb` file. Example code below:

#### controllers/api/v1/application_controller.rb
Child application controller for your API, using DeviseTokenAuth.
~~~ruby
module Api
  module V1
    class ApplicationController < ::ApplicationController
      skip_before_action :verify_authenticity_token
      include DeviseTokenAuth::Concerns::SetUserByToken
    end
  end
end
~~~

#### controllers/admin/application_controller.rb
Child application controller for full stack section, using standard Devise.
~~~ruby
module Admin
  class ApplicationController < ::ApplicationController
    before_action :authenticate_admin!
  end
end
~~~

#### controllers/application_controller.rb
The base application controller file. If you're using CSRF token protection, you can skip it in the API specific application controller (`api/v1/application_controller.rb`).
~~~ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
~~~

#### config/initializers/devise_token_auth.rb
Keep the `enable_standard_devise_support` configuration commented out or set to `false`.
~~~ruby
# config.enable_standard_devise_support = false
~~~

### Why are the `new` routes included if this gem doesn't use them?

Removing the `new` routes will require significant modifications to devise. If the inclusion of the `new` routes is causing your app any problems, post an issue in the issue tracker and it will be addressed ASAP.

### I'm having trouble using this gem alongside [ActiveAdmin](https://activeadmin.info/)...

For some odd reason, [ActiveAdmin](https://activeadmin.info/) extends from your own app's `ApplicationController`. This becomes a problem if you include the `DeviseTokenAuth::Concerns::SetUserByToken` concern in your app's `ApplicationController`.

The solution is to use two separate `ApplicationController` classes - one for your API, and one for ActiveAdmin. Something like this:

~~~ruby
# app/controllers/api_controller.rb
# API routes extend from this controller
class ApiController < ActionController::Base
  include DeviseTokenAuth::Concerns::SetUserByToken
end

# app/controllers/application_controller.rb
# leave this for ActiveAdmin, and any other non-api routes
class ApplicationController < ActionController::Base
end
~~~


### How can I use this gem with Grape?

You may be interested in [GrapeTokenAuth](https://github.com/mcordell/grape_token_auth) or [GrapeDeviseTokenAuth](https://github.com/mcordell/grape_devise_token_auth).

### How can I use this gem with Solidus/Spree?

You may be interested in [solidus_devise_token_auth](https://github.com/skycocker/solidus_devise_token_auth).

### I already have a user, how can I add the new fields?

1. First, remove the migration generated by the following command`rails g devise_token_auth:install [USER_CLASS] [MOUNT_PATH]` and then:.
2. Create another fresh migration:

```ruby

  # create migration by running a command like this (where `User` is your USER_CLASS table):
  # `rails g migration AddTokensToUsers provider:string uid:string tokens:text`

  def up
    add_column :users, :provider, :string, null: false, default: 'email'
    add_column :users, :uid, :string, null: false, default: ''
    add_column :users, :tokens, :text

    # if your existing User model does not have an existing **encrypted_password** column uncomment below line.
    # add_column :users, :encrypted_password, :string, null: false, default: ''

    # if your existing User model does not have an existing **allow_password_change** column uncomment below line.
    # add_column :users, :allow_password_change, :boolean, default: false

    # the following will update your models so that when you run your migration

    # updates the user table immediately with the above defaults
    User.reset_column_information

    # finds all existing users and updates them.
    # if you change the default values above you'll also have to change them here below:
    User.find_each do |user|
      user.uid = user.email
      user.provider = 'email'
      user.save!
    end

    # to speed up lookups to these columns:
    add_index :users, [:uid, :provider], unique: true
  end

  def down
    # if you added **encrypted_password** above, add here to successfully rollback
    # if you added **allow_password_change** above, add here to successfully rollback
    remove_columns :users, :provider, :uid, :tokens
  end

```

### I want to add a new param for sign up and account update

[Override the controller](https://devise-token-auth.gitbook.io/devise-token-auth/usage/overrides#custom-controller-overrides) and describe the new parameters you want to add in the configure_permitted_parameters method.

When creating an account, add params under `sign_up`.

When updating your account, add params under `account_update`.

For example:

```ruby
class RegistrationsController < DeviseTokenAuth::RegistrationsController
  before_action :configure_permitted_parameters

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: %i(name))
    devise_parameter_sanitizer.permit(:account_update, keys: %i(name))
  end
end
```
