---
layout: post
title: "ViewComponent and Stimulus Part 2: Stimulus"
day: 14
tags: [ruby, rails, vew_component, stimulus]
comments: true
---

Stimulus is a relatively new JavaScript framework built by the guys over at Basecamp, their opinionated answer to 
React/Angular/Vue. Initially released in January of 2018, and in their own words is a 
"[JavaScript framework of _modest_ ambitions](https://stimulusjs.org/handbook/introduction)". And it _is_ modest. With a
reference documentation of only 5 concise pages, it is definitely one of the smaller JavaScript frameworks out there,
and yet hits way above its weight class.

## Getting started with Stimulus

First we add the library through `yarn`:

```bash
yarn add stimulus
```

Next, lets update `javascaript/packs/application.js` to add Stimulus. Add the following lines:

```javascript
// requires all of the controllers in the app/components directory.
const componentContext = require.context("../../components/", true, /(.*)\/.+\.js$/);
application.load(definitionsFromContext(componentContext));
```

This will cause Stimulus to load all component controllers in the `app/components` directory. This effectively allows
us to sidecar the JavaScript that controls individual component behaviors with the components they control.

Finally, lets prepare Webpacker. First we'll update `config/webpack/environment.js` by adding the following just before
the export.

```javascript
environment.config.merge({
    resolve: {
        alias: {
            "javascript": path.join(__dirname, "..", "..",  "/app/javascript"),
            "channels": path.join(__dirname, "..", "..",  "/app/javascript/channels"),
        }
    }
})
```

These additions make it easier to access JavaScript in those directories without having to use longer path names.

And `config/webpacker.yml` replace:

```yaml
  resolved_paths: []
```

with

```yaml
  resolved_paths: ["app/components"]
```


Great! Now we are ready to get started integrating stimulus into our app.

## Going LIVE

I wanted to make the application a bit more dynamic. Make it fee more like a modern web application. Let's make this a 
live feed! Let's start by creating a feed component. This will replace the `shared/_feed.html.erb` partial that the app
currently uses:

**`app/components/feed/component.rb`**
```ruby
class Feed::Component < ViewComponent::Base
  def initialize(microposts:)
    @microposts = microposts
  end

  private

  attr_reader :feed_items
end

```

**`app/components/feed/component.html.erb`**
```html
<div data-controller="feed--component">
  <ol class="microposts" data-target="feed--component.micropostList">
    <%= render Feed::Micropost::Component.with_collection(microposts) %>
  </ol>
  <%= will_paginate microposts,
                    params: { controller: :static_pages, action: :home } %>
</div>
```

The `data-controller` and `data-target` attributes are used by Stimulus. The former indicates the name of the controller
that will be used for this component. The latter links a particular element to the component for easy access.

You may notice that I removed the `if @feed_items.any?` block from this. This is actually handled in the ViewComponent
by implementing the `#render?` method, we can update the component to look like this and get the same behavior:

```ruby
class Feed::Component < ViewComponent::Base
  def initialize(microposts:)
    @microposts = microposts
  end

  private

  attr_reader :feed_items
  
  def render?
    microposts.any?
  end
end
```

Now we are keeping the logic out of the template, and keeping it in the component. It makes the template far more
readable and easy to work with. Now we can update `app/views/static_pages/home.html.erb` replacing:

```html
<%= render 'shared/feed' %>
```

with

```html
  <%= render Feed::Component.new(feed_items: @feed_items) %>
```

But wait, we encounter an error:

![Render Error](/img/view-component-error-1.png)

This is because ViewComponent doesn't by default include all helpers outside of the Rails standard helpers. The
`#gravatar_url` method is in the `UsersHelper` which is not included in the component. Let's fix that. First,
let's migrate the micropost partial over to a ViewComponent:

**`app/components/feed/micropost/component.html.erb`**
```html
<li id="micropost-<%= micropost.id %>">
  <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
  <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
  <span class="content">
    <%= micropost.content %>
    <%= image_tag micropost.display_image if micropost.image.attached? %>
  </span>
  <span class="timestamp">
    Posted <%= time_ago_in_words(micropost.created_at) %> ago.
    <% if current_user?(micropost.user) %>
      <%= link_to "delete", micropost, method: :delete,
                  data: { confirm: "You sure?" } %>
    <% end %>
  </span>
</li>
```

