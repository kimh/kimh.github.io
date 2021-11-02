---
layout: post
title: "Running Docker Containers Asynchronously with Celluloid"
date: 2014-01-11 14:59
comments: true
permalink: /blog/en/docker/running-docker-containers-asynchronously-with-celluloid/
categories:
- en
- docker
---
![](/images/parallel.jpg)

## Baleen
I wrote a ruby gem called [baleen](http://rubygems.org/gems/baleen).
Baleen runs your cucumber tests by using docker containers in parallel. By using docker, baleen archives parallel test execution as well as providing completely isolated environments for each test runner.

For the basic usages, please see [github repo](https://github.com/kimh/baleen). In this post, I will write how balenn runs docker containrs in parallel by using remote API of docker.

## API Library
baleen server makes API call to Docker. To make API call, I use ruby gem that wraps the call. Currently, there are two gems of docker ruby api.

* [swipely / docker-api](https://github.com/swipely/docker-api)
* [geku / docker-client](https://github.com/geku/docker-client)

Unfortunately, docker-client is not under active development, so I will recommend to use docker-api which is also used in baleen.

## Celluloid
To run tests in parallel, we also have to run containers in parallel. To archive this, the operation of running containers should be done asynchronously and not blocking subsequent containers.

Fortunately, we have [Celluloid](https://github.com/celluloid/celluloid) to get the power of asynchronous very easily.

## Brief Explanation of How Baleen Works
To get the context, let me quicly go over how baleen works with docker.

This is rough image that shows how baleen interacts with Docker. In the image, you can see how the request made by baleen client is processed by baleen server that makes API calls to Docker host.


![basic flow of interaction](/images/basic-flow.png)

The flow is what happens when you run

```bash
baleen cucumber --image my-test-runner --files features/
```

The above command does the following things.

**1.** Baleen client asks baleen server to run your tests.
**2.** Baleen server receives the request and make API call to docker
**3.** Docker creates and starts containers accordingly
**4.** Baleen server retrieves results and pass them back to baleen client

In this post, we are interested in a bit of **2.** and more in **3.** and **4.**

## Making API Call To Run Containers

Making container object is very straightforward if you use docker-ruby.

```ruby
image = "my-app-runner"
test_files = ["test1.feature", "test2.feature", "test3.feature"]
containers = test_files.map do |test_file|
  Docker::Container.create('Cmd' => ["bash", "-c", "cucumber features/#{test_file}"], 'Image' => image)
end
```

Note that we are passing different test files to each container. The basic idea of baleen is let each container to run a single test. Therefore, if we have three test files, then we need to create three containers.

To start container, just call ***Container.start*** method like this.

```ruby
containers.each do |container|
  container.start
end
```

The important point here is ***start*** method immediately returns once it asks docker to start containers. Therefore this loop finishes very quickly.

## Running containers asynchronously

When you start containers, you have to know when it finishes otherwise you don't know when you can retrieve the results of containers. You can detect when a container finishes by using ***Container#wait***.

However, there is a problem.
Unlike ***start***,***wait*** blocks until the method returns. This behavior isn't good for our purpose (parallel tests running) since you don't want to be blocked at one container.

If you are blocked, you have wait each container one by one which is very inefficient.

![Waiting containers synchronously](/images/synchronous_wait.png)

In this diagram, you can see that you have wait each container that takes **90 sec** including the time to start containers.

What you want to do is waiting all containers at the same time and ask containers to notify you when they finish so that you can retrieve the results of containers.

![Waiting containers asynchronously](/images/asynchronous_wait.png)

In this diagram, the maximum time is the time for waiting the slowest container, which is 30 sec. This makes **60 sec** in total which is faster than synchronous version.

How can we archive this? This is where celluloid comes into play. We will put ***start*** and ***wait*** into a method called ***run*** and calling it asynchronously with the help of celluloid.

Here is how ***run*** method looks like (some codes are omitted)

```ruby
def run
  @container.start
  @container.wait(600)
  stdout, stderr = *@container.attach(:stream => false, :stdout => true, :stderr => true, :logs => true)

  return {
    status_code: @container.json["State"]["ExitCode"],
    stdout: stdout,
    stderr: stderr,
  }
end
```

At this point, ***run*** still blocks. However, by using [Futures](https://github.com/celluloid/celluloid/wiki/futures) of celluloid, you can make it asynchronous method.

The code looks something like this.

```ruby
containers.map {|container| container.future.run}
```

Note that the receiver of ***run*** is not the instance of container, but ***future***.

This method magically makes preceding method call asynchronous. In our case, ***run*** is called asynchronously.
Therefore, even if ***run*** is blocking method, ***{|container| container.future.run}*** immediately moves to next loop without waiting containers to finish.

But how can we be notified when containers finish? Well, we actually even don't have to get notified because celluloid does it for you. This is done by calling ***value*** method.
Let's retrieve results by modifying previous codes a bit.

```ruby
results = []
containers.map {|container| container.future.run}.each do |actor|
  results << actor.value
end
```

Methods called with ***future*** returns ***#<Celluloid::Future>*** that has ***value*** method. ***value*** returns the return value of ***future*** whenever it finishes.

So in our case, ***value*** returns the return value of ***run*** which is the hash of status code, stdout, and stderr. Now we could accomplish what we wanted. We could run multiple containers at one time and retrieve results whenever they finish.

In summary, we archived running containers in parallel by

* Implementing ***run*** method that calls ***start*** and ***wait***
* Calling ***run*** asynchronously with future of celluloid
* Retrieving return values with ***value***
