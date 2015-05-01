---
layout: post
title: Isolated Testing
tags: ruby rails rspec capybara unit-tests mocks
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

Let's go through a quick example of the two methods of testing. This is a test
to make sure that an object is valid if its name is "Bob" and it has "banana"
set to true.

```ruby
require 'rails_helper'

RSpec.describe MyObject do
  describe "#valid?" do
    context "when name is Bob" do
      it "should be valid" do
        obj = MyObject.new
        obj.name = "Bob"

        expect(obj).to be_valid
      end
    end
  end
end
```

No dependencies have been mocked out and there's no clear interface about what's
going on in the background. However, it was -very- easy to write.

Imagine the class getting bigger and
there being more validations. Do you test them with MyObject? Do you extract
validators? What should the interface be? What was the interface you committed
to in the first place?

The power of showing the dependencies can be seen when
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

This right here is **ridiculous**. Six lines of setup? For one test with two validations? That hurt to write.

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
have one dependency. Then you just test each collaborator, and the behavior of
the dependency which takes dependencies, and you're good!

And now you have the composite pattern, and a clean set of dependencies, because
you could easily see the interfaces and dependency graph.

## Why I Do This

I've only been a professional programmer for a few years. Before that I had very
little formal training, little understanding of patterns and practices, and had
a flawed idea of what "good" architecture was. I'd always learned via one simple
method: **I beat my head against something until it finally works.**

That's great for hacking, but not for architecture. The wall to hit against was
too far away, required too much experience to find, and was often hazy -
determining why something was a good practice took a tremendous time investment.

Bringing the interface and dependencies to the front in my tests DEFINES the
wall and brings it closer. I get nearly immediate feedback on whether or not
I've chosen correctly. In this way I can beat my way towards better software.

`Isolated testing enables me to develop better applications than my experience says I
should be able to.`
