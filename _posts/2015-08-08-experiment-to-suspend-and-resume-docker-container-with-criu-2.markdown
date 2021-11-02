---
layout: post
title: "Experiment To Suspend/Resume Docker Container With CRIU 2"
date: 2015-08-08 21:51
comments: true
permalink: /blog/experiment-to-suspend-and-resume-docker-container-with-criu-2/
categories:
---

## TL;DR: Simple demo of docker checkpoint & restore


#### Start container
```
$ export cid=$(docker run -d busybox tail -f /dev/null)
```

#### Checkpoint the container
```
$ docker checkpoint $cid
7cc692f22c11
```

#### It's not running anymore
```
$ docker ps --quiet
<No containers shown here>
```

#### Restore the container
```
$ docker restore $cid
7cc692f22c11
```

#### It's running again!!
```
$ docker ps --quiet
7cc692f22c11
```

## What is CR and CRIU?

CR (checkpoint and restart) is a technology that saves the memory state of process into files and resume the processes from the saved state. [CRIU](https://github.com/xemul/criu) is a tool originally developed to CR LXC containers.

Since Docker can run LXC containers, we should be able to CR Docker containers by using CRIU. I've experimented this before and wrote [this post](http://localhost:4000/blog/en/criu/experiment-to-suspend-and-resume-docker-container-with-criu/). Unfortunately, the experiment didn't succeed because CRIU didn't support Docker very well at that moment.

More than a year has passed since then and CRIU team made a lots of effort to support Docker.

In this post, I will show you how Docker checkpoint/restore works with CRIU and why I'm excited about it with use cases.

## Create CRIU vagrant box

CR Docker by using CRIU is still under experiment on libcontainer project, so you need to compile docker with experimental flag enabled. Also, it's not fully merged into Docker, so you need to use a fork of Docker that one of developers in CRIU team created. In addition to that, you also need to compile Kernel with special kernel module enabled.

It's not very fun to do all these things, but don't worry! I've created a Vagrant box which has done all these things and uploaded for you! You just need to download the box and run on your local machine. If you are interested in doing these things, I also put an instruction in appendix.

### Spin up vagrant box
Assuming that the name of VM is **vg-1**, run the following commands.

```
vagrant box add https://atlas.hashicorp.com/kimh/boxes/criu
mkdir <path to vg-1>
cd <path to vg-1>
vagrant init kimh/criu
vagrant up
vagrant ssh
```

That's all you have to do to try CRIU!

## docker restore/checkpoint commands
Docker running on **vg-1** is enabled experimental features and has two commands you've never seen before.

```
$ docker checkpoint --help

Usage:  docker checkpoint [OPTIONS] CONTAINER [CONTAINER...]

Checkpoint one or more running containers
....
```

```
$ docker restore --help

Usage:  docker restore [OPTIONS] CONTAINER [CONTAINER...]

Restore one or more checkpointed containers
....
```

These `checkpoint` and `restore` command use the CRIU command which I compiled and installed on the vagrant box.

```
vagrant@vagrant-ubuntu-trusty-64:~$ criu --help

Usage:
  criu dump|pre-dump -t PID [<options>]
  criu restore [<options>]
  criu check [--ms]
  criu exec -p PID <syscall-string>
  criu page-server
  criu service [<options>]
  criu dedup

....
```

Here's a very simple example of how to CR a Docker container.

#### Starting a container

```
docker run \
  --name np \
  --rm \
  busybox:latest \
  /bin/sh -c \
  'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
```

This command runs a number-printer container which keeps printing incremental numbers.

#### Checkpointing a container

Now, let's checkpoint this container. You can do this with `docker checkpoint` command. Because number-printer container is running in the foreground, do this from other terminal.

```
docker checkpoint np
```

Once you checkpoint the container, you will see that it stops printing numbers. `docker ps` command doesn't show the container anymore.

#### Restoring a container

Let's restore the number-printing container with `docker restore` command.

```
docker restore np
```

If restoring succeeded, the number-printing container again started printing numbers. It's also back in the result of `docker ps`. We succeeded in checkpointing and restoring the container!

## Pause/Unpause VS Checkpoint/Restore
You may say, *"Wait, you can already do this with docker pause/unpause commands"*.

That's right and you don't see a behavior difference between pause/unpause and checkpoint/restore in the previous example. Both commands stop a job of container in the middle and resume later.

The difference is while checkpoint/restart saves the memory state of containers into a disk while pause/unpause doesn't.

You can think of pause/unpause as sending SIGSTOP and SIGCONT to UNIX process. You can run a process in foreground and hitting Ctrl+z will stop the process. You can also restore the process with `fg job-id`.
Pause/unpause commands do the similar thing to running containers.

Checkpoint/restore does more complicated things. It dumps the memory state of running containers into the disk and restore the containers by reading the memory dump.

Because checkpoint/restore saves containers into the disk, we can do more interesting things that we cannot do with pause/unpase. We will see these interesting use cases in the next section.

## Use cases
Here are some use cases of CR Docker containers.

### (1) Resuming long-running containers
You sometimes want to run tasks that take very long time. For example, if you run a program that calculates digits of Pi on a Docker container, the container need to be long-running to calculate trillion digits.

But what if you accidentally shutdown your host machine? This will shutdown the Pi container and you will lose trillion digits that the container has calculated.

CRIU is a great tool to solve this problem. You can periodically checkpoint the container and be prepared for the accident. If the accident happens, you just need to restore the container and resume the calculation in the middle.

### (2) Speeding up slow-start containers
There are applications that take very long time to start. You can speed up such containers by checkpointing the containers after slow applications started.

Here is an example that demonstrates this use case. Let's assume that redis is super slow to start (This is completely not true in reality!!).

We just start a redis container as usual.

```
cid=$(docker run -d redis)
```

Because redis is slow to start in the world of this blog, we need to wait for 20 secs before redis is ready to accept connections. After wating 20 secs, we checkpoint the container.

```
docker checkpoint --image-dir=/tmp/redis $cid
```

We need to restore the redis container into a new container so that you can start multiple containers from the saved container. To do this, you need to use **--force=true** and pass a new container id.

```
redis=$(docker create --name=redis-1) docker restore --force=true --image-dir=/tmp/redis-checkpoint $redis
```

The started container is immediately ready to accept the connection without waiting 20 secs.

The cool thing is that you can repeat the same process to start multiple containers very fast.

```
for i in 1 2 3 4 5; do
  cid=$(docker create --name=redis-$i redis)
  docker restore --force=true --image-dir=/tmp/redis-checkpoint $cid
done
```

The above example theoretically takes 100 secs (20 sec x 5) to finish without CR. With CR, the five containers start in the blink of an eye.

### (3) Container migration
You can do the Docker container migration with CRIU.

To see how this works, you need to run two vagrant VMs. You should be already running **vg-1**, so you need to run one more VM **vg-2**.

```
# Create vg-2
mkdir <path to vg-2>
cd <path to vg-2>
vagrant init kimh/criu
vagrant up
# You need to run at least one container before doing the migration.
# Otherwise, it failed. Probably it's a bug in CRIU or Docker
vagrant ssh -- 'docker run --name=foo -d busybox tail -f /dev/null && docker rm -f foo'
```

Once you run the two vagrant VMs, run number-printer container on **vg-1**.

```
docker run \
  -d \
  --name np busybox:latest \
  /bin/sh -c \
  'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
```

You can see that number-printer container is keep printing number in the background.

```
$ docker logs -f np
1
2
3
4
5
....
```

Now let's migrate the container to **vg-2**. I made a [helper shell script](https://gist.github.com/kimh/79f7bcb195466acea39a) to do this, so you need to download it to your local machine.

```
curl -L -o docker-migrate.sh https://gist.githubusercontent.com/kimh/79f7bcb195466acea39a/raw/370ed974599e2105d56470fade5286050e79afaf/docker-migrate.sh
chmod +x docker-migrate.sh
```

You need to pass three arguments to use the script.

The first argument is the name of the container to migrate. In our case, it's **np**.

The second and third arguments are path to vagrant directories that you migrate from and to.

```
docker-migrate.sh np <path to vg-1> <path to vg-2>
```

Once the script successfully finished, you can go to **vg-2** and check with `docker logs np` command. You should see that the number-printing container resumes printing numbers from the place that it's paused on vg-1.

The migration succeeded!

## Wrapping up
We've seen how to CR Docker container with CRIU and some use cases. I hope people will come up more interesting use cases and develop tools that takes the advantages of CR.

Here are useful resources to learn more about CR Docker with CRIU.

- [Main website of CRIU](http://criu.org/Docker)
- [Docker fork with CRIU support](https://github.com/boucher/docker/blob/cr-combined/experimental/checkpoint_restore.md)
- [Demo of live migration](http://blog.kubernetes.io/2015/07/how-did-quake-demo-from-dockercon-work.html)
