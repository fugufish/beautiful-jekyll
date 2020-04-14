---
layout: post
title: "ViewComponent and Stimulus Part 1: ViewComponent"
tags: [ruby, rails, vew_component, stimulus]
comments: true
---

All of the example code for this post can be found at 
[https://github.com/fugufish/sample_app_6th_ed](https://github.com/fugufish/sample_app_6th_ed)

Github's [ViewComponent](https://github.com/github/view_component) and [Stimulus](https://stimulusjs.org/) is one 
of those matches made in heaven. Like chocolate and peanut butter, caramel and sea salt, or Abbott and Costello, they 
seem to just fit together perfectly. In this series of posts I will detail what I have found to be an ideal setup for
developing Rails applications using these two complimentary libraries.

This first post focuses exlcusively on ViewComponent. We will add Stimulus in the next post in this series and
demonstrate how well these two libraries pair together.

## Getting Started with ViewComponent

ViewComponent is a Rails library by Github that somewhat mirrors the idea of 
[React Components](https://reactjs.org/docs/react-component.html) in the Rails environment. In the
 [Readme's](https://github.com/github/view_component/blob/master/README.md) own words, it is an evolution of the 
 presenter/decorator/view model pattern.
 
 Unlike traditional Rails decorators, ViewComponents come with their own Ruby Component class that is used to define
 the variables and methods that the view needs in order to render. This makes testing views dead simple, and strongly
 encourages reusable code.

For the rest of this series of posts I will be using an existing application to demonstrate how to use ViewComponent
and Stimulus effectively. In this case, I will be using 
[Mike Hartl's excellent sample application](https://github.com/fugufish/sample_app_6th_ed) from his 
[Rails Tutorial](https://www.railstutorial.org/), if you are new to Rails, I strongly suggest you check it out. First
step let's fork and clone the sample application so we have something interesting to start with.

### Installing ViewComponent

Add ViewComponent by adding it to the Gemfile.

```ruby
gem "view_component", "~> 2.2"
```

As a second step, we need to require it in `config/application.rb`

```ruby
require_relative 'boot'

require 'rails/all'

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

require "view_component/engine"
```

### Refactoring Partials into ViewComponents

The easiest way to figure out what pieces can be refactored into components it to look for partials. Almost any partial
could, and should probably be refactored into a ViewComponent. For more information on why you would favor a 
ViewComponent over a partial, check out ViewComponent's readme on the render speed and testability of ViewComponents.

#### Header and Footer

The first two partials we will refactor are the header and footer. Start with the header by creating the necessary 
component files. I am not using the generator in this case because I am breaking from ViewComponent convention
here  a bit. It'll be come apparent why later. We create 3 files:

**`app/components/shared/header/component.rb`:**

```ruby
class Shared::Header::Component < ViewComponent::Base
end
```

**`test/components/shared/header/component_test.rb`**
```ruby
require "test_helper"

class Shared::Header::ComponentTest < ViewComponent::TestCase
end

```

And finally a new template file: `app/components/shared/header/component.html.erb`

Now lets move the contents of `app/views/layouts/_header.html.erb` to
`app/components/shared/header/component_component.html.erb`. ViewComponents require local variables to explicitly added.
To do this we can take a look at the template and determine precisely what this component needs:

```erbruby
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", root_path, id: "logo" %>
    <nav>
      <div class="navbar-header">
        <button type="button" class="navbar-toggle collapsed"
                data-toggle="collapse"
                data-target="#bs-example-navbar-collapse-1"
                aria-expanded="false">
          <span class="sr-only">Toggle navigation</span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
        </button>
      </div>
      <ul class="nav navbar-nav navbar-right collapse navbar-collapse"
          id="bs-example-navbar-collapse-1">
        <li><%= link_to "Home", root_path %></li>
        <li><%= link_to "Help", help_path %></li>
        <% if logged_in? %>
          <li><%= link_to "Users", users_path %></li>
          <li class="dropdown">
            <a href="#" class="dropdown-toggle" data-toggle="dropdown">
              Account <b class="caret"></b>
            </a>
            <ul class="dropdown-menu">
              <li><%= link_to "Profile", current_user %></li>
              <li><%= link_to "Settings", edit_user_path(current_user) %></li>
              <li class="divider"></li>
              <li>
                <%= link_to "Log out", logout_path, method: :delete %>
              </li>
            </ul>
          </li>
        <% else %>
          <li><%= link_to "Log in", login_path %></li>
        <% end %>
      </ul>
    </nav>
  </div>
</header>
```

As you can see, it mostly needs `current_user` and to be able to determine if the user is logged in.

Let's add that to the component and update it to look like this:

```ruby
class Shared::Header::Component < ViewComponent::Base
  def initialize(current_user:, logged_in:)
    @current_user = current_user
    @logged_in = logged_in
  end

  private

  attr_reader :current_user

  def logged_in?
    @logged_in
  end
end
```

The template will have access now to the `current_user` and `logged_in` getters. With that in place, we can no alter the
render statement in the layout to render this component instead of the partial. Change line 16 from:

```erbruby
    <%= render 'layouts/header' %>
```

to:

```erbruby
    <%= render Shared::Header::Component.new(current_user: current_user, logged_in: logged_in?) %>
```

Now unless you have missed something, everything should render correctly and there should be no discernable change to
the application.

Finally, lets focus on a place where ViewComponent REALLY shines. Testing. We can test components individually with
very little effort. Let's add some test to cover the logged in verses the not logged in logic.

```erbruby
require "test_helper"

class Shared::Header::ComponentTest < ViewComponent::TestCase
  test "render component when logged in" do
    render_inline(Shared::Header::Component.new(current_user: users(:michael), logged_in: true))

    assert_link("Users")
    assert_link("Profile")
    assert_link("Log out")
    refute_link("Log in")
  end

  test "render component when logged out" do
    render_inline(render_inline(Shared::Header::Component.new(current_user: users(:michael), logged_in: false)))

    assert_link("Log in")
    refute_link("Log out")
  end
end
```

As you can see, testing the component is incredibly simple, and it is FAST.

And you're done! Now take what you learned and apply it to the `_footer.html.erb` partial. Next up, we will integrate
Stimulus!