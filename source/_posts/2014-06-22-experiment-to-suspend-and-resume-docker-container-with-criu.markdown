---
layout: post
title: "Experiment To Suspend/Resume Docker Container With CRIU"
date: 2014-06-22 01:08
comments: true
categories:
- en
- criu
---

![](/images/criu.jpeg)

### TL;DR: You CANNOT suspend/resume Docker container as of Jun 2014 and this article ends with a bit disappointed result, but you can still find CRIU is cool thing.

With the rapid emergence of Docker, everybody knows many advantages of using LXC over virtual machines like VMWare or Xen.

However, there is one thing that LXC is missing: suspend/resume containers.

This is where [CRIU](http://criu.org/Main_Page) comes in.

CRIU is so called CR (checkpoint/restart) tool. It suspends a running process and save the memory state into files which can be resumed at anytime.

And since LXC container is a process, we should be able to suspend/resume containers. But does it really work?

In this article, we will install CRIU and see whether we can suspend/resume a Docker container.

**Note:**
You may say that LXC already has [C/R feature](http://lxc.sourceforge.net/man/lxc-checkpoint.html). My impression with the tool is not good from the past experience. So, I really didn't try this time.

## Installing CRIU

### Building Kernel
To get the fully functional CRIU, you need to have a kernel with certain options are enabled. We use Vagrant box as LXC host machine, but I couldn't find a box with kernel that meets the requirement of CRIU.

So, we need to rebeild kernel. Don't worry, building kernel is not difficult as it sounds.

We will use [official Ubuntu14.04 cloud image](https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box).

First you need to add the box.

```sh
$ vagrant box add ubuntu14.04 https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box
$ vagrant init ubuntu14.04
```

Before hit ```vagrant up``` command, let's increase CPU cores and memory of the box. Otherwise, it takes a few hours to build kernel. It really depends, but 2 cores with 2048M RAM would be enough.

Open ```Vagrantfile``` and add the following lines.

```
config.vm.provider :virtualbox do |vb|
  vb.customize ["modifyvm", :id, "--memory", "2048"]
  vb.customize ["modifyvm", :id, "--cpus", 2]
end
```

Once you increase cpu and memory, do ```vagrant up && vagrant ssh``` and ssh into the box. Become root user and install necessary packages to rebuild kernel.

```sh
$ apt-get -y update
$ apt-get -y install libncurses-dev build-essential libncurses-dev build-essential fakeroot kernel-package linux-source bc
```

After installing these, you should have `/usr/src/linux-source-<kernel version>` directory. Go into the directory and untar the source files.

```sh
$ cd /usr/src/linux-source-<kernel version>
$ tar xvjf linux-source-<kernel version>.tar.bz2
$ cd ./linux-source-<kernel version>
```

Now you need to prepare a kernel configuration *.config* file which enables certain options for CRIU.

I uploaded the one to save your time into gist. Just run this command. (Make sure to change current directory to kernel source directory where you just untared.)

```sh
$ curl https://gist.githubusercontent.com/kimh/c93f42981d14a33c63c0/raw/a73af0f7f745c2538253ef153a62a8ba1a2d97be/.config -o .config
```

If you want to know which kernel options should be enabled, the list is [here](http://criu.org/Installation#Kernel_configuration).

Once you put ```.config``` file, you are ready to build kernel.

**Once again, make sure you increased cpu and memory in the previous seteps before start building kernel.**

```sh
$ export LC_CTYPE=C
$ make-kpkg clean
$ CONCURRENCY_LEVEL=4 make-kpkg --rootcmd fakeroot --initrd --revision=`date +%Y%m%d` kernel_image kernel_headers
```

Once the kernel build is done, you should have ```linux-headers-<kernel version>_amd64.deb``` and ```linux-image-<kernel version>_amd64.deb``` under ```/usr/src/``` directory.

Now, let's install them.

```sh
$ dpkg -i linux-headers-<kernel version>_amd64.deb
$ dpkg -i linux-image-<kernel version>_amd64.deb
$ reboot
```

Done! Now You are running the kernel that works well with CRIU.

### Compile CRIU from source
Let's install CRIU now. Ubuntu doesn't provide up-to-date debian package of CRIU, so we need to build from source.

```sh
$ apt-get install bsdmainutils build-essential libprotobuf-dev libprotobuf-c0-dev protobuf-c-compiler protobuf-compiler python-protobuf xmlto asciidoc
$ git clone https://github.com/xemul/criu
$ cd criu/
$ make
$ sudo make install
```

Now CRIU is installed. Let's try if it works. CRIU provides a command for this.

```sh
$ criu check --ms
Warn  (tun.c:55): Skipping tun support check
Warn  (cr-check.c:259): Skipping mnt_id support check
Looks good.
```

Did you get ```Looks good.``` message? You may get some warnings, but you can ignore them.
Before doing our experiment with containers, let's checkpoint and restore normal Linux process with CRIU. The example comes from [one of CRIU HOWTO pages](http://criu.org/Simple_loop).

First, we need to create a simple loop script.

```
$ cat > test.sh <<-EOF
#!/bin/sh
while true; do
 date
 sleep 1
done
EOF

$ chmod +x test.sh
$ ./test.sh
```

We can suspend with ```criu dump``` command.

```sh
# Need to be root to run criu
$ sudo -s
$ export PID=`pgrep -f test.sh`
$ mkdir /tmp/test
$ criu dump -t $PID --images-dir /tmp/test --shell-job
```

If the dump succeeds, you should have many files under ```/tmp/test``` directory.

```sh
$ ls /tmp/test
cgroup.img         fanotify-mark.img   fs-4898.img     netlinksk.img     pstree.img         signalfd.img
core-4521.img      fanotify.img        ids-4521.img    ns-files.img      reg-files.img      sk-queues.img
core-4898.img      fdinfo-2.img        ids-4898.img    packetsk.img      remap-fpath.img    stats-dump
creds-4521.img     fdinfo-3.img        inetsk.img      pagemap-4521.img  sigacts-4521.img   tty-info.img
creds-4898.img     fifo-data.img       inotify-wd.img  pagemap-4898.img  sigacts-4898.img   tty.img
eventfd.img        fifo.img            inotify.img     pages-1.img       signal-p-4521.img  tunfile.img
eventpoll-tfd.img  filelocks-4521.img  inventory.img   pages-2.img       signal-p-4898.img  unixsk.img
eventpoll.img      filelocks-4898.img  mm-4521.img     pipes-data.img    signal-s-4521.img
ext-files.img      fs-4521.img         mm-4898.img     pipes.img         signal-s-4898.img
```

Let's resume the process with ```criu restore``` command.

```sh
$ criu restore -t $PID --images-dir /tmp/test  --shell-job
```

If the process is successfully resumed, ```test.sh``` starts printing the output of ```date``` command to your terminal.

## Trying CRIU with containers
So far so good? Now, we will try to suspend and resume Docker containers. Docker is not installed on your vagrant box, so let's install.

```sh
$ apt-get install docker.io jq
$ ln -sf /usr/bin/docker.io /usr/local/bin/docker
$ sed -i '$acomplete -F _docker docker' /etc/bash_completion.d/docker.io
```

And run a Ubuntu container executing a simple command.

```sh
$ docker run -t -i ubuntu /bin/bash
```

To suspend the container, we need to know the pid of the container.

```sh
$ ID=`docker ps -l -q`
$ PID=`docker inspect $ID | jq '.[0].State.Pid'`
```

Ok, our long journey is almost done. Let's suspend the container!!

```sh
$ criu dump -t $PID --images-dir /tmp/docker
Error (mount.c:449): 102:./dev/console doesn't have a proper root mount
Error (cr-dump.c:1882): Dumping FAILED.
```


CRIU said dumping failed. After googling the error message, I found this discussion.

[[lxc-devel] [CRIU] LXC live migrate](https://lists.linuxcontainers.org/pipermail/lxc-devel/2013-November/006326.html)
> That's container's console which is a bind mounted tty from
> the host. And since this is an external connection, CRIU doesn't dump one.

What?! But, [this page](http://criu.org/LXC) says CRIU supports LXC checkpoint/restart. Docker uses LXC under the hood, so how come it doesn't work?

In the same thread of the discussion, I also found this.

[[lxc-devel] [CRIU] LXC live migrate](https://lists.linuxcontainers.org/pipermail/lxc-devel/2013-November/006326.html)
> AFAIK cgroups are used _inside_ containers only with recent guest templates.
> In OpenVZ we use more old ones (and more stable) so haven't meet this yet.
> And yes, cgroups are in plans for the nearest future :)

So, it seems CRIU does not support cgroup at the time of writing this (Jun 2014). However Docker uses LXC template that uses cgroups. Therefore, CRIU doesn't work with Docker containers.

Sigh...

## Conclusion
With this experiment, I found that we cannot checkpoint/resume Docker container with CRIU v1.3 because it does not support cgroups.

The result turned out to be a bit disappointed. However, I'm sure now you know that CRIU is extremely exciting project.

In contrast to its potential impact to LXC ecosystem, I believe the project is not receiving enough attention, so give a star and start watching their [Github repo](https://github.com/xemul/criu) now!! I will definitely cover more things about CRIU on this blog, too.
