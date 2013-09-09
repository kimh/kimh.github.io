---
layout: post
title: "Using docker to run cucumber tests in parallel"
date: 2013-09-08 21:30
comments: true
categories: 
---
![](/images/homepage-docker-logo.png)
## What is Docker?
You can think of [Docker](https://www.docker.io/) as a wrapper to create and run Linux LXC very easily. In addition to that, Docker comes with very unique technology called "union file system" that allows you to manage the version of your containers with familiar command like _commit_ or _push_ just like you are using git.

LXC is the one of the virtualisation technology that allows you to run a container that is isolated from its host machine. In contrast to hypervisor type virtualisation like Xen Server, the LXC provides lightweight virtual machine. You can run more virtual machines in a host than hypervisor type.

Because of the lightweight nature of LXC, one very good use of it is running many tests.

You want to run your each test in very isolated so that running one test doesn't give other tests any side effects. To do that, running a each test in isolated virtual machine is very natural way. You can do this by using hypervisor type of virtual machine, but it gives you some cost starting and stopping hypervisor type of virtual machine isn't very fast.

With LXC, you can easily archive this because running a machine is as light as running a process!!

If you're using the GitHub for Mac, simply sync your repository and you'll see the new branch.

## Installing Docker
It is pretty easy to install Docker if you are using Ubuntu 12.04.

The below instructions are from [instruction page of Docker](http://docs.docker.io/en/latest/installation/ubuntulinux/) I tried them and it just worked.

```bash
sudo apt-get update
sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
sudo reboot
sudo sh -c "curl https://get.docker.io/gpg | apt-key add -"
sudo sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
sudo apt-get update
sudo apt-get install lxc-docker
```

Now you can provision a container via Docker.

```bash
sudo docker run -i -t ubuntu /bin/bash
```

If you are using Mac, you need to run Docker on vagrant since Docker doesn't suppor Mac at this point. Installing on Mac is explained [here](http://docs.docker.io/en/latest/installation/vagrant/)
However, I recommend you first to try Docker on Ubuntu. I tried Docker on Vagant VM, too, but provisioned container does not run as fast as running on Ubuntu probably because Vagrant is alrady running virtualised resource.

## Setting up test environment
As I mentioned earlier, Docker provides very similar interface to git. You can pull docker images from [public Docker repository](https://index.docker.io/) and use it to make your own images and then push.
Since I will run cucumber test, you need a container that has environment to run ruby program. Although you can create such container very easily, let's save time by using the image that I created before.

```bash
# You need to be root to use Docker
sudo -s

# Pulling images to your machine
docker pull kimh/ruby-base

# Now let's see if it works
docker run kimh/ruby-base echo "Running on Docker"
```

Let's login to the container you just pulled and install our test app.

```bash
# Login to the container
docker run -i -t kimh/ruby-base /bin/bash
cd /git
git clone https://github.com/kimh/docker_demo
cd docker_demo/ci_app
bundle install
```

Now, we have done time-consuming operation, so we want to save the state of the container. We will commit the change of container and save to images

```bash
# Dont't type exit on logged in container. If you do, your changes will be discarded!!
# Instead, you type Ctrl+p then Ctrl+q to quit from the console.
Ctrl+p
Ctrl+q
# First, you need to know the id of container
docker ps # In my case, it gives me 23fd82dcc088. Maybe different in different env?
docker commit 23fd82dcc088 kimh/ruby-base
```

You are ready to run the cucumber test on a container. Let's start only running one container.

```bash
docker run kimh/ruby-base /bin/bash -c "
  source /etc/profile
  cd /git/docker_demo/ci_app
  export LC_CTYPE="ja_JP.UTF-8"
  export RAILS_ENV=test
  bundle exec rake cucumber
"
```

With the script above, Docker runs kimh/ruby-base and execute bash script which runs cucumber test.

Finally, let's run tests in paralell. The idea is running multiple containers and each container run one cucumber test. In this case, I will run 5 containers in parallel.

```bash
DID=""
container="kimh/ruby-base:latest"
dir="/git/docker_demo/ci_app"

for feature in {0..4}
do
  DID=$DID" "`sudo docker run -d $container /bin/bash -c "
  source /etc/profile
  cd $dir
  export LC_CTYPE="ja_JP.UTF-8"
  export RAILS_ENV=test
  bundle exec rake cucumber
  "`
done
docker wait $DID
```

There are two things to note in above script.

First, the script stores the ID's of running containers into DID variable. I need the variable because I want to know the status code (whether test succeeds or fails) later.

Second, by calling _docker wait_ against container ID's, the shell reports me the status code of each container and blocks until all container's execution finishes.

You should see consecutive status code of 0 since all tests should pass. (cucumber returns 0 for success).

# Wrapping up
You learned basics things about Docker and how to use it to run cucumber tests in parallel. Since each test is invoked by a container,
tests are run in parallel and isolated environments. In this example, we only ran the same tests, so it is not very useuful, but you see how you can passes different tests
to each container and run them at once.

# Some disappointed outcome...
Our demo cucumber tests are very simple and fast to finish, so it looked like Docker runs tests very quickly. However, when I run tests in real project that I am working now,
it does not run as fast as I expected. I can still run 50 containers at the same time in Linux physical machine, but took about 1 hour to finish them and looks like
host machine is not allocating its resources enough to containers. My next task is running containers faster somehow.

I will update the blog once I found a way!!
