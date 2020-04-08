---
layout: post
title: "Building an application with Github's ViewComponent and Stimulus: Part 1 (Getting Started)"
tags: [ruby, rails, vew_component, stimulus]
comments: true
---

Github's [ViewComponent](https://github.com/github/view_component) and [Stimulus](https://stimulusjs.org/) is one 
of those matches made in heaven. Like chocolate and peanut butter, caramel and sea salt, or Abbott and Costello, they 
seem to just fit together perfectly. In this series of posts I will detail what I have found to be an ideal setup for
developing Rails applications using these two complimentary libraries.

## Stimulus

Stimulus is a relatively new JavaScript framework built by the guys over at Basecamp, their opinionated answer to 
React/Angular/Vue. Initially released in January of 2018, and in their own words is a 
"[JavaScript framework of _modest_ ambitions](https://stimulusjs.org/handbook/introduction)". And it _is_ modest. With a
reference documentation of only 5 concise pages, it is definitely one of the smaller JavaScript frameworks out there,
and yet hits way above its weight class.

Unlike frameworks such as React and Vue, Stimulus renders on top of the DOM, it doesn't implement a virtual or shadow 
dom. That means it works seamlessly with your existing HTML, and requires very little modification to get it to work
with existing applications.

## ViewComponent

ViewComponent is a Rails library by Github that somewhat mirrors the idea of 
[React Components](https://reactjs.org/docs/react-component.html) in the Rails environment. In the
 [Readme's](https://github.com/github/view_component/blob/master/README.md) own words, it is an evolution of the 
 presenter/decorator/view model pattern.
 
 Unlike traditional Rails decorators, ViewComponents come with their own Ruby Component class that is used to define
 the variables and methods that the view needs in order to render. This makes testing views dead simple, and strongly
 encourages reusable code.
 
## The Mix

Mixing Stimulus and ViewComponent seems like a no-brainer. ViewComponent renders the Component, and Stimulus handles the
JavasScript interactions of that component. With modern Rails stacks (>= 5) this means that all of the code for any
particular component can exist in a single place, much like it would in a React application.

## Getting Started

I am making the assumption here that at this point you are already using an application that utilizes 
[Webpacker](https://github.com/rails/webpacker), and is on at least Rails 5.

### Install ViewComponent

Start by installing ViewComponent. In your `Gemfile` add `gem ViewComponent`, and in your `config/application.rb` file
add `require view_component/engine`. That's it!

### Install StimulusJS

On the command line run `yarn add stimulusjs` to add Stimulus to your `package.json`. Next you will need to update your
`javascript/packs/application.js` to account for stimulus controllers:

```javascript
import { Application } from "stimulus"
import { definitionsFromContext } from "stimulus/webpack-helpers"

//...
const application = Application.start();
const componentContext = require.context("../../components/", true, /(.*)\/.+\.js$/);
application.load(definitionsFromContext(componentContext));

const controllerContext = require.context("../controllers", true, /(.*)\/.+\.js$/);
application.load(definitionsFromContext(controllerContext));
//..
```

## ViewComponents

You're ready to build your first ViewComponent. Let's say you have an existing application that relies on Semantic UI
 and you wish to refactor things into something a little more manageable. I use this example because it is easy to find 
 the defining lines of what a component is. Let's pick off something simple. Like an accordion component.

First we generate the view component

```bash
rails g view_component Bootstrap::Accordion::Component
```

I am straying here from ViewComponent's existing convention, but there is a reason for that that will become apparent
later so bear with me. This will build the following:

```
rails-root
 - app
   - components
     - semantic
       - accordion
         - component.html.erb
         - component.rb
```

Semantic requires running the jQuery directive `$(element).accordion()` to instantiate the accordion's JavaScript. We
can do this in a stimulus controller. Create a new Stimulus controller in the accordion component directory:

*app/components/semantic/accordion/component_controller.js*
```javascript
import {Controller} from "stimulus"

export default class extends Controller {
    connect() {
        $(this.element).accordion()
    }
}
```

This explicit way of setting up a component sets expectations for developers, they always know where to find the
JavaScript related to any given component. `this.element` refers to the element the controller is attached to 
(see below).

Now lets build out our component:

*app/components/semantic/accordion/component.html.erb*
```html
<div class="ui styled fluid accordion" data-controller="semantic--accordion--component">
  <%= content %>
</div>
```

In the template you may note the `data-controller` attribute. The controller is specifically which Stimulus controller
that will be attached to the element. Stimulus namespaces it's controllers using `--` depending on the directory
structure. Therefore `app/components/semantic/accordion/component_controller.js` translates to 
`semantic--accordion--component`.

All that is left is to render this new component in a view.

*app/views/some_view/index.html.erb*
```html
<%= render(Semantic::Accordion::Component.new) %>
```

That's it. This is of course a very simple example. In the next post we will discuss something a little bit more 
complicated.

