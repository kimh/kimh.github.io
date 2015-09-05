---
layout: post
title: "experiment-to-suspend-and-resume-docker-container-with-criu-2"
date: 2015-08-08 21:51
comments: true
categories: 
---

## TL;DR: Simple demo of docker checkpoint & restore


#### Start container
```
$ export CID=$(docker run -d busybox tail -f /dev/null)
```

#### Make sure it's running
```
$ docker ps --quiet
7cc692f22c11
```

#### Checkpoint the container
```
$ docker checkpoint $CID
7cc692f22c11
```

#### Make sure it got checkpoint
```
$ docker ps --quiet
```

#### Restore the container
```
$ docker restore $CID
7cc692f22c11
```

#### It's running again!!
```
$ docker ps --quiet
7cc692f22c11
```

I have tried to checkpoint/restore Docker container in [this post](http://localhost:4000/blog/en/criu/experiment-to-suspend-and-resume-docker-container-with-criu/) before. Unfortunately, I couldn't do that because CRIU didn't support Docker very well at that moment.
More than a year has passed since then and CRIU team made a lots of effort to support Docker. In this post, I will show you how Docker checkpoint/restore works with CRIU and why it's so cool with working examples.

## Create CRIU vagrant box

CR Docker by using CRIU is still under experiment on libcontainer project, so you need to compile docker with experimental flag enabled. Also, it's not fully merged into Docker, so you need to use a fork of Docker that one of developers in CRIU team created. In addition to that, you also need to compile Kernel with special kernel module enabled.

It's not very fun to do all these things, but don't worry! I've created a Vagrant box which has done all these things and uploaded for you! You just need to download the box and run on your local machine. If you are interested in doing these things, I also put an instruction in appendix.

### Spin up vagrant box
Just run the following command.

```
vagrant init kimh/criu; vagrant up --provider virtualbox
```

This will download the vagrant box from [here](https://atlas.hashicorp.com/kimh/boxes/criu).

That's all you have to do to try CRIU!

## docker restore/checkpoint commands
Docker running in the vagrant box is enabled experimental feature and has two commands you've never seen before.

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

These `checkpoint` and `restore` command use CRIU and do CR things. Here's a very simple example of how to use the commands.

#### Starting a container

```
$ docker run \
    --name number-printer \
    --rm \
    busybox:latest \
    /bin/sh -c \
    'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 3; done'
```

This command runs number-printer container which keeps printing a number to stdout in every 3 seconds.

#### Checkpointing a container

Now, let's checkpoint this container. You can do this with `docker checkpoint` command. Because number-printer container is running in the foreground, you need to open one more ssh session.

```
$ docker checkpoint number-printer
```

Once you checkpoint the container, you will see that it stops printing numbers. `docker ps` command doesn't show the container anymore. Now, our number-printer container is frozen and its memory state is dumped into disk. Checkpoint succeeded!

#### Restoring a container

Let's resume the number-printing container with `docker restore` command.

```
$ docker restore number-printer
```

If restoring succeeded, the number-printing container again started printing numbers. It's also back in 'docker ps`. It's cool, isn't it?

## Pause/Unpause VS Checkpoint/Restore
Docker already has pause/unpause commands. You may wonder what's difference between pause/unpause and checkpoint/restore.

You will not see a behavior difference in the previous example. Both commands stop a job of container in the middle and resume later.

The difference is while checkpoint/restart saves the memory state of containers into a disk while pause/unpause doesn't.

You can think of pause/unpause as sending SIGSTOP and SIGCONT to UNIX process. You can run a process in foreground and hitting Ctrl+z will stop the process. You can also restore the process with `fg job-id`. Pause/unpause commands do a similar thing to running containers.

Checkpoint/restore does more complicated things. It dumps the memory state of the container into the disk when you checkpoint and reads the dump and map the memory state into current running docker daemon.

We will see the benefit of doing this in the next section.











