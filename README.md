# Devise with Roles

## Learning Objectives

  1. Explain the use of role-based authorization with Devise.
  2. Design a set of roles to model a forum with different permission levels.
  3. Set up Devise roles to implement such a model.

## Overview

[Devise] gives you basically everything you need to solve the problem of authentication. [CanCanCan] offers a simple authorization model. In this lesson, we're going to look at using the role pattern for authorization.

## What's a role?

Let's say that our app is a message board. There are four kinds of users: guests, normal users, moderators, and administrators.

Here's how they're defined and what they can do:

   * guests can read posts
   * normal users can do everything guests can do. They can also create posts and edit their own posts.
   * moderators can do all that, and edit and delete the posts of other users.
   * administrators can do everything.

Roles are a way to express these different kinds of user within the `User` model, then use it for authorization.  Devise allows us to authenticate WHO you are, and devise's roles allow us to say given what KIND of user you are, what you are authorized to do.

## Using roles

Let's look at how we might implement the schema described above.

First, we'll put an [`enum`][ar_enum] in our `User` model to keep track of the `User`'s role:

    class User < ActiveRecord::Base
      enum role: [:normal, :moderator, :admin]
    end

ActiveRecord generates several helpful methods from this call. Given a `User`, we can ask,

    user.admin?

And ActiveRecord will translate that into:

    user.role == 2

As you can see, enums are stored as integers in the database. The list of allowable values, and their conversion into symbols, happens in Ruby.

What about guests, or users who are either not logged in or not signed up for our service at all? We can detect whether a `User` is written to the database by calling `persisted?`, which will catch user objects which have been created, perhaps filled out in a `/signup` route, but not yet saved and validated. These users should be guests.

Our `User` model updates like so,

    class User
      enum role: [:normal, :moderator, :admin]
      def guest?
        persisted?
      end
    end

Note: `current_user.role` will never return `'guest'` with this approach, since being a guest is not technically a role.

Devise's current_user method will always return us the user object for the currently logged in user.  What happens if we call `current_user.guest?` and `current_user` is `nil` because nobody is currently signed in. We can address that like so:

    class NilClass
      def guest?
        true
      end
    end

This will cause `current_user.guest?` to return true if `current_user` is nil. It will also cause *any* `nil` value, even literally `nil` to respond to `guest?` with `true`, so be wary of overusing this approach. Another technique is to use ActiveSupport's `try` whenever we're using an object that might be nil:

    if current_user.try(:admin?)
      # current user is not nil and is an admin
    else
      # current user is nil or has a role that is not admin
    end

This is perhaps cleaner, since it delegates the bizarre re-opening of `NilClass` to `ActiveSupport#try`, but it is a little bit less readable than this:

    if current_user.admin?
      # current user is not nil and is an admin
    else
      # current user is nil or has a role that is not admin
    end

So use your discretion.

## Using the pattern

You can use this pattern whether you're just doing authorization checks in your controllers, or using a framework like CanCan.

Using it bare in the controller,

    class Post
      def update
        @post = Post.find(params[:id])
        return head(:forbidden) unless current_user.admin? ||
	       			       current_user.moderator? ||
				       current_user.try(:id) == @post.id
        @post.update(post_params)
      end
      # more down here
    end


In a `CanCanCan` ability, we use the role (What type of user you are) to determine what you are authorized to do.

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

Note how the flow of the initializer creates cascading permissions. Since our permissions model is cascading—each new level of user can do everything the previous level can do, and then more—we start at the least-privileged user and add in the permissions for each level, returning when we reach the user's permission level.

## Conclusion
Roles sit somewhere between authorization and authentication.  By pre-defining what type of user each user is, and what each type of user is authorized to do, we can use roles to say who is allowed to do what.

## Resources

  * [ActiveRecord enum][ar_enum]
  * [CanCan]
  * [Devise]

[ar_enum]: http://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html
[CanCanCan]: https://github.com/ryanb/cancan
[Devise]: https://github.com/plataformatec/devise
