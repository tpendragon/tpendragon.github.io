---
layout: post
title: Response to Nothing is Something
tags: ruby rails composition
---

## Nothing is Something

For RailsConf 2015 [Sandi Metz](https://twitter.com/sandimetz) gave a fantastic
talk with two parts: the first discussed the null object pattern and the second
talked about composition over inheritance. This post is some of my thoughts on
the second half.

However, before I get started, please go watch her talk: [Nothing is
Something](http://confreaks.tv/videos/railsconf2015-nothing-is-something)

## Composition Over Inheritance

That talk is one of the best and most clearly communicated examples of the
benefits of composition I've seen to date. However, I'm writing this because I
feel like things could have gone just a step further.

At the end of the talk we're left with code that looks like this:

```ruby
class House
  attr_reader :data, :formatter
  DATA = [
    "the horse and the hound and the horn that belonged to",
    "the malt that lay in",
    "the house that Jack built"
  ]
  def initialize(orderer: DefaultOrder.new, formatter: DefaultFormatter.new)
    @formatter = formatter
    @data = orderer.order(DATA)
  end

  def recite
    (1..data.length).map {|i| line(i)}.join("\n")
  end

  def line(number)
    "This is #{phrase(number)}.\n"
  end

  def phrase(number)
    parts(number).join(" ")
  end

  def parts(number)
    formatter.format(data.last(number))
  end
end

class DefaultFormatter
  def format(parts)
    parts
  end
end

class EchoFormatter
  def format(parts)
    parts.zip(parts).flatten
  end
end

class RandomOrder
  def order(data)
    data.shuffle
  end
end

class DefaultOrder
  def order(data)
    data
  end
end
```

## Define Responsibilities

This is great code, but in a step towards refactoring even further it's
important to define responsibilities.

`House` should have one responsibility: recite. Take a dataset, use something to order
and format it, and then output it. However, in order to fulfill that
responsibility it must also know how to join an array of terms such that's
recitable and extract a certain piece of those terms for recitation.

`Formatters` take a phrase represented as an array of terms and formats them to
be joined by a space.

`Orderers` take a full dataset and orders them.

## Analyze Dependencies

`Formatters` have one dependency: an array of terms.

`Orderers` have one dependency: an array of terms to order.

`House` has two dependencies: a formatter and an orderer.

## Reduce Responsibilities

The Single Responsibility Principle says that each object should only have one
reason to change, and `House` has three right now. Let's see if we can get it
down.

First up is to extract the format piece.

```ruby
class HouseData
  attr_reader :data, :formatter
  def initialize(data:, formatter: DefaultFormatter.new)
    @data = data
    @formatter = formatter
  end

  def length
    data.length
  end

  def phrase(number)
    parts(number).join(" ")
  end

  private

  def parts(number)
    formatter.format(data.last(number))
  end
end
class House
  attr_reader :data, :formatter
  DATA = [
    "the horse and the hound and the horn that belonged to",
    "the malt that lay in",
    "the house that Jack built"
  ]
  def initialize(orderer: DefaultOrder.new, formatter: DefaultFormatter.new)
    @formatter = formatter
    @data = HouseData.new(data: orderer.order(DATA), formatter: formatter)
  end

  def recite
    (1..data.length).map {|i| line(i)}.join("\n")
  end

  def line(number)
    "This is #{data.phrase(number)}.\n"
  end

end
```

Now House just knows how to take something that responds to #phrase and #length
and recite them. HouseData knows what it takes to turn an array of terms into
phrases.

## Analyze Dependency Usage

If you look at the constructor for `House` you'll notice that orderer is only
used to initialize data and formatter is only used to be passed off into
`HouseData`. A good rule is that if dependencies are only used in constructors,
extract them and pass the good value in as the parameter instead. This makes
your class just that much more flexible.

```ruby
class HouseData
  attr_reader :data, :formatter
  def initialize(data:, formatter: DefaultFormatter.new)
    @data = data
    @formatter = formatter
  end

  def length
    data.length
  end

  def phrase(number)
    parts(number).join(" ")
  end

  private

  def parts(number)
    formatter.format(data.last(number))
  end
end
class House
  attr_reader :data, :formatter
  DATA = [
    "the horse and the hound and the horn that belonged to",
    "the malt that lay in",
    "the house that Jack built"
  ]
  def initialize(data: data)
    @data = data
  end

  def recite
    (1..data.length).map {|i| line(i)}.join("\n")
  end

  def line(number)
    "This is #{data.phrase(number)}.\n"
  end
end

# Initialization
house_data = HouseData.new(data: House::DATA)
House.new(house_data).recite
```

You'll notice that since you're passing in the data, there's no need for a
"default orderer" - just pass in the data.

## Finale

Now you're left with this:

```ruby
class HouseData
  attr_reader :data, :formatter
  def initialize(data:, formatter: DefaultFormatter.new)
    @data = data
    @formatter = formatter
  end

  def length
    data.length
  end

  def phrase(number)
    parts(number).join(" ")
  end

  private

  def parts(number)
    formatter.format(data.last(number))
  end
end

class House
  attr_reader :data
  DATA = [
    "the horse and the hound and the horn that belonged to",
    "the malt that lay in",
    "the house that Jack built"
  ]
  def initialize(data: data)
    @data = data
  end

  def recite
    (1..data.length).map {|i| line(i)}.join("\n")
  end

  def line(number)
    "This is #{data.phrase(number)}.\n"
  end
end

class DefaultFormatter
  def format(parts)
    parts
  end
end

class EchoFormatter
  def format(parts)
    parts.zip(parts).flatten
  end
end

class RandomOrder
  def order(data)
    data.shuffle
  end
end

# Initialization
house_data = HouseData.new(data: House::DATA)

random_house_data = HouseData.new(data: RandomOrder.new.order(house_data.data))

echo_house_data = HouseData.new(data: house_data.data, formatter: EchoFormatter.new)

random_echo_house_data = HouseData.new(data: random_house_data.data, formatter: echo_house_data.formatter)

# Recitation

House.new(data: house_data).recite
```

Everything has one responsibility and is completely flexible. The only other
thing I might do is simplify the interface for Formatter and Order. Something
like

```ruby
class EchoFormatter
  def self.call(parts)
    parts.zip(parts).flatten
  end
end

class DefaultFormatter
  def self.call(data)
    data
  end
end

class RandomOrder
  def self.call(data)
    data.shuffle
  end
end
```
 would make it easier to remember what to do to use the dependencies.

### Compliments

I just want to leave a final note of thanks to Sandi Metz for an excellent talk.
It provided a much better example of composition over inheritance for my team
than I've seen before. Highly recommended.
