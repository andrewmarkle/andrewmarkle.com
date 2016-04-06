---
layout: post
title:  "Simple Multitenancy with Rails and Pundit"
---
I’m building an application where each user needs their own silo of data. They log in to the app and then can only see and edit their own stuff. There's a name for this architecture and it's called multitenancy.

Think of it like a condo. The app is the condo building. The users are the tennants. Each tennant only has access to their own apartment. Inside each apartment they fill it up with all kinds of stuff (data) and your job (as Chief Condo Builder) is to make sure each apartment has a lock on it.

There are two well-known ways to accomplish this goal:

### The Schema approach
This approach uses [PostgreSQL schemas](http://www.postgresql.org/docs/9.1/static/ddl-schemas.html) to separate  data into different databases for each user. We’re *not* going to be using this approach. For starters, it’s complicated to set up but the main reason is because of [performance](https://devcenter.heroku.com/articles/heroku-postgresql#multiple-schemas). From the sounds of it, you’ll have big scalling problems down the road. We’re going to use an alternate approach.

### The scoping approach
This way uses scopes to isolate data. Every user shares the same database but their information is scoped to a specific column. It’s fast, performant, and simple to set up, but there is one big gotcha. When scoping data you need to make double sure that your scopes are working and you’re not accidentally showing the wrong data to the wrong user. I’ll show you a way to help catch some mistakes, but ultimately it’s up to you to make sure this is working. Good test coverage is essential.

## Setup
First of all we’re using Rails 5, PostgreSQL (you’ll see why shortly), Devise for authentication, and Pundit for authorization and help with scoping. I’m going to assume that you know how to get a Rails app set up with Devise and Pundit. We’re *not* going to be using subdomains so everything is out-of-the-box standard setup.

### UUIDs instead of subdomains
Most multitenant tutorials use subdomains to help scope the data. Basecamp (version 1) did this back in the day where each company would have their own subdomain (http://mycompany.basecamp.com). We’re *not* going to be doing that. It’s a bit complicated to set up, more expensive to operate (wildcard SSL certificates cost more), and a bit of a pain to use in development. None of those problems can’t be overcome but why don’t we keep it simple instead? Basecamp 2 and 3 seem to use UIDs instead of subdomains (https://3.basecamp.com/3313687/projects/603426). (I assume that those numbers in the url are a randomly generated UID?) Anyway, this approach is very simple and it works great.

### Setting up UUIDs in Rails
If we stick with the standard IDs that Rails gives us, we’re going to run into a potential problem. As we create models and add them in our app we’re going to get urls like this: http://yourapp.com/client/3. The problem is that the ID numbers won’t match up with the user’s experience. This Client (with id: 3) will be the third Client created, regardless of which user created it. This isn’t that big of a deal (if we’re scoping things correctly it won’t matter if a user types in an ID that doesn’t belong to them) but it may be a bit weird or confusing. My thinking is: if this number is going to represent a random abstraction, let’s go all out and really make it truly random. (If this doesn’t matter to you, feel free to skip this step!)

This is where using PostgreSQL as the database helps us out. By using UUIDs instead of IDs we’re going to generate urls like so: http://yourapp.com/client/9161d01d-b063-479f-80e3-83ee1906ca82. The added bonus is that UUIDs are magically better performing than regular IDs and saves space in your database. Amazing!

Setting up UUIDs is pretty simple but we need to use one of PostgreSQL extension’s [pgcrypto or uuid-ossp](http://edgeguides.rubyonrails.org/active_record_postgresql.html#uuid).

First, let’s enable the extension.

```ruby
rails g migration enable_uuid_extension
```

We’ll use the uuid-ossp extension in this example.

```ruby
class EnableUuidExtension < ActiveRecord::Migration[5.0]
  def change
    enable_extension 'uuid-ossp'
  end
end
```

Note: If you’re adding this to a Rails app that already has some migrations, this extension **needs to be your first migration** if it’s going to work properly. (I had to manually change the migration file number and completely regenerate my database from scratch for this to work).

UUIDs won’t be enabled yet so, for existing migrations, I found it easiest to alter my migration files and rebuild the database. Your mileage will vary if you have this in production!

```ruby
class DeviseCreateUsers < ActiveRecord::Migration
  def change
    create_table :users, id: :uuid do |t|
    ## stuff omitted
    ## ...
  end
end
```

Bonus: In Rails 5 we can use the purge command to level our database and start over.
  rake db:purge db:create db:migrate

Okay, now that we’ve got that set up it would be nice if we had a way to add :uuids to every new model we created. Once again, Rails has our back. We’ll add a line to the config/application file:

```ruby
# config/application.rb
class Application < Rails::Application
  config.generators do |generate|
    generate.orm :active_record, primary_key_type: :uuid
  end
end
```
Now when we generate a new migration, the id will be set to a uuid as default. Nice.

### Everything belongs_to a company
In my app I’m going to be using a Company model to scope the user’s data. (I’m assuming that you’ve already [installed and set up Devise](http://devise.plataformatec.com.br/#controller-filters-and-helpers#getting-started)).

So let’s make that Company model.

```bash
rails g model Company name
```

(If you’ve got everything set up properly the Company id will now be setup as a uuid).

In my app I’m going to eventually add the ability for multiple users to belong_to the same company. So in our Company model, we’ll add the line:

```bash
has_many :users
```

Then we’ll add the company_id to our User model (this will be the column that we scope from).

```bash
rails g migration add_company_to_users company:references
```

There is a bit of a bug in Rails as of this writing and you’ll get an error if you migrate this. You need to manually add the uuid type to the migration for it to work.

```ruby
def change
  add_reference :users, :company, foreign_key: true, type: :uuid
end
```
That should work. Double check that the User model has:
  belongs_to :company
and you’re all set to migrate. (Feel free to repeat this process for any other models that you want scoped to the company. All of your scoped models should have a belongs_to :company. The Company, in turn, should have a has_many method for each scoped model that belongs to it).

This isn’t working yet though. We still need to set up the scopes. For that, we’ll turn to [Pundit](https://github.com/elabs/pundit) for some help. But first, let’s write a test to make sure we’re on the right path.

### Testing scopes
I’m writing an integration test here using RSpec and Capybara. For this example, I’ll use the Plan model to test this out. I won’t go through every step of this but if you want to TDD along, go for it! I like to use fixtures for my integration tests to keep the test fast. They’re setup like so:

```yml
# spec/fixtures/plans.yml
plan_a:
  company: company_a
  name: This plan belongs to Company A

plan_b:
  company: company_b
  name: This plan belongs to Company B

# spec/fixtures/companies.yml
company_a:
  name: Company A

company_b:
  name: Company B

# spec/fixtures/users.yml
user_a:
  email: user_a@example.com
  encrypted_password: <%= User.new.send(:password_digest, "password") %>
  company: company_a

user_b:
  email: user_b@example.com
  encrypted_password: <%= User.new.send(:password_digest, "password") %>
  company: company_b
```

Next we’ll write out our spec.

```ruby
# spec/features/plan_scoping_spec.rb
require "rails_helper"

feature "Plan scoping" do
  fixtures :all

  scenario "display's only User A's records" do
    sign_in_with "user_a@example.com", "password"
    visit plans_url
    expect(page).to have_content "This plan belongs to Company A"
    expect(page).not_to have_content "This plan belongs to Company B"
  end

  scenario "display's only user B's records" do
    sign_in_with "user_b@example.com", "password"
    visit plans_url
    expect(page).to have_content "This plan belongs to Company B"
    expect(page).not_to have_content "This plan belongs to Company A"
  end
end

def sign_in_with(email, password)
  visit new_user_session_path
  fill_in "user_email", with: email
  fill_in "user_password", with: password
  click_button "Sign in"
end
```

This spec signs in with one user, goes to the Plan#index page, and sees if it prints out the plan’s name. It should only show the plan from the user that is signed in.

For this tutorial I’m going to skip creating a plan. You can figure that stuff out. Your stock scaffold will work for this example as long as you’re showing the plan.name in the #index. Also, make sure that your Plan belongs_to :company, and your Company has_many :plans. Let’s skip ahead, assuming we’ve set up the plan, and our tests are failing because we’re showing both Plan A’s name and Plan B’s name.

### Scoping our data using Pundit
He’s where Pundit steps in. Pundit is a very simple gem that is built for authorization. It also has a little feature to scope data (which is what we’re going to leverage). To get started, add the gem to your gemfile and bundle install.

We’ll include Pundit in our application controller.

```ruby
  class ApplicationController < ActionController::Base
    include Pundit
    protect_from_forgery
  end
```

And then install the boilerplate policy.

```bash
rails g pundit:install
```

This will install a file in app/policies/application_policy.rb that all of our policies will inherit from. I won’t go into too much detail explaining how to setup your policies. There are [many](https://gorails.com/episodes/authorization-with-pundit) [great resources](http://www.sitepoint.com/straightforward-rails-authorization-with-pundit/) out there, including [Pundit’s documentation](https://github.com/elabs/pundit) which is excellent. The part that we’re interested in is the Scope class at the bottom of the application_policy:

````ruby
...
class Scope
  attr_reader :user, :scope

  def initialize(user, scope)
    @user = user
    @scope = scope
  end

  def resolve
    scope
  end
end
````

How Pundit works is it passes in the current_user as the user argument. Then it passes in the scope argument (which I’ll explain soon). We’ll alter the resolve method to this:

```ruby
  def resolve
    scope.joins(:company).where(company_id: user.company_id)
  end
```

What this method does is look for the company_id and makes sure it matches up with the current_user’s company_id. So, in our PlansController we can implement this Pundit method like this:

```ruby
def index
  @plans = PlanPolicy::Scope.new(current_user, Plan).resolve
end
```

It passes in the Plan model as the scope argument. Then, by calling resolve on it, it’s the same as:

```ruby
Plan.joins(:company).where(company_id: current_user.company_id)
```

Pundit provides a convenient shorthand method which does the same thing.

```ruby
def index
  @plans = policy_scope(Plan)
end
```

If we set everything up right, our test should now be passing. Our Plan model’s #index should correctly be scoped and only showing the Plans that belong to our user’s company. We’re still not done though. Our other actions aren’t scoped yet and it would be nice if we had a way for the code to throw an error if any scopes haven’t yet been implemented.

### Implementing our scope across the application
Pundit has a method (which we’ll add to our application_controller) that will throw an error if we’re not using our new scope.

```ruby
class ApplicationController < ActionController::Base
  ...
  after_action :verify_policy_scoped, unless: :devise_controller?
  ...
end
```

(I’ve added the unless: :devise_controller? which will skip any Devise controllers. If we scope Devised we won’t be able to sign in or sign up!)

Now, assuming you have some basic integration tests (creating, updating, destroying our Plan) your tests should blow up (which is what we want!) Pundit will throw a Pundit::PolicyScopingNotPerformedError every time we initialize a model in a controller. This is great, because now every time we add a new model/controller Pundit will complain if we don’t set the scope. It’s a nice little backup that will hopefully prevent us from scoping things wrong or not scoping them at all.

To fix our PlansController is very simple. We just add our policy_scope method, every time we initialize the Plan.

```ruby
# replace Plan with policy_scope(Plan) everywhere
...
def create
  @plan = policy_scope(Plan).new(plan_params)
  ...
end
...
def set_plan
  @plan = policy_scope(Plan).find_by(id: params[:id])
end
```

By adding this everywhere, the company_id will automatically be entered (and match the current_user) every time we create a new Plan, update a plan, etc. This is very helpful!

### Cleaning up
No doubt there are some controllers that you *don’t* want to be scoped. When a user creates her Company, for example, it can’t be scoped to a company_id that doesn’t yet exist. Our landing page doesn’t need a scope. Neither will any controllers that belong_to a scoped model (say if we create a Customer that belongs_to a Plan).

This is very easy to solve by adding a skip_after_action in each controller that doesn’t need a scope.

```ruby
class CompanyController < ApplicationController
  skip_after_action :verify_policy_scoped, only: [:new, :create]
  ...
end
```

(I like to make sure this skip_after_action is only set for the actions I’m using. That way if I add another action in the future, Pundit will throw an error and I’ll have to decide if I should skip it or scope it).

### Wrapping up
That’s basically how you set up a multitenant app using Pundit. I skipped over a few things though. The main gotcha is making sure you have good test coverage. Integration tests that test all the CRUD actions of your models will go a long way to preventing errors. Also we’ve only specifically tested the scope for the index on our Plan model. In my app I also like to test the scope for the show action as well. This is repetitive, but it needs to be done for every model that you’re scoping.

Hopefully that makes sense! If I missed something or you have any questions feel free to get in touch!

-------------------

#### RESOURCES
A lot of thanks goes out to Jon McCartie for adding UUID generators in Rails and explaining how to use them. See his post: [Default UUIDs in Rails](http://blog.mccartie.com/2015/10/20/default-uuid's-in-rails.html) for more info.

A huge thanks to Ryan Bigg for his book [Multitenancy with Rails](https://leanpub.com/multi-tenancy-rails-2). Highly recommended if you want to dig deep into the world of multitenancy.
