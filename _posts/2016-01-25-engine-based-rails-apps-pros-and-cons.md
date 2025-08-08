---
layout: post
title: "Engine-based Rails Apps: Pros & Cons"
author: "Ben Roesch"
---
Within the Rails community, there's a growing number of applications
that are being built as a collection of [Rails engines][1], rather than
a single, monolithic application. At [Cultivate Labs][2], we've been
using this pattern to build [Cultivate Forecasts][3], a fantasy money
[prediction market][4]. Along the way, we learned a great deal about
building applications using this pattern, so I wanted to share some of
our takeaways.

## Pros

### Code Organization

In my opinion, the biggest benefit inherent in the engine-based Rails
app (EBRA) pattern is that it forces you to organize and separate your
code. By doing so, it forces you to think through where to put each
piece of code that you write.

A great example of this is the user model. In many Rails apps, the user
model ends up being a catch-all place for any code looking for a home,
resulting in a long, overly complex chunk of code. In practice, this
frequently results in the user model having structural knowledge of
almost every other piece of the system. Based on [ The Law of
Demeter][5], we know that this spreading of structural knowledge results
in an application that is tightly coupled and difficult to maintain --
an environment that makes future development slow and error-prone.

If you set up your dependencies properly in an EBRA, then your user
model doesn't "know" about most of the other components (n.b. component
== engine, here) in the system. This forces you to put methods/code in
their proper place, rather than just dumping them in a user model.

For example, say we want a method that returns a list of questions where
the user has made a prediction. Sometimes you'll see something like:

    
    class User < ActiveRecord::Base
      def questions_where_user_has_made_prediction
        Question.joins(:predictions).where(predictions: { user_id: id })
      end
    end

This might seem innocuous, but (1) these tend to pile up en-masse in the
user model and (2) we're introducing knowledge of the question concept
into the user model.

When building an EBRA, the accounts/users component woudn't "know" about
questions or predictions. This forces us to put this code in a more
appropriate place:

    
    class Question < ActiveRecord::Base
      def self.with_prediction_by_user(user)
        joins(:predictions).where(predictions: { user_id: user.id })
      end
    end

### Decreased Conflicts

Another benefit that we noticed along the way was a marked decrease in
the number of merge conflicts we were experiencing. I think that there
are two main reasons for this (which are, in a way, kind of the same
thing):

1.  The improved code organization mentioned above. If we're being
    forced to properly isolate our code rather than dumping it all into
    a giant user model, then there are going to be fewer conflicts in
    the user model.

2.  Since the components tend to encapsulate a functional section of the
    application, it's likely that most of the code you're working on for
    a feature will fall into one or two components. When one developer
    is working in the admin component and another is working in the
    reporting component, they're unlikely to overlap and conflict.

### Code Re-Use

When starting out, we expected code reuse to be a major benefit of the
engine-based approach. In practice, we didn't end up with as many
reusable components as we expected and most of our engines were pretty
application specific. However, we did end up with some pieces that we
could reuse in other projects, including a [commenting component][6].

## Cons

### Associations Can be a Pain

If you're setting up your dependencies appropriately, you won't have
circular dependencies between components. So, in our case, the
forecasting component knows about the user/account component, but not
vice versa.

This can make associations a pain. If the user component doesn't "know"
about the forecasting component (which contains a Question model), then
we really shouldn't be putting forecasting-related associations in the
user model. So something like this would be a no-go:

    
    class User
      has_many :questions
    end

Instead, we'd do something like:

    
    class Question
      scope :for_user, ->(user){ where(user_id: user.id) }
    end
    
    # Use it like this:
    Forecast::Question.for_user(user)

This takes some getting used to, feels less clean than a normal
association, and is still not my favorite solution.

### Component Isolation is Difficult to Enforce

When building a Rails app, pretty much everything gets loaded up and is
available to you in any other place in the app -- no `import
"filename"` needed. This is convenient, but also makes it trivial to
pollute a component with knowledge of another, even if they should be
isolated.

For example, I mentioned above that we have an accounts component and a
forecasting component. Technically, the accounts component has no
dependency on the forecasting component (and does not declare one in the
gemspec). However, something like this will "just work" if you do it:

    
    module Accounts
      class User
        def do_something_with_a_question
          question = Forecast::Question.first
          question.do_something
        end
      end
    end

If we want the account/users component to be an isolated component that
could be extracted &amp; reused, then it should be agnostic of the
Forecast component. For better or worse, it's easy to accidentally
violate this isolation.

### Test Environment Setup is Challenging

Ideally, all of your components would be able to run their test suites
in isolation. If an engine didn't declare (via its gemspec) a dependency
on another component, then that component wouldn't be loaded in the test
environment for that engine.

To make this happen, you need a dummy app inside every engine. This can
get cumbersome very quickly as the number of engines you have grows. For
example, in [Cultivate Forecasts][2], we have an FeedUI component that
contains an activity feed. We also have a commenting component, which is
utilized by FeedUI to allow users to comment on activity feed stories.
So we built out the code in the FeedUI which leverages the commenting
component.

Naturally, we want to have test coverage for the commenting
functionality, including feature/integration tests. We just built out an
endpoint in FeedUI that could be used in a feature spec to test creation
&amp; listing/viewing of comments. But the commenting gem has no
dependency on FeedUI and technically doesn't even know it exists, so
putting a feature spec inside the commenting engine that goes and does
`visit "/activity"` seems like a bad idea.

The solution then becomes to build something into the commenting
component's dummy app that also utilizes the commenting features,
essentially duplicating the work we just did on the activity feed. This
tends to happen fairly frequently.

To help address this, we decided to eliminate dummy apps from the
engines and instead use the host app to do testing. This helped
eliminate some of the duplication, but has the downside of polluting
components with knowledge of other components where there should be no
dependency. Frankly, I haven't yet found or seen an approach to this
issue that solves both of these issues.

### Increased Developer Ramp-Up Time

Probably the least surprising of the cons we encountered with this
pattern was increased developer ramp up time. The pattern is relatively
new and plenty of Rails developers don't have a ton of experience with
building engines or gems. There's also a period of learning where to
find things inside the app, since you can no longer just go to
app/models to find every model in the app.

The flip side of this is that developers can be productive in a single
component without having to understand every piece of the app. If
someone learns the commenting engine well, they can make progress on
comment-related features without having to understand all of the other
components.

If you enjoyed this post, let me know on twitter: [@bcroesch][7]



[1]: http://guides.rubyonrails.org/engines.html
[2]: https://www.cultivatelabs.com/forecasts
[3]: https://sportscast.cultivateforecasts.com/activity
[4]: https://en.wikipedia.org/wiki/Prediction_market
[5]: http://devblog.avdi.org/2011/07/05/demeter-its-not-just-a-good-idea-its-the-law/
[6]: https://github.com/CultivateLabs/flyover_comments
[7]: http://twitter.com/bcroesch