**`app/components/feed/micropost/component.rb`**
```ruby
class Feed::Micropost::Component < ViewComponent::Base
  include UsersHelper
  include SessionsHelper

  with_collection_parameter :micropost

  def initialize(micropost:)
    @micropost = micropost
  end

  private

  attr_reader :micropost
end
```

Note that we are including the `UsersHelper` explicitly. This will give us access to the missing helper. Now update 
`` to look like this:

```html
<ol class="microposts">
  <%= render  Feed::Micropost::Component.with_collection(microposts) %>
</ol>
<%= will_paginate microposts,
                  params: { controller: :static_pages, action: :home } %>

```

We're back in business! But we're not done yet. Remember, we wanted to make this a live feed. For our next step, lets
wire up ActionCable to provide a channel for us. Start by running `rails g channel MicropostFeed` This is going to
create a few files for us:

```
Running via Spring preloader in process 23483
      invoke  test_unit
      create    test/channels/micropost_feed_channel_test.rb
      create  app/channels/micropost_feed_channel.rb
   identical  app/javascript/channels/index.js
   identical  app/javascript/channels/consumer.js
      create  app/javascript/channels/micropost_feed_channel.js
```

Let's start by setting up the channel by updating  `app/channels/application_cable/connection.rb` to look like this:

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def current_user
      User.find(request.session[:user_id])
    end
  end
end
```

This will set up action cable to be able to reference connections by the current user, and makes it so we can stream
data through action cable to the connected users. Next we update `app/channels/micropost_feed_channel.rb` to stream
for the current user:

```ruby
class MicropostFeedChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end
end
```
And finally, we update the `MicropostsController` at `app/controllers/microposts_controller.rb` by modifying the
`#create` action from this:

```ruby
def create
    @micropost = current_user.microposts.build(micropost_params)
    @micropost.image.attach(params[:micropost][:image])
    if @micropost.save
      flash[:success] = "Micropost created!"
      redirect_to root_url
    else
      @feed_items = current_user.feed.paginate(page: params[:page])
      render 'static_pages/home'
    end
  end
```

to this:

```ruby
if @micropost.save
  current_user.followers.each do |follower|
    MicropostFeedChannel.broadcast_to(
        follower,
        content: render_to_string(Feed::Micropost::Component.new(micropost: @micropost))
    )
  end
  flash[:success] = "Micropost created!"
  redirect_to root_url
else
  @feed_items = current_user.feed.paginate(page: params[:page])
  render 'static_pages/home'
end
```
This will effectively broadcast an update through ActionCable to all connected followers. By rendering 
`Feed::Micropost::Component` we can capture that broadcast message and add the rendered component directly to the feed.

Now we can add our component controller by creating the file `app/components/feed/component_controller.js`, and updating
it to look like this:

```javascript
import {Controller} from "stimulus"
import consumer from "channels/consumer";

export default class extends Controller {
    static targets = ["micropostList"]

    connect() {
        consumer.subscriptions.create("MicropostFeedChannel",
            {
                received: (data) => {
                    $(this.micropostListTarget).prepend(data.content)
                }
            })
    }

}
```

As you can see, the controller is pretty straight forward. We add `static targets = ["micropostList"]` to make the
controller aware that we are interested in that element. If you recall earlier we added the attribute
`data-target="feed--component.micropostList"`. This is the other side of that link.

The `connect()` method is called when the controller is connected to the element in the DOM. It is safe at this point to
assume that the component has been rendered to the DOM. In this case, once we are connected, we set up a subscription
to the `MicropostFeedChannel`.

When we receive an update from the channel, we can use the newly rendered component from our controller and prepend it
to our list.

Congrats! You now have live updates! In my next post I will demonstrate how to share data between controllers.

All of the example code for this post can be found at 
[https://github.com/fugufish/sample_app_6th_ed](https://github.com/fugufish/sample_app_6th_ed)

|| <!-- empty table header -->
|:--:| <!-- table header/body separator with center formatting -->
| [Part 1: ViewComponent](/2020-04-15-working-with-view-component-and-stimulus-part-1/) |