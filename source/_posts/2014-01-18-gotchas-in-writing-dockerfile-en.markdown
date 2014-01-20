---
layout: post
title: "Gotchas in Writing Dockerfile"
date: 2014-01-18 23:22
comments: true
categories:
---
![](/images/instruction.jpg)

## Contents of This Article
#### [Why do we need to use Dockerfile?](#why_do_we_need_to_use_dockerfile)
#### [ADD and understanding context in Dockerfile](#add_and_understanding_context_in_dockerfile)
#### [Build caching: what invalids cache and not?](#build_caching_what_invalids_cache_and_not)
* [Cache invalidation at one instruction invalids cache of all subsequent instructions](#cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions)
* [Cache is invalid even when adding commands that don't do anything](#cache_is_invalid_even_when_adding_commands_that_dont_do_anything)
* [Cache is invalid when you add spaces between command and arguments inside instruction](#cache_is_invalid_when_you_add_spaces_between_command_and_arguments_inside_instruction)
* [Cache is used when you add spaces around commands](#cache_is_used_when_you_add_spaces_around_commands_inside_instruction)
* [Cache is used for non-idempotent instructions](#cache_is_used_for_non_idempotent_instructions)
* [Instructions after ADD never cached (Only versions before 0.7.3)](#instructions_after_add_never_cached_only_versions_before_0.7.3)

#### [Hack to run container in the background](#hack_to_run_container_in_the_background)


<a id="why_do_we_need_to_use_dockerfile"></a>
## Why do we need to use Dockerfile?
Dockerfile is not yet-another shell. Dockerfile has its special mission: **automation of Docker image creation.**

Once, you write build instructions into Dockerfile, you can build the same image just with `docker build` command.

Dockerfile is also useful to tell the knowledge of what a job the container does to somebody else. Your teammates can tell what the container is supposed to do just by reading Dockerfile. They don't need to know login to the container and figure out what the container is doing by using ps command.

For these reasons, you **must** use Dockerfile when you build images. However, writing Dockerfile is sometimes painful. In this post, I will write a few tips and gochas in writing Dockerfile so that you love the tool.

<a id="add_and_context_in_dockerfile"></a>
## ADD and understanding context in Dockerfile
***ADD*** is the instruction to add local files to Docker image.  The basic usage is very simple. Suppose you want to add a local file called *myfile.txt* to /myfile.txt of image.


[foo]: This is your directory entries.
```bash
$ ls
Dockerfile  mydir  myfile.txt
```

Then your Dockerfile looks like this.

```
ADD myfile.txt /
```

Very simple. However, if you want to add */home/vagrant/myfile.txt*, you can't do this.

```bash
# Your have this in your Dockerfile
# ADD /home/vagrant/myfile.txt /

$ docker build -t blog .
Uploading context 10240 bytes
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : ADD /home/vagrant/myfile.txt /
Error build: /home/vagrant/myfile.txt: no such file or directory
```

You got `no such file or directory` error even if you have the file. Why? This is because */home/vagrant/myfile.txt* is not added to the **context** of Dockerfile. Context in Dockerfile means files and directories available to the Dockerfile instructions. This, only files and directories can be added during build.
Files and sub directories under the current directory are added to the context. You can see this when you run build command.

```
$ docker build -t blog .
Uploading context 10240 bytes
```

What's happening here is Docker client makes tarball of entries under the current directory and send it to Docker daemon. It says *Uploading* because your Docker daemon may be running on remote machine.

There is a pitfall, though. Since automatically entries under current directories are added to the context, it tries to upload huge files and take longer time for build even if you don't add the file.

```bash
$ ls
Dockerfile  very_huge_file

$ docker build -t blog .
Uploading context xxxxxx bytes
..... # Takes very long time
```

So the best practice is only placing files and directories that you need to add to image under current directory.

<a id="build_caching_what_invalids_cache_and_not"></a>
## Build caching: what invalids cache and not?
Docker creates a commit for each line of instruction in Dockerfile. As long as you don't change the instruction, Docker thinks it doesn't need to change the image, so use cached image which is used by the next instruction as a parent image.
This is the reason why `docker build` takes long time in the first time, but immediately finishes in the second time.

```bash
$ time docker build -t blog .
Uploading context 10.24 kB
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : RUN apt-get update
 ---> Running in 15705b182387
Ign http://archive.ubuntu.com precise InRelease
Hit http://archive.ubuntu.com precise Release.gpg
Hit http://archive.ubuntu.com precise Release
Hit http://archive.ubuntu.com precise/main amd64 Packages
Get:1 http://archive.ubuntu.com precise/main i386 Packages [1641 kB]
Get:2 http://archive.ubuntu.com precise/main TranslationIndex [3706 B]
Get:3 http://archive.ubuntu.com precise/main Translation-en [893 kB]
Fetched 2537 kB in 7s (351 kB/s)

 ---> a8e9f7328cc4
Successfully built a8e9f7328cc4

real	0m8.589s
user	0m0.008s
sys	0m0.012s

$ time docker build -t blog .
Uploading context 10.24 kB
Step 1 : FROM ubuntu
 ---> 8dbd9e392a96
Step 2 : RUN apt-get update
 ---> Using cache
 ---> a8e9f7328cc4
Successfully built a8e9f7328cc4

real	0m0.067s
user	0m0.012s
sys	0m0.000s
```
However, when cache is used and what invalids cache are sometimes not very clear. Here is a few cases that I found worth to note.

<a id="cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions"></a>
#### Cache invalidation at one instruction invalids cache of all subsequent instructions
This is the basic rule of caching. If you cause cache invalidation at one instruction, subsequent instructions doesn't use cache.

```bash
# Before
From ubuntu
Run apt-get install ruby
Run echo done!

# Before
From ubuntu
Run apt-get update
Run apt-get install ruby
Run echo done!
```

Since you add *Run apt-get update* instruction, **all** instructions after that have to be done from the scratch even if they are not changed.
This is inevitable because Dockerfile uses the image built by the previous instruction as a parent image to execute next instruction. So, if you insert an instruction that creates a new parent image, all subsequent instructions cannot use cache because now parent image differs.

<a id="cache_invalidation_at_one_instruction_invalids_cache_of_all_subsequent_instructions"></a>
#### Cache is invalid even when adding commands that don't do anything
*This invalidates caching.* For example,

```bash
# Before
Run apt-get update

# After
Run apt-get update && true
```

Even if `true` command doesn't change anything of the image, Docker invalids the cache.


<a id="cache_is_invalid_when_you_add_spaces_between_command_and_arguments_inside_instruction"></a>
#### Cache is invalid when you add spaces between command and arguments inside instruction
*However, this invalids cache*
```bash
# Before
Run apt-get update

# After
Run apt-get               update
```

<a id="cache_is_used_when_you_add_spaces_around_commands_inside_instruction"></a>
#### Cache is used when you add spaces around commands inside instruction
*Cache is valid even if you add space around commands*

```bash
# Before
Run apt-get update

# After
Run                apt-get update
```

<a id="cache_is_used_for_non_idempotent_instructions"></a>
#### Cache is used for non-idempotent instructions
This is kind of pitfall of build caching. What I mean by non-idempotent instructions is the execution of commands that may return different result each time.
For example, `apt-get update` is not idempotent because the content of updates changes as time goes by.

```bash
From ubuntu
Run apt-get update
```

You made this Dockerfile and create image. 3 months later, Ubuntu made some security updates to their repository, so you rebuild the image by using the same Dockerfile hoping your new image includes the security updates.
However, this doesn't pick up the updates. Since no instructions or files are changed, Docker uses cache and skips doing `apt-get update`.

If you don't want to use cache, just pass `--no-cache` option to build.

```bash
$ docker build . --no-cache
```


<a id="instructions_after_add_never_cached_only_versions_before_0.7.3"></a>
#### Instructions after ADD never cached (Only versions before 0.7.3)
If you use Docker before v7.3, watch out!

```bash
From ubuntu
Add myfile /
Run apt-get update
Run apt-get install openssh-server
```

If you have Dockerfile like this, ***Run apt-get update*** and ***Run apt-get install openssh-server***  will never be cached.

The behavior is changed from v7.3 in a good way. It caches even if you have ADD instruction, but invalids cache if file content is changed.

```bash
$ echo "Jeff Beck" > rock.you

From ubuntu
Add rock.you /
Run add rock.you

$ echo "Eric Clapton" > rock.you

From ubuntu
Add rock.you /
Run add rock.you
```

Since you change *rock.you* file, instructions after Add doesn't use cache.

<a id="hack_to_run_container_in_the_background"></a>
## Hack to run container in the background
If you want to simplify the way to run containers, you should run your container on background with `docker run -d image your-command`.
Instead of running with `docker run -i -t image your-command`, using `-d` is recommended because you can run your container with just one command and you don't need to detach terminal of container by hitting `Ctrl + P + Q`.

However, there is a problem with `-d` option. Your container immediately stops unless the commands are not running on foreground.

Let me explain this by using case where you want to run apache service on a container. The intuitive way of doing this is

```bash
$ docker run -d apache-server apachectl start
```

However, the container stops immediately after it is started. This is because `apachectl` exits once it detaches apache daemon.

Docker doesn't like this. Docker requires your command to keep running in the foreground.
Otherwise, it thinks that your applications stops and shutdown the container.

You can solve this by directly running apache executable with foreground option.

```bash
$ docker run -e APACHE_RUN_USER=www-data \
             -e APACHE_RUN_GROUP=www-data \
             -e APACHE_PID_FILE=/var/run/apache2.pid \
             -e APACHE_RUN_DIR=/var/run/apache2\
             -e APACHE_LOCK_DIR=/var/lock/apache2\
             -e APACHE_LOG_DIR=/var/log/apache2\
             -d apache-server /usr/sbin/apache2 -D NO_DETACH -D FOREGROUND
```

Here we are manually doing what `apachectl` does for us and run apache executable. With this approach, apache keeps running on foreground.

The problem is that some application does not run in the foreground. Also, we need to do extra works such as exporting environment variables by ourselves. How can we make it easier?

In this situation, you can add `tail -f /dev/null` to your command and they will run in the foreground forever. We can use this technique in the apache case.

```bash
$ docker run -d apache-server apachectl start && tail -f /dev/null
```

Much better, right? Since `tail -f /dev/null` doesn't do any harm, you can use this hack to any applications.






