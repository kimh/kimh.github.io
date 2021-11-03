---
layout: post
title: "Creating Web Application with Ruby Lotus"
date: 2014-09-11 18:40
comments: true/blog/en/lotus/creating-web-application-with-ruby-lotus/
permalink: /blog/en/lotus/creating-web-application-with-ruby-lotus/
categories:
- en
- lotus
---

![](/assets/lotus.jpeg)

***WARNING***  This is not your article if you are looking for information about the collaboration software made by IBM.
Instead, this article is about [Lotus](http://lotusrb.org/), a new web framework written in Ruby.

## Contents
* [What is Lotus](#what_is_lotus)
* [Why Lotus instead of Rails](#why_lotus_instead_of_rails)
* [Creating one file aplication](#creating_one_file_application)

<a id="what_is_lotus"></a>
## What is Lotus
[Lotus](http://lotusrb.org/) is a web framework that is being developed by relatively small team of [lotus team](https://github.com/lotus).
You can use Lotus and build a complete MVC web application just like Rails.
Lotus is still under active development and not production ready as of Sep, 2014, but you can definitely use it to create a web application.

I fell in love with Lotus at first sight of the mission that Lotus tries to archive in the project page.

The page reads

>Lotus is lightweight, fast and testable. It aims to bring back Object Oriented Programming to web development, leveraging on a stable API, a minimal DSL, and plain objects.

I felt this is what I was looking for (explained more later) and decided to use Lotus to create a small API server in my private project.
Since there are not much documentation and information about Lotus in the wild yet, I sometimes had a hard time to figure out how to use it.
But, I am recently getting used to the ***Lotus way*** so I'd like to share them in this and subsequent posts.

<a id="why_lotus_instead_of_rails"></a>
## Why Lotus instead of Rails
Recent applications are built in modular way more than ever before.
Whether you call this type of application architecture SOA or microservices,
it is true that many recent great projects [(my favorite example is Fylnn)](https://github.com/flynn) are taking this design approach.

There are a few major benefits of taking this approach

 - more testability
 - more portability
 - more reusability
 - easier deployment

You will realize that it is not easy to accomplish all of these with Rails.
Rails is definitely great, but the framework stack is huge and lots of things are built in. After all, Rails is a big framework, so not a good option if you want to create lots of small components.

You may think there are small frameworks such as [Sinatra](https://github.com/sinatra/sinatra) or [rails-api](https://github.com/rails-api/rails-api).
Yes, Sinatra is lightweight, but I recently prefer to ***pure ruby code*** than DSL because it gives me steep learning curve (meaning easy to learn).
To be honest, I never tried rails-api by myself, but I am suspicious that it is lightweight because the base is still Rails. Let me know if you have different opinions.

As you can tell from [Lotus github page](https://github.com/lotus), it is made of many components. You can easily bring one of components into your application.
For example, you can just grab [lotus-router](https://github.com/lotus/router) and mixin to your Rack application to handle http request.

Although, Lotus is made modular way, you can still use it as fullstack web-framework with relatively small amount of codes.
Apparently, Lotus steals many good designs from Rails such as CoC and that allows you to build applications easy.

So, my point in this section is this: **Lotus is flexible but easy to use, so why not give it a shot?**

Hopefully, this article helps you starting Lotus.

<a id="creating_one_file_application"></a>
## Creating one file application
Let's make our first Lotus application. We will follow the example of [one file application](https://github.com/lotus/lotus#one-file-application).

First, create a project directory. I am assuming that you set your current directory to this directory in the subsequent instructions.

```
mkdir onefileapp && cd onefileapp
```

Lotus does not publish official gem as far as I know. There is [one](https://rubygems.org/gems/lotusrb) here but the last update was on Jan, 2014.
Current master branch is far ahead of this release. So, clone the Lotus project in order to install from source.

```
git clone https://github.com/lotus/lotus.git
```

We need bundler to manage gem dependancy. Let's create a Gemfile.

```
bundle init
```

Edit your Gemfile. Change `<your-path-to-lotus-repo>` to the directory where you clone Lotus repo.

```ruby
source "https://rubygems.org"
gem 'lotusrb', :path => <your-path-to-lotus-repo>
```

Then install gems.

```
bundle install --path vendor/
```

Now, we can start writing our application which is just one file. Save below codes as ***config.ru***.

***config.ru***
```ruby
require 'lotus'

module OneFile
  class Application < Lotus::Application
    configure do
      routes do
        get '/', to: 'home#index'
      end
    end
    load!
  end

  module Controllers
    module Home
      include OneFile::Controller

      action 'Index' do
        def call(params)
        end
      end
    end
  end

  module Views
    module Home
      class Index
        include OneFile::View

        def render
          "Hello, Lotus"
        end
      end
    end
  end
end

run OneFile::Application.new
```

You can run the app with rackup command.

```
bundle exec lotus server
```

Successfully run? Then, access http://localhost:2300 from your browser. You should see `Hello, Lotus`.

Let me explain what is doing.

First, you will notice that classes inside `Controllers` and `Views` modules not using inheritance.
One important philosophy of Lotus is this:`include a module and implement a minimal interface.`
This philosophy encourages developers to use only what you need with mixin.

Now let's look at `Application` class. In our application, the class only configures routes.
We use `get` method to configure a route for http `GET /` method that uses `Home::Index` controller.
Here we only configure `get` http method, but you can configure other methods easily such as these:

```ruby
routes do
  post   '/books',             to: 'book#create'
  put    '/books/:id',         to: 'book#update'
  delete '/books/:id',         to: 'book#destroy'

  # You can also define one liner response
  get    '/ping',              to: ->(env) {[200, {}, ['pong']]}
end
```

Next, let's look at `Controllers`. We are not doing anything but defining `action`.
What is `action`? `action` is the HTTP endpoint where you can handle incoming request and creating response.
This is also the place where you can implement your business logic. I think it is safe to think that the responsibility of Lotus action is very similar to Rails controller.
We define `Index` action so that we can use it from our router code that we looked at previously.

Last thing to look at is `Views`. The responsibility of the class is not the same as the Rails view class.
In Rails, you write codes that is actually rendered by browser (if html format) in view. This is done by `Template` in Lotus which I don't cover in this post.
In Lotus, the responsibility of view is more like of presenter which does not come with Rails by default (you can use gems such as Draper to implement presetation layer in Rails, too.)
What presenter does is receiving data from controller and abstracts them to template layer.
In this way, your template layer gets clean and can focus on redning content.

Let's go back to our code. Here we define `render` method and simply print `Hello, Lotus` message.

This is neither interesting nor useful. We will make view to interact with controller by via data.

Let's modify your controller.

```ruby
module Controllers
  module Home
    include OneFile::Controller
    action 'Index' do

      expose :time

      def call(params)
        @time = Time.now
      end
    end

  end
end
```

Now, we see two new things `expose` and `call`.

To pass data from controller to view, you need to manually expose what you want to pass.
Again, here you see Lotus philosophy: `only use what you need`.

`call` is the entry point of http request. As I mentioned earlier, you can write business logic as well as response handling codes here.

Let's modify your view to get data from the controller.

```ruby
module Views
  module Home
    class Index
      include OneFile::View

      def render
        "Current time: #{time}"
      end
    end
  end
end
```

Since our controller exposes `@time`, you can access the data via `time` from your view.
Now, restart your Rack process and access from your browser. Now you should see something like this: `Current time: 2014-09-11 23:18:30 +0900`

## Wrap up
Did you see it is quite easy to write Lotus application?

Ok, I can hear your voice: *this example is too simple. I want to see real Lotus application.*

Yes, let me do that in the next post. I thought I can do that in the same post, but writing this takes more time than I thought...

Next post will be something like this: ***Creating Full Stack Web Application with Lotus***
