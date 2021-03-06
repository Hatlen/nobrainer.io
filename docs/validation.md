---
layout: docs
title: Validations
prev_section: callbacks
next_section: dirty_tracking
permalink: /validations/
---

## Declaring Validators

Validations works the same way as in ActiveRecord because NoBrainer reuses the
ActiveModel validation logic. If you are not familiar with how validations
typically operate, please read the following documentation:
[ActiveRecord validations](http://edgeguides.rubyonrails.org/active_record_validations.html).  
However, there are a some differences with the
associated validator and the uniqueness validator, which are explained below.

There are six ways to declare validations with NoBrainer:

* `validate_presence_of :field_name`
* `validates :field_name1, :field_name2, :presence => true`
* `validate { errors.add(:base, "too many friends") if too_many_friends? }`
* `field :field_name, :validates => { :presence => true }`
* Using shorthands as described below.
* Using types: `field :field_name, :type => Integer`. This will validate that the
  given field is an integer. Read more about the type checking mechanism in the
  [Types](/docs/types) section.

## Shorthands

### required

You may use the `required` shorthand to specify a presence validator:

{% highlight ruby %}
class Model
  field :name, :required => true
end
# Equivalent to:
class Model
  field :name, :validates => { :presence => true }
end
{% endhighlight %}

### unique

You may use the `unique` shorthand to specify a uniqueness validator:

{% highlight ruby %}
class Model
  field :name, :unique => true
end
# Equivalent to:
class Model
  field :name, :validates => { :uniqueness => true }
end
{% endhighlight %}

### in

You may use the `in` shorthand to specify a inclusion validator:

{% highlight ruby %}
class Model
  field :state, :in => %w(start finish)
end
# Equivalent to:
class Model
  field :name, :validates => { :inclusion => { :in => %w(start finish) } }
end
{% endhighlight %}

## When are validations performed?

### Validations are performed on:

Validations are performed when calling the following methods on an instance:
* `save`
* `save!`
* `create`
* `create!`
* `update_attributes`
* `update_attributes!`

If you want to bypass validations, you may pass the `:validate => false` option
to these methods, which can be quite handy in a development console. Do not use
such thing in your actual code.

The bang versions follow the same semantics of ActiveRecord which is to raise
when validation fails. NoBrainer raises a `NoBrainer::Error::DocumentInvalid`
exception when validation fails. If left uncaught in a Rails controller, a 422
status code will be returned.
The non bang versions populate the errors array attached to the instance.
`save` and `update_attributes` return true or false depending if the validations
failed, while `create` returns the instance with an non empty `errors`
attribute.

### Validations are *not* performed on:

Validations are not performed when updating all documents matching a criteria,
such as `Model.update_all()`.

## Presence Validations on belongs\_to Associations

To add a presence validation on a `belongs_to` association, there are two ways.
You are offered the trade-off between correctness and performance.

The first way is to add the presence validation on the association:

{% highlight ruby %}
class Comment
  belongs_to :post, :required => true
end
{% endhighlight %}

Adding a presence validation on the association instructs NoBrainer to verify
that the object pointed by the foreign_key actually exists in the database.
An extra database read query will be performed if the association is not already
cached.

---

The other way to add a presence validator is to declare it on the foreign key:

{% highlight ruby %}
class Comment
  belongs_to :post
  field :post_id, :required => true
end
{% endhighlight %}

By declaring the presence validator on the foreign key, NoBrainer will only
check that the foreign key contains something. No extra database lookup is
performed to verify that the foreign key actually points to a valid object.

## The Uniqueness Validator

The uniqueness validator ensures that a field value can be present at most once
table wide.

For performance reasons, NoBrainer only performs uniqueness validations when the involved
fields change.

### Using scopes

The uniqueness validator accept a `:scope` option which can be a field, or an
array of fields. For example:

{% highlight ruby %}
class TeamMember
  field :team_name
  # Do not allow duplicate emails within a team.
  field :email, :validates => {:uniqueness => {:scope => :team_name}}
end
{% endhighlight %}

It is highly recommended that you add a presence validator on the scoped fields,
because RethinkDB considers `nil` and `undefined` to be two different things.

### Race Free Uniqueness Validations

When working with traditional ORMs, the uniqueness validator is known to be
racy: two concurrent requests may both pass the validation, and both could
persist successfully the same supposedly unique field.
Uniqueness validators are useful in conjunction with unique secondary indexes.
Since RethinkDB is a sharded database, implementing unique
secondary indexes is a performance problem, and so the RethinkDB team rightfully
decided not to implement them. To really ensure uniqueness, one must either
leverage the primary key uniqueness guarantee, or use a distributed lock.

NoBrainer can be configured to use distributed locks to perform non-racy uniqueness
validations. This mechanism is enabled by providing a *`Lock`* class through the
`distributed_lock_class` setting when configuring NoBrainer.
You may use any lock service such as Redis or ZooKeeper by providing a `Lock`
class that complies to the following API:

{% highlight ruby %}
class Lock
  # The initializer must accept a key argument as a String.
  # The key follow the following format:
  #   "nobrainer:database_name:table_name:field_name:field_value"
  # Example: "nobrainer:production:users:username:nico"
  def initialize(key)
  end

  # Acquires a lock on @key. Returns nothing. May raise exceptions.
  def lock
  end

  # Releases the lock on @key. Returns nothing. May raise exceptions.
  def unlock
  end
end
{% endhighlight %}

The following shows an example of using [Redis](http://redis.io/) and the
[robust-redis-lock](https://github.com/crowdtap/robust-redis-lock) gem
to provide non-racy uniqueness validations:

{% highlight ruby %}
require 'robust-redis-lock'
Redis::Lock.redis = Redis.new(:url => 'redis://localhost')

NoBrainer.configure do |config|
  config.distributed_lock_class = Redis::Lock
end
{% endhighlight %}

The locks are acquired after the `before_create/update` callbacks, and before
the `after_create/update` callbacks.
NoBrainer alpha sorts the keys to be acquired to avoid deadlock issues when
performing multiple uniqueness validations on the same document.

## Differences with ActiveModel

### Associated Validator

The associated validator is not implemented.

### Uniqueness Validator

The uniqueness validator does not accept the `:case_sensitive` option.
Downcasing the attribute in a `before_save/validation` callback is a better idea.
