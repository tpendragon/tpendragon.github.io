---
layout: post
title: ActiveFedora Relationships
tags: ruby rails active-fedora
---

In a recent sprint on [ActiveFedora
Aggregations](https://github.com/projecthydra-labs/activefedora-aggregation/) I
was working on Relationships in ActiveFedora and felt like I should catalogue my
understanding of how they work.

## Entry Point

Let's start with the `has_many` method, as used below:

```ruby
class Collection < ActiveFedora::Base
end
class MyObject < ActiveFedora::Base
  has_many :collections, :class_name => "Collection"
end
```

That has_many method is defined in the associations module which is included
into ActiveFedora::Base here: [lib/active_fedora/associations.rb#L142-L144](https://github.com/projecthydra/active_fedora/blob/master/lib/active_fedora/associations.rb#L142-L144)

The relevant code looks like this:

```ruby
def has_many(name, options={})
  Builder::HasMany.build(self, name, options)
end
```

As you can see, when you call has_many on the ActiveFedora class it delegates
down to the HasMany builder.

## Builder

Builders are responsible for setting up a Reflection (a registry of the metadata
for an association) and defining readers/writers on the class it was called on.
The class for HasMany is defined here:
[lib/active_fedora/associations/builder/has_many.rb](https://github.com/projecthydra/active_fedora/blob/master/lib/active_fedora/associations/builder/has_many.rb)

The relevant code looks like this:

```ruby
def self.build(model, name, options)
  reflection = new(model, name, options).build
  define_accessors(model, reflection)
  define_callbacks(model, reflection)
  reflection
end

def build
  reflection = super
  configure_dependency
  reflection
end
```

When the super call is inlined it looks like this:

```ruby
def self.build(model, name, options)
  reflection = new(model, name, options).build
  define_accessors(model, reflection)
  define_callbacks(model, reflection)
  reflection
end

def build
  configure_dependency if options[:dependent] # see https://github.com/rails/rails/commit/9da52a5e55cc665a539afb45783f84d9f3607282
  reflection = model.create_reflection(self.class.macro, name, options, model)
  configure_dependency
  reflection
end
```

A reflection is created from the calling model (`MyObject`) which allows you to
obtain all relevant information about the association by calling
`MyObject.reflections`

After that it defines accessors and callbacks. Let's look at the accessors
created.

## Accessors

Accessors are defined for has_many in a function that looks like this:

```ruby
def self.define_readers(mixin, name)
  super

  mixin.redefine_method("#{name.to_s.singularize}_ids") do
    association(name).ids_reader
  end
end
```

If you inline the super call:

```ruby
def self.define_readers(mixin, name)
  mixin.send(:define_method, name) do |*params|
    association(name).reader(*params)
  end

  mixin.redefine_method("#{name.to_s.singularize}_ids") do
    association(name).ids_reader
  end
end
```

This makes it so if you do `has_many :plums` it will define an instance method
called `plums` on the object which then calls the association's `reader` method.

The reader method:

```ruby
# Implements the reader method, e.g. foo.items for Foo.has_many :items
# @param opts [Boolean, Hash] if true, force a reload
# @option opts [Symbol] :response_format can be ':solr' to return a solr result.
def reader(opts = false)
  if opts.kind_of?(Hash)
    if opts.delete(:response_format) == :solr
      return load_from_solr(opts)
    end
    raise ArgumentError, "Hash parameter must include :response_format=>:solr (#{opts.inspect})"
  else
    force_reload = opts
  end
  reload if force_reload || stale_target?
  @proxy ||= CollectionProxy.new(self)
end
```

Effectively what happens is when you call `#plums` you get back a
CollectionProxy object, which is defined here:
[lib/active_fedora/associations/collection_proxy.rb](https://github.com/projecthydra/active_fedora/blob/master/lib/active_fedora/associations/collection_proxy.rb)

On this object are methods you might expect - `#find`, `#first`, `#last`, etc.
If you wanted to change what methods are available on a set of related objects,
that's where you'd change things.

## And that's it

That's all there is to it. A builder gets called when you do things like
`has_many` or `belongs_to`, that builder stores the metadata about the
association, it defines a reader which delegates down to an association object
(such as
[lib/active_fedora/associations/has_many_association.rb](https://github.com/projecthydra/active_fedora/blob/master/lib/active_fedora/associations/has_many_association.rb)),
and the reader often defines a proxy object to handle actions on the associated
items as a whole.

For me the biggest trouble I had was tracing the path. To do so, just start at
the associations module and work your way down.
