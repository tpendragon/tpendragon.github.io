---
layout: post
title: Abusing Decorators
tags: ruby patterns decorators
---

## The Decorator Pattern

The Decorator Pattern's goal is to dynamically add responsibilities to an object
instance during runtime. This lets an individual object instance behave
differently depending on the circumstances and requirements it may have at the
time. This flexibility is powerful, but with power comes danger - so let's
abuse the pattern until it hurts.

## Decorators in Ruby

When I implement a new pattern in ruby I like to attach it to an interface I
define ahead of time, that way I can easily swap out individual instances and as
long as they conform to that interface I can call it an "X".

### The Interface

A Decorator is any object which responds to new, takes one argument (the
instance to be decorated), and returns an object that responds to at least the
same interface as the previous object and includes additional behavior.

### The Code

I like to use SimpleDelegator to accomplish this. Let's take a simple
ActiveRecord class.

```ruby
class Person < ActiveRecord::Base
end
```

Now let's say we want to keep a close look at all people whose names are "Bob."
Whenever those records are saved, log a message.

```ruby
class LogsBobSaves < SimpleDelegator
  def save(*args)
    with_logger do
      super(*args)
    end
  end

  private

  def with_logger
    yield.tap do |result|
      if name == "Bob"
        logger.info "Bob was saved"
      end
    end
  end

  def logger
    Rails.logger
  end
end
```

This works, but one of the benefits of using the decorator pattern is you can
use dependency injection to make even more flexible objects which can use a
variety of collaborators.

```ruby
class LogsBobSaves < SimpleDelegator
  attr_reader :logger
  def initialize(obj, logger)
    @logger = logger
    super(obj)
  end

  def save(*args)
    with_logger do
      super(*args)
    end
  end

  private

  def with_logger
    yield.tap do |result|
      if name == "Bob"
        logger.info "Bob was saved"
      end
    end
  end
end
```

Now you can initialize a Person which saves logs about Bob by doing the
following:

```ruby
person = LogsBobSaves.new(Person.new(:name => "Bob"), Rails.logger)
person.save # => Logs the message "Bob was saved"
```

But wait! We said earlier the interface was that there was only one argument to
 #new, and this has two. Now begins the abuse: Let's use the `Adapter` pattern to
maintain the interface.

```ruby
class DecoratorWithArguments
  attr_reader :decorator, :args
  def initialize(decorator, *args)
    @decorator = decorator
    @args = args
  end

  def new(obj)
    decorator.new(obj, *args)
  end
end

person = Person.new(:name => "Bob")
decorator = DecoratorWithArguments.new(LogsBobSaves, Rails.logger)
person = decorator.new(person) # One argument
person.save # => Logs the message "Bob was saved"
```

## Interface Importance

The adapter above is clearly more complicated than just passing the logger in.
Why maintain an arbitrary (and simple) interface?

Let's say I want to also log when the name is Fred.

```ruby
class LogsNameSaves < SimpleDelegator
  attr_reader :logger, :name_check
  def initialize(obj, name_check, logger)
    @logger = logger
    @name_check = name_check
    super(obj)
  end

  def save(*args)
    with_logger do
      super(*args)
    end
  end

  private

  def with_logger
    yield.tap do |result|
      if name == name_check
        logger.info "#{name} was saved"
      end
    end
  end
end

person = Person.new(:name => "Fred")
decorator1 = DecoratorWithArguments.new(LogsNameSaves, "Bob", Rails.logger)
decorator2 = DecoratorWithArguments.new(LogsNameSaves, "Fred", Rails.logger)
person = decorator1.new(person)
person = decorator2.new(person) # Decorate a second time.
person.save # => Logs
person.name = "Bob"
person.save # => Logs
```

To apply multiple decorators we have to call #new on them multiple times. We
have a consistent interface, and we could end up decorating an arbitrary number
of times, so let's use the `Composite` pattern to encapsulate that.

```ruby
class DecoratorList
  attr_reader :decorators

  def initialize(*decorators)
    @decorators = decorators
  end

  def new(undecorated_object)
    decorators.inject(undecorated_object) do |obj, decorator|
      decorator.new(obj)
    end
  end
end

person = Person.new(:name => "Bob")
decorator = DecoratorList.new(
  DecoratorWithArguments.new(LogsNameSaves, "Bob", Rails.logger),
  DecoratorWithArguments.new(LogsNameSaves, "Fred", Rails.logger),
)
person = decorator.new(person) # One decoration call results in two decorations.
person.save # => Logs
person.name = "Fred"
person.save # => Logs
```

Now how about adding Susan when we already have two decorators?

```ruby
decorator = DecoratorList.new(
  DecoratorWithArguments.new(LogsNameSaves, "Bob", Rails.logger),
  DecoratorWithArguments.new(LogsNameSaves, "Fred", Rails.logger),
)
decorator = DecoratorList.new(
  decorator,
  DecoratorWithArguments.new(LogsNameSaves, "Susan", Rails.logger),
)
person = decorator.new(person) # One decoration call results in two decorations.
```

The beauty of Composites - you can build composites of composites, because they
all maintain the same simple interface.

It was a bit of a bother having to decorate at all, why not encapsulate what
decorations go into creating a loggable person?

```ruby
class LoggablePersonFactory
  def new(*args)
    decorator.new(Person.new(*args))
  end

  private

  def decorator
    DecoratorList.new(*decorators)
  end

  def decorators
    logging_names.map do |name|
      DecoratorWithArguments.new(LogsNameSaves, name, logger)
    end
  end

  def logging_names
    ["Bob", "Fred", "Susan"]
  end

  def logger
    Rails.logger
  end
end
person = LoggablePersonFactory.new(:name => "Fred")
person.save # => Logs
person.name = "Susan"
person.save # => Logs
```

## The Pros

1. Decorators result in immensely configurable behavior enhancement.
2. One object can have different behavior from another despite sharing a class.
3. Composing behaviors allows for single responsibility and yet powerful 
   objects.
4. Dependencies can be injected, but only have to be done at the decorator
   level.
5. You never have to ask "how can I get an object that doesn't do X?" Just don't
   decorate the object with that behavior.

## The Cons

1. Deciding when and where to apply decorators is difficult.
2. SimpleDelegators can sometimes lose context (in the above, if you use "save!"
   instead of "save" it's not defined on the decorator, so it won't log even if the
   decorated object calls "save" in its "save!" implementation.)
3. Instantiating all the decorators is expensive.
4. As you abuse the pattern it becomes difficult to keep track of which
   behaviors are available to you.

To solve 1, 3, and 4 I like to define concrete factories whose responsibility is
generating objects with a specific subset of decorators. If you need a
combination of factories, just compose them.

Problem 2 is more difficult - all you can do is make sure that your calling
objects are using interfaces you've defined on the decorator. Ruby's
implementation is a "Delegator", and lacks more context-aware decoration that's
available in other languages.

## Should I Decorate?

I'm not sure. I am. It gives me the strengths of something like multiple
inheritance with the flexibility to change my mind. Further, unlike mixins
decoration doesn't allow for two-way interactions - my underlying object will
never come to rely on outside code, so I can be sure that it works independent
of my decorators and thus gives me a smaller axis to debug.

However, as above, it can be abused. I've been leaning on the side of decorating
lately, and it's becoming time to take a step back and see where it doesn't
work. When I have that boundary clearly defined, I'll post again.

Programming is all about figuring out when and where to apply a solution.
