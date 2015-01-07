---
layout: post
title: "Using I18n and Draper to Render Database Attributes"
date: 2011-11-04 20:27:04 -0800
date_formatted: November 4, 2011
comments: true
categories:
published: true
---


Using I18n and Draper to Render Database Attributes

TL;DR: Check out my additions to ApplicationDecorator [in this gist](https://gist.github.com/1338134).

Update: This has been released as a gem: [Olson](https://github.com/carnesmedia/olson).

When my models have an attribute that matters to the code (like `Admin#role` or `User#status`), I like to store the value as a string that makes sense as an identifier. For example, `User#status` might be `'active'` or `'awaiting_approval'`. However, when it comes time to render the admin's role or the users status in the view, we want to show 'Awaiting approval' instead of 'awaiting_approval'. Another example of this sort of thing is the `#type` attribute for STI.

Ok, this isn’t too hard, we can just use `#humanize`. But, here’s what happens: <!--More-->

``` erb In some views
# Show a user-friendly version of our identifier
Status: <%= current_user.status.humanize %>

# Now we need to customize some of them, use I18n
Status: <%= t current_user.status, default: current_user.status.humanize %>

# But that's polluting our I18n namespace
Status: <%= t :"user.status.#{ current_user.status }", default: current_user.status.humanize %>

# Ok, this is getting out of hand, lets refactor this to the model
Status: <%= current_user.status_string %>

# This breaks down when you need to render the select field to edit this user
<%= form.input :status, collection: User::STATUSES.map { |s| [User.new(status: s).status_string, s] } %>

# So, how 'bout a helper
Status: <%= humanize_with_i18n current_user.status, %(user status) %>

# Not too bad
<%= form.input :status, collection: User::STATUSES.map { |s| [humanize_with_i18n(s), s] } %>

# Ah, this is a bit better
<%= form.input :status, collection: user_roles_for_select %>
```

``` ruby Model or helper code
class User < ActiveRecord::Base
  # ...
  def status_string
    I18n.t status, scope: %w(user status), default: status.humanize
  end

  def another_thing_string
    I18n.t ...
  # ...
end

# Or something like:

module UserHelper
  def humanize_with_i18n(string, scope = [])
    I18n.t string, scope: scope, default: string.humanize
  end

  # The method prefix tells me that this should be in an object
  # But it doesn't belong in our model, does it?
  def user_roles_for_select
    User::STATUSES.map { |s| [humanize_with_i18n(s), s] }
  end
end
```

Ok, let's be fair. All of these solutions are actually quite fine. In most cases Ya Ain't Gonna Need anything more complicated. The helper version handles most situations just fine.

However, after a bunch of this I tend to end up with a bunch of methods in my model that seem to be somewhat presentation related, and/or methods in my helper that seem like they belong to an object and not in the "global" view namespace.

### Enter decorators

A decorator (or presenter) is an object that holds the presentation logic for a model, so that the model can stick to the business logic. I've been using a great gem called [Draper](https://github.com/jcasimir/draper). I won't go into too much detail about how to use Draper (check out the [Github readme](https://github.com/jcasimir/draper) or [Railscast](http://railscasts.com/episodes/286-draper)).

Here's how you would implement the above pattern with Draper:

``` ruby user_decorator.rb
class UserDecorator < ApplicationDecorator
  decorates :user

  def status
    # self.model gives us the User
    self.class.humanize_with_i18n(:status, model.status)
  end

  # This could be moved to ApplicationDecorator (or ApplicationHelper),
  # but is shown here for simplicity.
  def self.humanize_with_i18n(attribute, value)
    I18n.t value, scope: ['user', attribute], default: value
  end

  def self.status_options
    User::STATUSES.map { |s| [humanize_with_i18n(:status, s), s] }
  end
end
```

Then, in our view:

``` erb
Status: <%= current_user.decorator.status %>

<%= form.input :status, collection: UserDecorator.status_options %>
```

``` ruby Bonus
module ApplicationHelper

  def current_user
    super.decorator
  end
end
```

### My Abstractions

And now the reason for this post. I find that I use this pattern frequently, so I generalized it to `ApplicationDecorator`. It adds a class method `ApplicationDecorator.humanizes` that can be used in each decorator to define attributes that need automatic humanization.

The full source can be found here: https://gist.github.com/1338134.

Here's how you would use it:

``` ruby user_decorator.rb
class UserDecorator < ApplicationDecorator
  decorates :user
  humanizes :status

  def self.status_options
    options_for_select_with_i18n :status, model_class::STATUS_OPTIONS
  end
end
```

``` erb And in a view
Status: <%= current_user.status %>

<%= form.input :status, collection: UserDecorator.status_options %>
```

### To Conclude

I like this because each layer is really simple and really focuses on only what it needs to.

The view doesn't have to know that that data is not user-friendly.
The model isn't polluted with methods designed for the view.
There isn't much complexity or black-magic to make this abstraction simple.
If this pattern works out in my current project I will probably pull this out into a gem. Would anyone else find this useful? If I do I'll be looking for name suggestions...

