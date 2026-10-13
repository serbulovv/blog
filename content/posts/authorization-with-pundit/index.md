+++
date = '2026-10-03T00:00:00+02:00'
title = 'Authorization in Rails API with Pundit'
author = 'Genry Wood'
tags = ['pundit', 'authorization', 'ruby', 'gem', 'rails', 'security']
+++

![Post Logo](./header.png)

Authentication identifies the user. In an API this usually means decoding a JWT and finding the corresponding user. Authorization determines whether that user is allowed to perform a specific action on a specific resource. To check if user has allow to perfor something we will use the [Pundit gem.](https://github.com/varvet/pundit) 
Pundit is a gem for authorization in Rails applications. Each model has a policy, and a policy is a plain Ruby class.

Without a dedicated tool, authorization checks tend to end up inline in controllers:

```ruby
return head :forbidden unless post.author == current_user
```

The same check is then repeated across several controllers and drifts out of sync over time. Pundit moves these rules into separate classes, one per model.

## How Pundit works

A policy is a Ruby class. A rule is a method that returns `true` or `false`. Pundit finds the policy by naming convention: the model `Post` uses `PostPolicy`, and the controller action `update` maps to the method `update?`.

Pundit does not use a DSL. CanCanCan, a similar gem, defines rules through its own syntax:

```ruby
can :update, Post, author_id: user.id
cannot :destroy, Post
```

`can` and `cannot` belong to a domain-specific language (DSL) provided by the gem. In Pundit the same rule is a regular method:

```ruby
def update?
  record.author_id == user.id
end
```

## Setup

Add the gem to the `Gemfile` and run `bundle install`:

```ruby
gem 'pundit'
```

`ActionController::API` does not include Pundit, so the module has to be included in the base controller:

```ruby
# frozen_string_literal: true

class ApiController < ActionController::API
  include Pundit::Authorization

  def pundit_user
    current_user
  end
end
```

`pundit_user` defines which user performs the action. By default Pundit calls `current_user`, so the override is optional when the method has that name.

## Writing a policy

The generator `rails g pundit:install` creates `ApplicationPolicy`. The same file can be written manually:

```ruby
# frozen_string_literal: true

class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?
    false
  end

  def show?
    false
  end

  def create?
    false
  end

  def update?
    false
  end

  def destroy?
    false
  end
end
```

All actions are denied by default. A policy that does not override a method leaves that action closed.

Each policy receives two objects: `user`, the one performing the action, and `record`, the object the action is performed on. A policy for posts for example:

```ruby
# frozen_string_literal: true

class PostPolicy < ApplicationPolicy
  def index?
    true
  end

  def show?
    true
  end

  def create?
    user.present?
  end

  def update?
    owner?
  end

  def destroy?
    owner?
  end

  private

  def owner?
    user.present? && record.author_id == user.id
  end
end
```

Under these rules any user can read posts, any authenticated user can create a post, and only the author can update or delete it.

## Using policies in controllers

`authorize` takes a record, finds its policy, and calls the method that matches the current action. If the method returns `false`, Pundit raises `Pundit::NotAuthorizedError`.

```ruby
# frozen_string_literal: true

module Api
  module V1
    class PostsController < ApiController
      before_action :check_authentication!

      def update
        post = Post.find(params[:id])
        authorize post

        post.update!(post_params)

        render json: post
      end

      private

      def post_params
        params.permit(:title, :body)
      end
    end
  end
end
```

In the `update` action, `authorize post` calls `PostPolicy#update?`. To check a different rule, pass the method name explicitly:

```ruby
authorize post, :publish?
```

The `policy` helper returns the policy object without raising an error. It can be used to include permission flags in a response:

```ruby
render json: post, meta: { can_edit: policy(post).update? }
```

A frontend can use such flags to show or hide controls, while the rule itself stays in the policy.

## Summary

Pundit keeps authorization rules in plain Ruby policy classes, one per model, denies everything by default, and applies a rule with a single authorize call in the controller, which makes it a good fit once permissions depend on the record itself, such as its owner, state, or group membership.