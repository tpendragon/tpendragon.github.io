---
layout: post
title: Why Not Callbacks
tags: ruby rails
---

## The Problem

You often want something to happen at a stage in an ActiveRecord object's
standard workflow - most often this is "After I save I want to do X."

For Example:

```ruby
class File < ActiveRecord::Base
  validates :file_path, :presence => true
end

File.save # => Persist, if successful,
# create a thumbnail version for the web.
```

## Why Callbacks

An often recommended way to deal with this is as an after_save callback.

```ruby
class File < ActiveRecord::Base
  validates :file_path, :presence => true
  after_save :create_thumbnail
  
  private

  def create_thumbnail
    ThumbnailCreator.call(file_path)
  end
end
```

**PROs**:
- Simple, concise, all the logic is in one place you can look at.
- You can never accidentally save a file without it creating a thumbnail.

Before I get to the CONs, let's look at some assumptions made:

**Assumptions**:
- We're okay opening up the file class whenever we want to change when and how
  we create a thumbnail.
- We ALWAYS want to create a thumbnail.
- If we save a file, we're okay with whatever overhead exists to create that
  thumbnail.
- We're okay with not injecting dependencies (ThumbnailCreator), and thus when
  that changes we're happy to change it in File.

**CONs**:
- State is mixed up with business logic. Suddenly my "File" is not just
  "attributes with persistence", and is now "attributes with persistence
  and side effects too"
- NOT creating a thumbnail is hard. How do I avoid callbacks? Only this SPECIFIC
  callback?
- Doing something like pushing that thumbnail creation to the background is
  complicated logic that will sit in the model.
- ThumbnailCreator is hard-wired, its responsibility is inflexible.

## Why Not Callbacks

It turns out the assumption that `we ALWAYS want to create a thumbnail` is a big
one. It's rare that you ALWAYS want something to happen, and building in that
assumption is the beginning to expensive backtracking or hacks later.

The big point is this: `business logic changes at a different rate than object
representation`. Tying the logic of when and how to create a thumbnail to the
representation of how a File is stored means coupling changes in the File to
changes in how a thumbnail is created, unnecessarily.

## Alternatives

### Service Objects

We can encapsulate what to do when an object is saved by putting all of it in
one object.

```ruby
class FileSaver
  attr_reader :file, :thumbnail_creator
  def initialize(file, thumbnail_creator)
    @file = file
    @thumbnail_creator = thumbnail_creator
  end

  def run
    if file.save
      thumbnail_creator.call(file.file_path)
    end
  end
end
```

This works - you can inject ThumbnailCreator, you only use the FileSaver when
you want, and you will never accidentally create a thumbnail.

However, what tends to happen is that ALL behavior gets stored here and the
problem of "how do I ignore just ONE callback action" remains.

### Decoration of Functionality

We can decorate JUST this functionality onto the object.

```ruby
class CreatesThumbnails < SimpleDelegator
  attr_reader :thumbnail_creator

  def initialize(file, thumbnail_creator)
    @thumbnail_creator = thumbnail_creator
    super(file)
  end

  def save
    if __getobj__.save
      thumbnail_creator.call(file_path)
    end
  end
end

CreatesThumbnails.new(File.new(:file_path => "/file"), ThumbnailCreator).save
```

Now when you want to create a thumbnail on save you can decorate the object with
this, call save, and you're good. If when or how to run the thumbnail creator
changes, you can inject a new thumbnail creator or change just the
CreatesThumbnails object. Business logic is now separated from state.

Decorating all the necessary actions and dependencies for a transaction can get
complicated, at which point the Factory Pattern can help. I may do a post on
this at a later date.
