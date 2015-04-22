---
layout: post
title: Avoiding integration Tests
tags: ruby rails rspec capybara unit-tests
---

*To see the previous post in this series please check out [How I Used to
Test]({% post_url 2015-04-21-how-i-used-to-test %})*

## What Now?

Integration tests with Capybara had failed me and in my experience there was only
one other option: *Unit Tests*.

My reasoning went like this: People had been automatically testing applications,
even web applications, long before they could automatically launch a browser and
click around with a clean DSL. How did they do it?

The answer is messages.

## Trust Your Interface

Rather than testing that all the pieces hook together, you test that each of the
parts (units) send the right messages to the right collaborators, and that those
collaborators do what you expect them to do. The web, and Ruby on Rails, has a
fairly specific workflow that you can trust. It goes like this:

```
URL In Bar -> Route -> Controller -> View
```

Fortunately, RSpec has tests for each of these layers.

### Routing Tests

Testing that a route goes to the right place looks like this:

```ruby
# spec/routing/post_routing_spec.rb
require 'rails_helper'

RSpec.describe "post routes" do
  it "should route /posts/1 to the posts controller" do
    expect(get "/posts/1").to route_to :controller => :posts, :action => :show, :id => "1"
  end
end
```

If you run this and the controller doesn't exist, it will tell you. However, it
won't tell you if the action doesn't exist.

### Controller Tests

Now (assuming the above test passes), you have a route that goes to a specific
controller, an action, and sets a parameter. Time to write the controller spec,
to make sure it behaves appropriately.

```ruby
# spec/controllers/posts_controller_spec.rb
require 'rails_helper'

RSpec.describe PostsController do
  describe "show" do
    before do
      @post = Post.create
      get :show, :id => @post.id.to_s
    end
    # Ensures that @post is set for the view to use
    it "should set @post" do
      expect(assigns(:post)).to eq @post
    end
    it "should render the show template" do
      expect(response).to render_template "posts/show"
    end
  end
end
```

This test will complain if a route doesn't exist, if the action doesn't exist,
and will properly fail if either the instance variable or the template isn't
set.

### View Specs

Now we know the controller responds to the right route, populates an instance
variable, and renders the appropriate template. Now how do we prove that the
template shows the right information?

Enter view specs.

```ruby
# spec/views/posts/show.html.erb_spec.rb
require 'rails_helper'

RSpec.describe "posts/show.html.erb" do
  let(:post) { Post.create(:title => "Test") }
  before do
    assign(:post, post)
    render
  end
  it "should show the post's title" do
    # This requires Capybara to be installed,
    # since it's using its matchers.
    expect(rendered).to have_content "Test" 
  end
end
```

## What's Wrong With This?

This leaves just a little bit of a hole - nothing right now actually enforces
that these connections exist. At each layer you're trusting that the other layer
has tests to ensure that it's doing the right thing. There's likely room for
some automated contract verification, but it doesn't exist yet. A simple
integration test which just fails if the pieces fall down is probably in order.

```ruby
# spec/features/show_posts_spec.rb
require 'rails_helper'

feature "posts" do
  background do
    @post = Post.create(:title => "Test")
  end
  scenario "visiting a post page" do
    expect{visit posts_path(@post)}.not_to raise_error
    # May or may not be required depending on your Rails settings.
    expect(page).not_to have_content "Exception"
  end
end
```

Wait! We just wrote an integration test. Weren't we avoiding that? Let's go over
why we avoided integration tests in the first place:

### The Goals

Tests that are:

1. Fast
2. Predictable
3. Informative
4. Reliable

There is only one feature spec - and there will always be only one feature spec
for a given feature. Reducing this number to 1 means that tests will be fast and
there's very little chance the browser will get in the way.

Now we're left with:

1. ~~Fast~~
2. ~~Predictable~~
3. Informative
4. ~~Reliable~~

### Informative

Test Driven Development is supposed to drive good design of an
application. We have a fast test suite now, but at no point did the tests
actually warn us that we may be doing something bad. For instance, the
PostsController could look like this:

```ruby
class PostsController
   def show
     @post = post_factory.new
   end

   private

   def post_finder
     PostFinder
   end

   def post_factory
     PostFactoryFactory.new(post_finder.new(self, params, :with_cached))
   end
end

```

The tests wouldn't complain, so long as those classes exist. The test would be
easy to write, and just as concise.

The reason is this: `the complexity in programming comes from branches and
dependencies`. In these tests we've hidden away that dependencies exist - if
tests are written such that they tell us about them then the complexity becomes
apparent **while writing the test.**

#### Recommended Watching

[Integrated Tests Are A Scam](https://vimeo.com/80533536) - Great talk by J.B.
Rainsberger about why integrated tests eventually calcify and his proposal on
what to do about it.

## Up Next

In the next post I'll talk about writing isolated unit tests as a strategy to
make tests informative, and why that information is important.
