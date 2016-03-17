# Rails Model Methods, Validations, and Callbacks

## Learning Objectives

- Explain how methods defined on a model help practice seperation of concerns
- Defend how business logic and custom queries can be encapsulated in models definitions
- Utilize Rails validations to protect application's state from bad data
- Define how Rails implements callbacks in relation to data persistence

## Framing (15 / 15)

**Q**:  So far in the Rails apps we have been building, where does the majority of our logic live?

>  A: Our controllers

**Q**: Where should it be?

> A: Our models

**Why?**

In short, to practice separation of concerns.

In Rails, your controller is the coordinator that takes parameters and returns HTML, JSON or some other format. It shouldn’t be making decisions above that.

Let's review the model's role and responsibility in Rails' MVC architecture.

Through Active Record, our models encapsulate the data and domain logic of our application, by using Ruby classes as wrappers for the records in our database.

With this, business logic used in models can be re-used in multiple different routes, or in web-service APIs, or a command-line scripts. It can also be easily unit tested.

In comparison, the same business logic in the controller can only be used via the web interface.

Lets look at an example.

Say we are working on a blog, and a couple of our business rules are:

- If a user is logged in, and they visit the `index` for `posts`, they should only see all of their posts.
- If a user is not logged in and they visit the `index` page for `posts`, they should see all posts.

An app where all the logic lives in the `posts` controller might look similar to:

```ruby
class PostsController < ApplicationController
  def index
    @user = User.find(session[:user_id]) || User.new
    if @user
      if @user.signed_in?
        @posts = @user.posts
      else
        @posts = Post.all
      end
    else
      @posts = Post.all
    end
  end
end
```

**Q** Now this is just pseudo-code, but can we identify any problems with the current implementation?
> A: Our code is not Dry, and is not practicing seperation of concerns

By moving this business logic to a method on the `Post` model, we could re-write our `index` action using something like:

```ruby
class PostsController < ApplicationController
  def index
    @user = User.find(session[:user_id]) || User.new
    @posts = Post.find_all_posts(@user)
  end
```
The implementation for our custom method, `find_all_posts` might look something like this:

```ruby
class Post < ActiveRecord::Base
  belongs_to :user

  def self.find_all_posts user
    if user && user.signed_in?
      posts = user.posts
    else
      post = self.all
  end
end
```
**Q** Important to note, what type of method did we just write?
> A: A class method

FYI, We can also add custom instance methods to our models.

### Fat Models, Skinny Controllers

This design pattern has developed such a strong following in the Rails Community, that it even has it's own catch phrase: "**Fat Models, Skinny Controllers**"

> The motto “Fat Model, Skinny Controller” refers to how the M and C parts of MVC ideally work together. Namely, any non-response-related logic should go in the model, ideally in a nice, testable method. Meanwhile, the “skinny” controller is simply a nice interface between the view and model.

