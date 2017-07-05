---
date: 2016-04-20T00:00:00Z
title: Money Entry with Virtual Attributes
url: /2016/04/20/money_entry_with_virtual_attributes/
---

I recently discovered something called virtual attributes in Rails. They’re super cool and useful!

The basic premise is simple. You, as a developer, want to store some data in your database in a particular format but your user's want to enter it in a different format. Money is a great example. In Stripe’s API they store money as an integer with cent values.

```ruby
#five dollars and ninetynice cents
  599
```

This is a great way to store money in a database but it’s a terrible format if it’s part of your user interface. I’m working on a program where users need to enter an amount (in dollars) as they create a plan. Ideally the user should be able to enter almost anything and it’ll just work.

```ruby
# five bucks
5
5.00
5.0
5.000009
#all saved in the database as 500
```

To accomplish this goal we’re going to use a virtual attribute for the money entry (plus a sprinkle™ of javascript for the formatting). The idea behind using a virtual attribute is that we can use it to save the user’s money entry in memory, and then parse it to our database-friendly format.

## Tests
Let’s write a test.

```ruby
#spec/models/plan_spec.rb

describe 'money is saved as an integer with cents' do
  it 'tests value as integer' do
    plan = Plan.new(dollars: 19)
    expect(plan.amount_in_cents).to eq(1900)
  end

  it 'tests value as a fraction' do
    plan = Plan.new(dollars: 19.99)
    expect(plan.amount_in_cents).to eq(1999)
  end

  it 'tests value as a string' do
    plan = Plan.new(dollars: '19')
    expect(plan.amount_in_cents).to eq(1900)
  end

  it 'tests value as a string with a dollar sign' do
    plan = Plan.new(dollars: '$19.99')
    expect(plan.amount_in_cents).to eq(1999)
  end

  it 'tests value as a string with decimal' do
    plan = Plan.new(dollars: '19.99')
    expect(plan.amount_in_cents).to eq(1999)
  end
end
```

You’ll notice a couple things about this test. First we have two attributes. ```Plan.dollars``` and ```Plan.amount_in_cents```. The dollars attribute is going to be our virtual attribute. It won’t be saved to the database. It won’t need a migration. We’re going to define it right in the model. Alternatively, the amount_in_cents attribute **will** be saved to the database.

So first off we’ll add the amount_in_cents to our plan model.

```bash
rails g migration plan amount_in_cents:integer
```

Next, we’ll hop into the Plan model and define our virtual attribute.

```ruby
class Plan < ApplicationRecord
  #...
  attr_accessor :dollars

  def dollars=(value)
    @dollars = update(amount_in_cents: value.to_money.cents)
  end
end
```

Let me explain. I’ve defined getter and setter methods for the dollars attribute (```attr_accessor :dollars```). I’m using the [Money gem](https://github.com/RubyMoney/money) to parse the values into cents. I’ve overridden the setter method (```dollars=(value)```) which takes the user’s value to update our the amount_in_cents attribute. That value uses the money gem’s DSL to convert it to a money value(```.to_money```) and then into cents(```.cents```). Exactly what we need.

That works for the business logic of our app. Our tests should be passing so let’s move over to the user interface level.

## Forms

There’s lots of good javascript libraries out there to help with money parsing. I decided to use the [Autonumeric-rails gem](https://github.com/randoum/autonumeric-rails). With the gem installed and added to our javascript.js manifest, all we have to do is add the dollars field to our plan form. (Note that the amount_in_cents is left out from the form).

```ruby
#app/views/plans/_form.html.erb
#...
<div class="field">
  <%= f.label :dollars %>
  <%= f.text_field :dollars, data: {role: 'money', autonumeric: {aSign: '$'}} %>
</div>
```
There are [lots of options available](https://github.com/BobKnothe/autoNumeric#options) with how you want to format your dollars field. I’m keeping it pretty simple and I’m simply adding the dollar sign to the field.

That’s pretty much it.

It depends on your setup a bit to make sure that the autonumeric  javascript loads and updates itself as the user interacts with the form. I’m using Turbolinks5 so my js code looks something like this:

```javascript
document.addEventListener("turbolinks:load", function() {
  $(document).trigger('refresh_autonumeric');
});
```

That’s it! You should have a nicely formatted money entry field that saves to a Stripe-friendly database format. There are lots of possibilites to use virtual attributes. Plus it's even easier with [Rail's Attributes API](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/attributes.rb). [Documentation](http://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html) is a bit sparse right now but I'm looking forward to exploring this more!

Get in touch if you have any questions or comments!




