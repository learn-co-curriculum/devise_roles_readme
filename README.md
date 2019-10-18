# Devise with Roles

## Learning Objectives

  1. Explain the use of role-based authorization with Devise.
  2. Design a set of roles to model a forum with different permission levels.
  3. Set up Devise roles to implement such a model.

## Overview

[Devise] gives you basically everything you need to solve the problem of
authentication. [CanCanCan] offers a simple authorization model. In this
lesson, we're going to look at using the role pattern for authorization.

## What's a role?

Let's say that our app is a message board. There are four kinds of users:
guests, normal users, moderators, and administrators.

Here's how they're defined and what they can do:

* **guests** can read posts
* **normal users** can do everything guests can do. They can also create posts and edit their own posts.
* **moderators** can do all that, and edit and delete the posts of other users.
* **administrators** can do everything.

Roles are a way to express these different kinds of users within the `User`
model, then use it for authorization.  Devise allows us to authenticate WHO you
are, and devise's roles allow us to say (given what KIND of user you are) what
you are authorized to do.

## Using Roles

Let's look at how we might implement the schema described above.

First, we'll put an [`enum`][ar_enum] in our `User` model to keep track of the
`User`'s role:

```ruby
class User < ActiveRecord::Base
  enum role: [:normal, :moderator, :admin]
end
```

`ActiveRecord` generates several helpful methods from this call. Given a
`User`, we can ask,

```ruby
user.admin?
```

And `ActiveRecord` will translate that into:

```ruby
user.role == 2
```

As you can see, `enum`s are stored as integers in the database. The list of
allowable values, and their conversion into symbols happens in Ruby.

What about **guests**, or users who are either not logged in or not signed up
for our service at all? We can detect whether a `User` is written to the
database by calling `persisted?`, which will catch user objects that have been
created, perhaps filled out in a `/signup` route, but not yet saved and
validated. These users should be guests.

Our `User` model updates like so,

```ruby
class User
  enum role: [:normal, :moderator, :admin]
  def guest?
    persisted?
  end
end
```

Note: `current_user.role` will never return `'guest'` with this approach, since
being a guest is not _technically_ a role.

## `current_user` and `nil`

Devise's `current_user` method will always return to us the user object for the
currently logged-in user &mdash; ***provided that there is a currently logged-in
user***.

That's really important. If the agent visiting our site is not logged in and we
call a method on `current_user` we're going to get a `Nil` error since we'd be
calling `.guest?` on `nil`.

To protect ourselves, this leads to code like:

```ruby
if current_user
  if current_user.admin?
    # current user is not nil and is an admin
  else
    # current user is nil or has a role that is not admin
  end
else
  # Not logged-in experience.
end
```

Generally, as programmers, we want to start feeling a little bit uncomfortable
when we see nested conditionals. In order to make sure we don't have any
accidental outcomes, we have to cover `2` raised to the power of the number of
conditionals. In the example above, because there are two conditionals (`if`)
and every conditional has a `true`/`false` pair, then we have to cover
`2<sup>2</sup>` or `4` code paths.

In Rails, there are a couple of ways to keep our conditional to only two paths.
One way would be to require that this action will only execute when there is a
`current_user`. This can be done with Rails' [before action]. If this filter
were in place, we could remove the outermost `if`.

Another way is to find a way to write this conditional more cleverly.  The
`ActiveSupport` gem, a default part of Rails, provides a method called `try`.
We pass `try` a method we _hope_ will not be invoked on `nil`. If `try` detects
it would invoke the method on `nil`, it simply returns `nil` itself, a falsey
value. If the method _would not_ be invoked on `nil`, `try` invokes the method
and returns the result for you.

```ruby
if current_user.try(:admin?)
  # current_user is not nil
  # invoke .admin? on the current_user and return true or false 
else
  # return `nil`, a falsey value
end
```

The three approaches for handling a `nil` current user are:

1. Nested conditionals
2. Before filters
3. Using `try`

Developers will have different opinions about which is better. The first mode
is very simple to read and very obvious. The latter two take advantage of cool
features of Rails. We think the smartest way to ensure actions that need a
`current_user` have it in Rails controller actions is to use a [before action].

## Using the Pattern

You can use this pattern whether you're just doing authorization checks in your
controllers, or using a framework like [CanCanCan].

Using it bare in the controller,

```ruby
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    return head(:forbidden) unless current_user.admin? ||
                 current_user.moderator? ||
           current_user.try(:id) == @post.owner_id
    @post.update(post_params)
  end
  # more down here
end
```


In a `CanCanCan` ability, we use the role (What type of user you are) to determine what you are authorized to do.

```ruby
class Ability
  include CanCan::ability

  def initialize(user)
    user.can :read, Post
    return if user.guest?

    user.can :update, Post, {owner_id: user.id}
    return if user.normal?

    user.can :update, Post
    return if user.moderator?

    user.can :manage, Post if user.admin?
  end
end
```

Note how the flow of the initializer creates cascading permissions. Since our
permissions model is cascading—each new level of user can do everything the
previous level can do, and then more—we start at the least-privileged user and
add in the permissions for each level, returning when we reach the user's
permission level.

## Conclusion

Roles sit somewhere between authorization and authentication.  By pre-defining
what type of user each user is, and what each type of user is authorized to do,
we can use roles to say who is allowed to do what.

## Resources

  * [ActiveRecord enum][ar_enum]
  * [CanCanCan]
  * [Devise]

[ar_enum]: http://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html
[CanCanCan]: https://github.com/CanCanCommunity/cancancan
[Devise]: https://github.com/plataformatec/devise
[try]: https://guides.rubyonrails.org/active_support_core_extensions.html#try
[before action]: https://guides.rubyonrails.org/action_controller_overview.html#filters

<p data-visibility='hidden'>View <a href='https://learn.co/lessons/devise_roles_readme'>Devise Roles </a> on Learn.co and start learning to code for free.</p>

