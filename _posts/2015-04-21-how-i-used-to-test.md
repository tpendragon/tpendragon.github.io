---
layout: post
title: How I Used to Test
tags: ruby rails rspec capybara
---

## Starting Out

I'll always remember my first interview for a real development position. It was
for a developer position here at Oregon State University as a PHP programmer. I was
asked to report on a project I had completed in the past and then asked a series
of questions about how I develop. The one that stands out the most went like so:

**Interviewer**: `How do you test that your applications work?`

There was only one answer, obviously.

**Me**: `Well, first I open the app in a browser and I...`

You can see where I was going. It wasn't the answer they were looking for and I
could feel it. Time to figure out what went wrong.

## Dipping The Toes

I got my first full time job shortly afterwards, having answered a similar question by saying "well, I'm familiar with *Test Driven Development*, but I've never done it."

Fortunately they were nice.

I knew I needed to figure out how to test, and they had me learning Ruby, so it
was time to explore the options:


### Potential Tools 

1. Test::Unit
1. RSpec
1. Capybara
1. Cucumber

Alright, lots of people are using RSpec, I quickly decided I didn't like
Cucumber, and then...there was Capybara.

## Capybara

`Capybara` is a testing framework in ruby which tests web applications by
simulating a browser. It goes to pages, clicks through forms, and validates that
text is on the page. An automated Capybara test (back then) looked like this:

```ruby
require 'spec_helper'

describe "viewing posts" do
  before do
    @post = Post.create(:title => "My Title")
    visit "/posts"
  end
  it "should display all the post titles" do
    expect(page).to have_content("My Title")
  end
end
```

Finally something which automated everything I would normally do by hand! Things
were taking seconds, rather than minutes, to make sure they work. If I wrote a
whole suite of these *Integration* tests, then I could be sure the site was
working and could refactor my code at will.

### Some Problems

A few little problems popped up.

1. Testing Was Slow
   
     Testing my application was taking something like 30-60 minutes to run through
the full set of tests.

     Oh well, I could work around that - throw in a Rails preloader like
[spring](https://github.com/rails/spring), run only the specs I care about while
developing, and then things only hurt when I had to make sure the all the tests
were done. **Solved**

2. Test Driven Development
   
     I kept reading that I should be writing tests first, and it was hard. Often
times the last thing I knew was what the UI would look like - how could I write
a test that included information about it? Oh well, do my best. **Not So
Solved**

3. Code Design

     This felt like the biggest problem to me - I kept hearing that tests should
make me recognize when my code needed help, but none of my tests were saying
anything about the code - just the application that it created. I could refactor
without changing tests, yes, but I could also **NOT** and it would all feel the
same. **Not Solved**

4. Unreliable

     Sometimes I'd run my tests and they'd fail - but seemingly randomly. I
couldn't trust the application unless they passed, but there was no way to find
out why they were failing (often times it'd be a race condition within capybara
or my application logic that couldn't be reproduced by a user), so all I could
do was restart the tests and wait for them to pass - another 30-60 minutes down
the drain.

## Now What?

I knew I had some problems, and I knew if I wanted stable, reliable, functional
applications that I'd have to solve them. I needed automated tests which were:

1. Fast
2. Predictable
3. Informative
4. Reliable

Integration tests had failed me, there was only one other option - *Unit Tests*

My next post will be all about the resources I used to get where I am now, and
what I think the solution to the problems above are.