We as Rails developers looking to follow best practices, want to stay on top of recent industry trends. Thankfully Sandi Metz, renowned Rubyist and author of [POODR](http://www.poodr.com/), has come up with a set of rules to help serve as guidelines when trying to write clean, DRY, code that practices seperation of concerns.

> **Watch** [Matz's talk](https://www.youtube.com/watch?v=npOGOmkxuio) introducing her rules in Barcelona Ruby Conference 2013. The rules start around the 8 minute mark.

Matz's Rules for Writing Object-Oriented Code

1. **Classes should be no more than 100 lines long**.
2. **Ruby methods should be no more than 5 lines long**.
3. **No Ruby method should have more than 4 parameters**.
4. **A Rails controller action may pass no more than 1 instance variable per view**
5. **A controller action may reference no more than 2 class names per action**

>  **Note**: In practice, you might find yourselves breaking these rules from time to time, but the important thing is if you can justify it, especially to your pair or team.  These are not meant to be set in stone, more as constraints to get you to ask the proper questions during development.

### (You-Do) Garnet Scavenger Hunt (10 / 25)

Incorporating what you just learned about best practices, take some time reviewing the code base for [Garnet](http://github.com/ga-dc/garnet), to see how our team take's advantage of the power of model method in a production Rails app.

In particular, look at:

- The models code, especially the [User model](https://github.com/ga-dc/garnet/blob/master/app/models/user.rb) in `app/models/user.rb`
- Look at some of the code in the controller files in `app/controllers`

Some things to consider:

- What are some of the general themes of the purpose of these model methods?
-  Are there any aggregious examples of breaking Matz's rules?
- Can you think of any opportunities to refactor, by moving current controller or view code to the model?

### (Exercise) Rails Contacts

We will be working in groups to take an existing app with "fat controllers", and see how we could refactor by moving the business logic to the model.

#### Setup (5 / 30)

Go ahead and fork/clone this [repo](https://github.com/ga-wdi-exercises/rails-contacts) and run through the `Getting Stared` [section](https://github.com/ga-wdi-exercises/rails-contacts#getting-started).

`#slackTheBack` if you have trouble getting setup.

#### Code Review (10 / 40)

Spend some time with your group looking through the code base, and familiarizing yourself with whats there. **YOU SHOULD NOT BE WRITING ANY NEW CODE!**

- Run the app locally and play around in the browser. Think about what's happening behind the scenes with each request
  - **Hint**: What's changes in the `url`?
- **Hint:** Pay attention to what's already in `people_controller.rb`
- **Hint:** Look at your app's `schema`, and see what data is being generated from `seeds.rb`
- Run the app's tests with `$ rspec`!

### Break (10 / 50)

#### Refactor (35 / 85)

Now start getting those tests to pass!  

Read through the notes for [each method](https://github.com/ga-wdi-exercises/rails-contacts#get-the-specs-to-pass-for) you're tasked with implementing

If you finish up early, be sure to [check out](https://github.com/ga-wdi-exercises/rails-contacts#bonus) the **bonus**

### Rails Validations

With Active Record, Rails allows us to validate the state of a model before it gets written into the database. 

#### Doc Dive (5 / 90)

Read sections 1-1.3 and 2 in the Rails Guides official [Documentation](http://guides.rubyonrails.org/active_record_basics.html#validations)

#### Turn and Talk (5 / 95)

1. Why use validations?
2. What types of validations are there?
3. When are validations triggered?

#### Adding Validations (15 / 110)

Validation is a very important issue to consider when persisting to the database. In Active Record, the methods `save` and `update` take validations into account when run.

When invoked, both methods  return `false` when validation fails and they didn't actually perform any operation on the database.

Both `save` and `update` have a stricter implementation, with their bang counterparts ( `save!` and `update!`), which raise the exception `ActiveRecord::RecordInvalid` if validation fails.

 A quick example to illustrate:

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true
end

user = User.new
user.save  # => false
user.save! # => ActiveRecord::RecordInvalid: Validation failed: Name can't be blank
```

Another great thing about using validations in our Rails apps is that we can use the error messages generated by invalid data persistence to then display helpful notifications to the user.

Sticking with the example above:

``` ruby
class User < ActiveRecord::Base
  validates :name, presence: true
end

user = User.new
user.save  # => false
user.save! # => ActiveRecord::RecordInvalid: Validation failed: Name can't be blank

# we can access those messages programatically
user.errors.full_messages.first # => "Name can't be blank"
```

Lets look at how we could add validations to our `Contacts` app.

### (You-Do): Implement Validations in Contacts (5 / 115)

Use validations to satisfy our domain's logic as follows:

- No person can be created with blank `first_name`, `last_name`, and `email` attributes
- No two people can share the same email address

**Bonus**

Write and implement a [custom validation](http://guides.rubyonrails.org/active_record_validations.html#performing-custom-validations) that checks for the validity of an email address

- An email address must include an `@` sign, followed by a domain

### Break (10 / 125)

### Callbacks (10 / 135)

Active Record callbacks allow us to attach code to certain events in the life-cycle of our models.

This lets us add behavior to our models by transparently executing code when those events occur, like whenever we create a new record, update it, destroy it.

Callbacks allows us to trigger logic before or after an alteration of an object's state.

For example:
```ruby
class User < ActiveRecord::Base
  validates :username, :email, presence: true

  before_create :generate_name

  def generate_name
    self.name = username.capitalize if name.blank?
  end
end
```

The most commonly used ones are:

- Before Save
- After Create
- After Save
- Before Validation
- After Validation

> [Callback Docs](http://guides.rubyonrails.org/active_record_callbacks.html)

#### (You-Do): Implement Callbacks in Contacts (15 / 150)

Create instance methods that will run before each instance saves to:

- Generate a domain name for a person if currently blank
- Sanitizes any email entered so that its always in a uniform case

To do this, make the pending tests active in `person_spec.rb` .

See further instructions [here](https://github.com/ga-wdi-exercises/rails-contacts#bonus)

## Homework

None. But it might be a good idea to incorporate custom model methods to help implement any further features for Scribble

## Sample Quiz Questions

1. Describe the philosophy behind the phrase "fat models, skinny controllers"?
2. What are validations in Rails? Provide an example.
3. List 4 helpful Active Record Callbacks and describe their purpose

## Resources

- [Sandy Metz Rules For Developers](https://robots.thoughtbot.com/sandi-metz-rules-for-developers)
- [Ruby on Rails Best Practices](http://www.sitepoint.com/10-ruby-on-rails-best-practices/)
- [Simple Rules to Good OO in Rails](http://thunderboltlabs.com/blog/2013/01/23/5-simple-rules-to-good-oo-in-rails/)
- [Rails Validations](http://guides.rubyonrails.org/active_record_validations.html#validations-overview)
- [Active Record Callbacks](http://guides.rubyonrails.org/active_record_callbacks.html)
