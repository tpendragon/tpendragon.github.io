---
layout: post
title: Isolated Testing
tags: ruby rails rspec capybara unit-tests
---

*To see the previous post in this series please check out [Avoiding Integration Tests]({% post_url 2015-04-22-avoiding-integration-tests %})*

## Test Driven Development

When I left off I had fast, predictable, and reliable unit tests which, if I was
careful, would ensure that my application worked. However - I'd been promised
something: if I wrote the tests first, they would tell me if I was architecting
the application well.

I didn't feel that way yet.

### Finding Complexity

Experience and (a **lot**) of reading later I decided that a good application
architecture is one which fulfills the use cases with the smallest possible
amount of complexity and the lowest amount of coupling. Complexity in
applications comes, primarily, from two places:

1. Branches
2. Dependencies

My tests were showing me the branches - if I had too many, I would have too many
tests over a web of possibilities. +1 to unit tests for this.

However, writing the tests was not showing the dependencies it would
have to call for them to pass. If they could do that, then the tests could show
me if my architecture could be better.

## Surfacing Dependencies

`RSpec-Mocks` has some great tools for mocking out dependencies. I won't go into
detail here, but I suggest reading some
[documentation](https://www.relishapp.com/rspec/rspec-mocks/docs).

I'm going to go through an example of how one might test a new
Validator which makes sure that the attribute "name" is "Bob."

### Step 1:

Begin to write the unit test for the highest level object.

```ruby
require 'rails_helper'

RSpec.describe MyObject do
  subject { described_class.new }

  describe "#valid?" do
  end
end
```

### Step 2:

Consider your architecture, and possible distribution of responsibilities.
There's no reason MyObject needs to know how to validate. Rather, why not
delegate that so that what the object is and how it validates can be separated?
We don't have that class yet, so let's fill it in with a fake.

```ruby
require 'rails_helper'

RSpec.describe MyObject do

  describe "#valid?" do
    it "should be valid if the given validator returns true" do
      # This creates a totally fake object.
      # If you send messages to it that aren't stubbed it will error.
      validator = double("validator")
      obj = MyObject.new(validator)
      # This lets validator receive .validate(obj).
      # If anything else is passed, it will error.
      allow(validator).to receive(:validate).with(obj).and_return(true)

      expect(obj).to be_valid
    end
    it "should be invalid if the given validator returns false" do
      validator = double("validator")
      obj = MyObject.new(validator)
      allow(validator).to receive(:validate).with(obj).and_return(false)

      expect(obj).to be_valid
    end
  end
end
```

### Step 3

Make tests pass for the initial object.

```ruby
class MyObject
  attr_reader :validator
  def initialize(validator)
    @validator = validator
  end

  def valid?
    validator.validate(self)
  end
end
```

### Step 4

Fill in the collaborator test with a real object.

```ruby
require 'rails_helper'

RSpec.describe MyObject do

  describe "#valid?" do
    it "should be valid if the given validator returns true" do
      # This ensures that BobValidator exists, and you can't allow
      # any methods to be called on it that don't exist for the real class.
      validator = instance_double(BobValidator)
      obj = MyObject.new(validator)
      allow(validator).to receive(:validate).with(obj).and_return(true)

      expect(obj).to be_valid
    end
    it "should be invalid if the given validator returns false" do
      validator = instance_double(BobValidator)
      obj = MyObject.new(validator)
      allow(validator).to receive(:validate).with(obj).and_return(false)

      expect(obj).to be_valid
    end
  end
end
```

### Step 5

Write a test for the collaborator

```ruby
require 'rails_helper'

RSpec.describe BobValidator do
  describe "#validate" do
    context "when given an object with name Bob" do
      it "should return true" do
        validating = instance_double(MyObject)
        allow(validating).to receive(:name).and_return("Bob")
        validator = described_class.new

        expect(validator.validate(validating)).to eq true
      end
    end
    context "when given an object with a different name" do
      it "should return false" do
        validating = instance_double(MyObject)
        allow(validating).to receive(:name).and_return("Joe")
        validator = described_class.new

        expect(validator.validate(validating)).to eq false
      end
    end
  end
end
```

### Step 6

Make the test for the collaborator pass.

```ruby
class BobValidator
  def validate(record)
    record.name == "Bob"
  end
end
```

**Everything Passes!**

## The Alternative

If we weren't to do this, the test for MyObject might look something like this:

```ruby
require 'rails_helper'

RSpec.describe MyObject do
  describe "#valid?" do
    context "when name is Joe" do
      it "should be valid" do
        obj = MyObject.new
        obj.name = "Test"

        expect(obj).to be_valid
      end
    end
  end
end
```

This looks much easier to manage. However, imagine as the class gets bigger and
there are more validations. Do you test them with MyObject? Do you extract
validators? What should the interface be? What was the interface you committed
to in the first place? The power of showing the dependencies can be seen when
the pattern you've chosen is less than ideal:

```ruby
# app/models/my_object.rb
class MyObject
  def valid?
    BobValidator.new.validate(self) && BananaValidator.new.validate(self)
  end
end

# spec/models/my_object_spec.rb
require 'rails_helper'

RSpec.describe MyObject do
  subject { MyObject.new }
  describe "#valid?" do
    context "when name is Bob and banana is true" do
      it "should be valid" do
        subject.name = "Bob"
        subject.banana = true

        expect(subject).to be_valid
      end
    end
    context "when name is not Bob and banana is true" do
      it "should be invalid" do
        subject.name = "Joe"
        subject.banana = true

        expect(subject).not_to be_valid
      end
    end
    context "when name is not Bob and banana is false" do
      it "should be invalid" do
        subject.name = "Joe"
        subject.banana = false

        expect(subject).not_to be_valid
      end
    end
    # etc..
  end
end
```
It's pretty easy to keep this up, and seems to make sense. Set the parameters,
check the result.

If you'd mocked dependencies it would have looked like this:

```ruby
require 'rails_helper'

RSpec.describe MyObject do
  subject { MyObject.new }
  context "when name is not Bob and banana is false" do
    it "should be invalid" do
      bob_validator = instance_double(BobValidator)
      allow(BobValidator).to receive(:new).and_return(bob_validator)
      allow(bob_validator).to receive(:validate).with(subject).and_return(false)

      banana_validator = instance_double(BananaValidator)
      allow(BananaValidator).to receive(:new).and_return(banana_validator)
      allow(banana_validator).to receive(:validate).with(subject).and_return(false)

      expect(subject).not_to be_valid
    end
  end
end
```

This right here is **ridiculous**. Six lines of setup? For two validations? That
hurt to write.

Something must be wrong.

The test said there was something wrong with the architecture, and now there's
an easy place to iterate. Maybe we should pass it in via dependency injection?

```ruby
require 'rails_helper'

RSpec.describe MyObject do
  context "when name is not Bob and banana is false" do
    it "should be invalid" do
      bob_validator = instance_double(BobValidator)
      banana_validator = instance_double(BananaValidator)
      obj = MyObject.new(bob_validator, banana_validator)

      allow(bob_validator).to receive(:validate).with(obj).and_return(false)
      allow(banana_validator).to receive(:validate).with(obj).and_return(false)

      expect(obj).not_to be_valid
    end
  end
end
```

That's better..now we don't have to stub that the items get created. I only want
to inject one thing though, this could easily get out of control. What if there
was a validator which took validators as an argument and returned true if both
of ITS validators were true? Then you just pass that one validator in, and only
have one dependency.

And now you have the composite pattern, and a clean set of dependencies, because
you could easily see the interfaces and dependency graph.
