---
layout: post
title: "Using Trusted Builds to generate Docker image automatically"
date: 2013-11-10 08:20
comments: true
permalink: /blog/en/docker/using-trusted-builds-to-generate-docker-image-automatically/
categories:
- en
- docker
---
![](/images/construction.png)
## Trusted Builds
Recently, [index.docker.io](http://index.docker.io/ "index.docker.io") released a new feature called "Trusted Builds".
Trusted Builds allows you to automatically builds a Docker container according to Dockerfile when you push to your Github repository.
In this post, I will share how to use it and one pitfall that I experienced.

### Place a Dockerfile in your repo
Trusted Builds knows how to make a new image by reading Dockerfile in your repository and uses repository as a context for the Dockerfile.
So, you first need to add a Dockerfile to your repository. Let's put a very simple Dockerfile.

```bash
from ubuntu
```

This Dockerfile tells Trusted Builds to create a new image based on ubuntu image. Add the file and push to Github.

### Link Github and register repository
To use Trusted Builds, you need to link your Github account. Login to [index.docker.io](http://index.docker.io/ "index.docker.io") and go to [setting page](https://index.docker.io/builds/github/select/ "").
Once you link Github accountm, you should see the list of your public repositories. Select the repository that you want to build and enter information used by Trusted Builds.

##### Default Branch
Branch name that Trusted Builds uses

##### Repo Name
This is used as a name of new image. Probably, Repo here means Docker's repository, not Github repository. __You should pick up the name carefully because you can't change the name later.__

##### Docker Tag Name
The name of tag that you want put to a image

##### Dockerfile Location
Specify the location of Dockerfile in your repo. As I mentioned earlier, Trusted Builds uses your repo as a build context. Therefore, the top directory of your repo becomes the root of file path.
For example, if you put Dockerfile just under your repo's top directory, you should specify __/__ as Dockerfile location. Note that you only need to specify the directory name, not the path of Dockerfile.

```
# Good
/
/build

# Bad
/Dockerfile
/build/Dockerfile
```

Also note the file name should be Dockerfile. If you put other name, Trusted Builds can't find the file.

You should be ready now! Press Submit button and a new build will start. Once builds finish, you will see a new image in the list of your repository.

## A few common mistakes
Did your build succeed? In my case, I made try & error several times until it succeeds. Let me share them so you can save your time.

##### Is the file name of Dockerfile correct?
The file name of your Dockerfile should be *Dockerfile*.

##### Did you specify correct path to your Dockerfile?
Dockerfile Location should be the path of directory where you have Dockerfile, not the whole path name of Dockerfile. Check the examples above.

##### Is your Dockerfile correct?
Maybe you put wrong instruction in Dockerfile. You can test this by manually building a image by using docker command.

```bash
docker build -t kimh/test_dockerfile github.com/kimh/baleen
```

The above command assumes that you have Dockerfile at the top directory of your repo. If the command fails, make sure your instructions are correct.

##### Do you have submodule in your repository?
This one took the most of debugging time for me. Currently, [Trusted Builds does not support git submodule](https://groups.google.com/forum/#!topic/docker-user/ZothnJ46Pps), so builds fails if you have a submodule in your repository.
For now, you have to remove the submodule to make build succeed.
