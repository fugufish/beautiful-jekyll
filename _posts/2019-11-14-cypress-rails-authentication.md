---
layout: post
title: Bypassing Authentication in Cypress in Rails with Devise
tags: [ruby, rails, cypress, testing, selenium, devise]
comments: true
---

For quite some time Capybara and Selenium have been standards for end to end testing in Ruby applications. On one 
particular project, this was quickly becoming a problem. Setting aside the issues of dealing with asynchronous requests,
and the massive overhead of Selenium, every time a browser updated in our test environment tests would break. Most of 
it seemed to do with slight differing sizes in the browser window that would force the application to render
differently.

## Enter Cypress
The first thing that attracted us to Cyrpess was it's consistency. In our evaluations we didn't seem to encounter the
same level of inconsistency in our tests as we did in Capybara. This might be attributed to flaws in our test strategy,
but for now I'm going to bury my head in the sand and blame it on Selenium, because Cypress just seemed to work.

There are plenty of blog posts on how to get Cypress working in Rails using the awesome [Cypress on Rails](https://github.com/shakacode/cypress-on-rails)
gem, and the readme should suffice for most.

There is very little however on how to deal with more complex problems in testing, such as authentication to the 
application without having to go through the manual authentication steps every time.

In Capybara, there are already tried and true solutions to this, and it's direct integration with the Rails environment.
Cypress has no such convenience, as it runs off of NodeJS, making tasks like this significantly more difficult. 
Fortunately, with a little elbow grease and a lot of poking around in the dark depths of Devise and Warden, it is not
hard to accomplish some of these tasks.

## Bypassing Authentication

Cypress on Rails integrations with the application by setting up some conditional endpoints that allow Cypress to
interact with the environment. These endpoints map to code that will be evaled within the Rails environment allowing
more complex tasks to be accomplished.

For authentication we use a [Cypress on Rails app command](https://github.com/shakacode/cypress-on-rails#example-of-using-app-commands)
which is effectively is a file containing any arbitrary ruby code that can be executed within the Rails environment. Our
app command is relatively simple:

```ruby
# test/cypress/app_commands/login_as.rb
module TestHelpers
  class LoginAs
    include Warden::Test::Helpers

    def self.run(username)
      new.run(username)
    end

    def run(username)
      u = User.find_by_name(username)
      login_as(u, scope: :user)
    end
  end
end

TestHelpers::LoginAs.run(command_options)
```

In this case, we are including `Warden::Test::Helpers` which is the same set of helpers used by Capybara in order to
accomplish the same task of bypassing authentication with devise. The class definition itself is pretty arbitrary, and
and only really used to give us access to the `login_as` method which is what we care about.

Now we can call this app command to bypass authentication as needed in our cypress tests:

```javascript
// test/cypress/integration/some_integration_test.js

describe('do something as a user', () => {
  beforeEach(() => {
    cy.app("login_as", "someusername")
  });

  it("does something cool", () => {
    // ....
  });

});
```

Now before every test it will pre-authenticate as the user. Pretty easy right? Lets quickly break down how this works.
When you call `cy.app("login_as", "someusername")` Cypress actually calls a provisional URL that is mounted only during 
test, passing `someusername` as a parameter. This gets set as the variable `command_options` which is globally available
in your `login_as.rb` script.

We build the `LoginAs` test helper class because we need to include `Warden::Test::Helpers` to get this to work. But 
because this script is executed as just that, a script. We need to run our test helper immediately.

And that's that. We've bypassed authentication and logged in as `someusername`.

## Conclusion
Cypress has changed how we approach testing. It is so incredibly easy to write tests using it, even for very complex 
scenarios we have made it mandatory that system tests be written for most front end changes. With Capybara, developing
these tests caused so much overhead that we only did it where them most major changes to our front end were being made.
But Cypress is a game changer in the field, modernizing how front end integration tests are written, and simplifying it
enough that it doesn't make sense to work it into our existing test requirements.


