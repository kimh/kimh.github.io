---
layout: post
title: "Introducing Baleen Gem"
date: 2014-01-11 14:59
comments: true
categories:
- en
- docker
---
![key is blade](/images/keyblade.jpg)

## Motivation to write this gem
[Docker](http://www.docker.io/) is by far my most favorite project in the past 6 months. Many people already begun using docker to run their tests. In most cases, bash is used to start and stop containers, [which I also did](http://localhost:4000/blog/en/docker/using-docker-to-run-cucumber-tests-in-parallel/), but I wanted go further with [Docker Remote API](http://docs.docker.io/en/latest/api/docker_remote_api/).

Also, I had a problem of very slow tests in my working company. We had many cucumber tests that are run one by one and normally it takes about 2 hours to run all of them. Also, we had many random failures in these tests because we run the tests by the way that each tests affect each other.
To solve these problems, I wanted the two things: **parallelism** and **isolation**. These are reasons to write [baleen](http://rubygems.org/gems/baleen)

Baleen is a gem that

* runs ruby tests on Docker container
* runs tests in parallel
* runs each tests in isolated environment

For the basic usages and options, please see [github repo](https://github.com/kimh/baleen). In this post, I will write about internals of the gem.

## Key libraries

### docker-ruby
baleen server makes API call to Docker. To make API call, I use ruby gem that wraps the call. Currently, there are two gems of docker ruby api.

* [swipely / docker-api](https://github.com/swipely/docker-api)
* [geku / docker-client](https://github.com/geku/docker-client)

Unfortunately, docker-client is not under active development, so I will recommend to use docker-api which is also used in baleen.

### Celluloid
To run tests in parallel, we have to run containers asynchronously.

Fortunately, we have [Celluloid](https://github.com/celluloid/celluloid) to get the power of asynchronous very easily. How celluloid being used is explained later.

There are a few ways to make ***wait*** asynchronous,
but I use what called [Futures](https://github.com/celluloid/celluloid/wiki/futures) of celluloid.

## How baleen interacts with docker
This is rough image that shows how baleen interacts with Docker. In the image, you can see how the request made by baleen client is processed by baleen server that makes API calls to Docker host.


![basic flow of interaction](/images/basic-flow.png)

The flow is what happens when you run

```bash
baleen cucumber --image my-test-runner --files features/
```

Let's go one by one.

**1.** baleen client asks server to start running cucumber test

This part is nothing interest, so make it brief. baleen client makes json request to baleen server and asks to run cucumber for features/test1.feature.

**2.** Breaking up features/ directory

The basic idea of baleen is to let each container running one test. In other words, we will run the same number of containers as the number of test files. In our case, we want do something like this in each container.

```bash
# container 1
baleen cucumber --image my-test-runner --files features/test1.feature
# container 2
baleen cucumber --image my-test-runner --files features/test2.feature
...
# container 10
baleen cucumber --image my-test-runner --files features/test10.feature
```

Since baleen can't know how many feature files exist under features directory, baleen server first needs to breaks up the directory into single file. To do this, I run one container that executes below command.

```bash
find ./features | grep ".feature"
```

You can get the output of the command by calling *Container#attach* method.

```ruby
stdout, stderr = *@container.attach(:stream => false, :stdout => true, :stderr => true, :logs => true)
# => [["./features/test1.feature\n./features/test2...]]
```

**3.** Running containers

Now, you know what test files exist under *./features* directory, so you can pass each file to a container and run them in parallel. However, we are not going to run all of them at once.
Although LXC is very lightweight, each container requires certain memory. Therefore, we need to adjust the number of containers to run at one time.

In baleen, this can be configured by *--concurrency* option.

```bash
baleen cucumber --image my-test-runner --files features/ --concurrency 4
```

If you specify concurrency 4, baleen will create and start 4 containers, wait all of them to execute its test, and then move to next 4 containers.

You can fire containers easily. Just call ***Container.start*** method.

```ruby
containers.each do |container|
  container.start
end
```

***start*** method asks Docker to run the container and immediately returns.

**4.** Wait containers asynchronously

When you start containers, you have to know when it finishes and stops. You can detect when a container finishes by using ***Container#wait***.

However, there is a problem.
Unlike ***run***,***wait*** blocks until container stops and returns. This behavior isn't good for parallelism since you don't want to be blocked at one container.
What you want containers to do is notifying you when they finish and stop so that you can inspect the results of finished containers.

This can be archived by putting ***start*** and ***wait*** into a method called ***run*** and calling it asynchronously with the help of celluloid.

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
results = []
containers.map {|container| container.future.run}.each do |actor|
  results << actor.value
end
```

Note that the receiver of ***run*** is not the instance of container, but ***future***. This method magically makes preceding method call asynchronous. In our case, ***run*** is called asynchronously.
Therefore, even if ***run*** is blocking method, ***{|container| container.future.run}*** immediately moves to next loop without waiting containers to finish.

Methods called with ***future*** returns ***#<Celluloid::Future>*** that has ***value*** method. ***value*** returns the return value of ***future*** whenever it finishes.
So in our case, ***value*** returns the return value of ***run*** which is the hash of status code, stdout, and stderr. Now we could accomplish what we wanted. We could run multiple containers at one time and retrieve results whenever they finish.

In summary, we archived running containers in parallel by

* Implementing ***run*** method that calls ***start*** and ***wait***
* Calling ***run*** asynchronously with future of celluloid
* Retrieving return values with ***value***

