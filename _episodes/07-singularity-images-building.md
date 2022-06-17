---
title: "Building Singularity images"
teaching: 30
exercises: 30
questions:
- "How do I create my own Singularity images?"
objectives:
- "Understand the different Singularity container file formats."
- "Understand how to build and share your own Singularity containers."
keypoints:
- "Singularity definition files are used to define the build process and configuration for an image."
- "Singularity's Docker container provides a way to build images on a platform where Singularity is not installed but Docker is available."
- "Existing images from remote registries such as Docker Hub and Singularity Hub can be used as a base for creating new Singularity images."
---

## Building Singularity images

### Introduction

As a platform that is widely used in the scientific/research software and HPC communities, Singularity provides great support for reproducibility. If you build a Singularity image for some scientific software, it's likely that you and/or others will want to be able to reproduce exactly the same environment again. Maybe you want to verify the results of the code or provide a means that others can use to verify the results to support a paper or report. Maybe you're making a tool available to others and want to ensure that they have exactly the right version/configuration of the code.

Similarly to Docker and many other modern software tools, Singularity follows the "Configuration as code" approach and a container configuration can be stored in a file which can then be committed to your version control system alongside other code. Assuming it is suitably configured, this file can then be used by you or other individuals (or by automated build tools) to reproduce a container with the same configuration at some point in the future.

### Different approaches to building images

There are various approaches to building Singularity images. We highlight two different approaches here and focus on one of them:

 - _Building within a sandbox:_ You can build a container interactively within a sandbox environment. This means you get a shell within the container environment and install and configure packages and code as you wish before exiting the sandbox and converting it into a container image.
- _Building from a [Singularity Definition File](https://apptainer.org/user-docs/3.7/build_a_container.html#creating-writable-sandbox-directories)_: This is Singularity's equivalent to building a Docker container from a `Dockerfile` and we'll discuss this approach in this section.

You can take a look at Singularity's "[Build a Container](https://apptainer.org/user-docs/3.7/build_a_container.html)" documentation for more details on different approaches to building containers.

> ## Why look at Singularity Definition Files?
> Why do you think we might be looking at the _definition file approach_ here rather than the _sandbox approach_?
>
> > ## Discussion
> > The sandbox approach is great for prototyping and testing out an image configuration but it doesn't provide the best support for our ultimate goal of _reproducibility_. If you spend time sitting at your terminal in front of a shell typing different commands to add configuration, maybe you realise you made a mistake so you undo one piece of configuration and change it. This goes on until you have your completed, working configuration but there's no explicit record of exactly what you did to create that configuration. 
> > 
> > Say your container image file gets deleted by accident, or someone else wants to create an equivalent image to test something. How will they do this and know for sure that they have the same configuration that you had?
> > With a definition file, the configuration steps are explicitly defined and can be easily stored (and re-run).
> > 
> > Definition files are small text files while container files may be very large, multi-gigabyte files that are difficult and time consuming to move around. This makes definition files ideal for storing in a version control system along with their revisions.
> {: .solution}
{: .challenge}

> ## Using Docker to build Singularity images
> Another common and convenient approach to building images to use with Singularity is to use Docker
> on your local system (e.g. laptop) and them convert them to Singularity images on the remote system
> using Singularity itself. This has a number of potential advantages:
>
> - We can reuse our skills from building Docker containers and do not need to learn a different syntax
> - We do not need to install Singularity ourselves
> - Getting images on to the remote system can be easier as they can be pulled from Dockerhub
>
> If you do not want to share images publicly on Dockerhub to transfer them, you can export images from
> Docker to a file, copy the file to the remote system and then convert them to Singularity images 
> directly.
{: .callout}

### Creating a Singularity Definition File

A Singularity Definition File is a text file that contains a series of statements that are used to create a container image. In line with the _configuration as code_ approach mentioned above, the definition file can be stored in your code repository alongside your application code and used to create a reproducible image. This means that for a given commit in your repository, the version of the definition file present at that commit can be used to reproduce a container with a known state. It was pointed out earlier in the course, when covering Docker, that this property also applies for Dockerfiles.

We'll now look at a very simple example of a definition file:

~~~
Bootstrap: docker
From: ubuntu:20.04

%post
    apt-get -y update && apt-get install -y python3

%runscript
    python3 -c 'import sys; print("Hello World! Hello from Python %s.%s.%s in our custom Singularity image!" % sys.version_info[:3])'
~~~
{: .language-bash}

A definition file has a number of optional sections, specified using the `%` prefix, that are used to define or undertake different configuration during different stages of the image build process. You can find full details in Singularity's [Definition Files documentation](https://apptainer.org/user-docs/3.7/definition_files.html). In our very simple example here, we only use the `%post` and `%runscript` sections.

Let's step through this definition file and look at the lines in more detail:

~~~
Bootstrap: docker
From: ubuntu:20.04
~~~
{: .language-bash}

These first two lines define where to _bootstrap_ our image from. Why can't we just put some application binaries into a blank image? Any applications or tools that we want to run will need to interact with standard system libraries and potentially a wide range of other libraries and tools. These need to be available within the image and we therefore need some sort of operating system as the basis for our image. The most straightforward way to achieve this is to start from an existing base image containing an operating system. In this case, we're going to start from a minimal Ubuntu 20.04 Linux Docker image. Note that we're using a Docker image as the basis for creating a Singularity image. This demonstrates the flexibility in being able to start from different types of images when creating a new Singularity image.

The `Bootstrap: docker` line is similar to prefixing an image path with `docker://` when using, for example, the `singularity pull` command. A range of [different bootstrap options](https://apptainer.org/user-docs/3.7/definition_files.html#preferred-bootstrap-agents) are supported. `From: ubuntu:20.04` says that we want to use the `ubuntu` image with the tag `20.04` from Docker Hub.

Next we have the `%post` section of the definition file:

~~~
%post
    apt-get -y update && apt-get install -y python3
~~~
{: .language-bash}

In this section of the file we can do tasks such as package installation, pulling data files from remote locations and undertaking local configuration within the image. The commands that appear in this section are standard shell commands and they are run _within_ the context of our new container image. So, in the case of this example, these commands are being run within the context of a minimal Ubuntu 20.04 image that initially has only a very small set of core packages installed.

Here we use Ubuntu's package manager to update our package indexes and then install the `python3` package along with any required dependencies. The `-y` switches are used to accept, by default, interactive prompts that might appear asking you to confirm package updates or installation. This is required because our definition file should be able to run in an unattended, non-interactive environment.

Finally we have the `%runscript` section:

~~~
%runscript
    python3 -c 'import sys; print("Hello World! Hello from Python %s.%s.%s in our custom Singularity image!" % sys.version_info[:3])'
~~~
{: .language-bash}

This section is used to define a script that should be run when a container is started based on this image using the `singularity run` command. In this simple example we use `python3` to print out some text, including the currently running version of Python, to the console.

We can now save the contents of the simple defintion file shown above to a file and build an image based on it. In the case of this example, the definition file has been named `my_test_image.def`. (Note that the instructions here assume you've bound the image output directory you created to the `/home/singularity` directory in your Docker Singularity container, as explained in the "[_Getting started with the Docker Singularity image_]({{ site.baseurl }}{% link _episodes/06-singularity-images-prep.md %}#getting-started-with-the-docker-singularity-image)" section above.):

~~~
$ singularity build /home/singularity/my_test_image.sif /home/singularity/my_test_image.def
~~~
{: .language-bash}

Recall from the details at the start of this section that if you are running your command from the host system command line, running an instance of a Docker container for each run of the command, your command will look something like this:

~~~
$ docker run --privileged --rm --mount type=bind,source=${PWD},target=/home/singularity quay.io/singularity/singularity:v3.7.3-slim build /home/singularity/my_test_image.sif /home/singularity/my_test_image.def
~~~
{: .language-bash}

The above command requests the building of an image based on the `my_test_image.def` definition file with the resulting image saved to the `my_test_image.sif` file. Note that you will need to prefix the command with `sudo` if you're running a locally installed version of Singularity and not running via Docker because it is necessary to have administrative privileges to build the image. You should see output similar to the following:

~~~
INFO:    Starting build...
Getting image source signatures
Copying blob sha256:ea362f368469f909a95f9a6e54ebe0121ce0a8e3c30583dd9c5fb35b14544dec
Copying config sha256:15162bcf92407ebb6035c0b78021103ea75860975b1c871f4437439393d9907e
Writing manifest to image destination
Storing signatures
2022/01/18 14:07:55  info unpack layer: sha256:ea362f368469f909a95f9a6e54ebe0121ce0a8e3c30583dd9c5fb35b14544dec
INFO:    Running post scriptlet
+ apt-get -y update
Get:1 http://archive.ubuntu.com/ubuntu focal InRelease [265 kB]
...
  [Package update output truncated]
...
Fetched 20.5 MB in 3s (8128 kB/s)
Reading package lists... 
+ apt-get install -y python3
Reading package lists... 
...
  [Package install output truncated]
...Processing triggers for libc-bin (2.31-0ubuntu9.2) ...
INFO:    Adding runscript
INFO:    Creating SIF file...
INFO:    Build complete: /home/singularity/my_test_image.sif
$ 
~~~
{: .output}

You should now have a `my_test_image.sif` file in the current directory. Note that in the above output, where it says `INFO:  Starting build...` you may see a series of `skipped: already exists` messages for the `Copying blob` lines if the Docker image slices for the Ubuntu 20.04 image have previously been downloaded and are cached on the system where you're running this example. On your system, if the image is not already cached, you will see the slices being downloaded from Docker Hub when these lines of output appear.

> ## Permissions of the created image file
>
> You may find that the created Singularity image file on your host filesystem is owned by the `root` user and not your user. In this case, you won't be able to change the ownership/permissions of the file directly if you don't have root access.
>
> However, the image file will be readable by you and you should be able to take a copy of the file under a new name which you will then own. You will then be able to modify the permissions of this copy of the image and delete the original root-owned file since the default permissions should allow this.
> 
{: .callout}

It is recommended that you move the created `.sif` file to a platform with an installation of Singularity, rather than attempting to run the image using the Docker container. However, for testing purposes, and to save you having to copy the image to a remote platform, if you do wish to try using Singularity within the Singularity Docker container to run your test image, see the notes below on "_Using `singularity run` from within the Docker container_".

> ## Cluster platform configuration for running Singularity containers
>
> For testing purposes, we can run our image that we've created on ARCHER2 by copying the image to the platform and running it interactively at the terminal.
>
> However, for a real job, this is not an option and we would be required to submit our job to the job scheduler on ARCHER2 to have it run on the system's compute nodes.
>
> Doing this requires a number of configuration parameters to be specified. The parameters that need to be set and the values that they need to be set to are specific to the cluster you're running on. In the next section, where we look at running parallel jobs, we'll provide platform-specfic parameters for ARCHER2. However, if you're running on your own institutional cluster or another HPC platform that you have access to, it is likely that you'll need to liaise with the system administrators or look at documentation to identify the parameters that need to be passed to Singularity.
> 
{: .callout}

If you have access to a remote platform with Singularity installed on it, you should now move your created `.sif` image file to this platform. You could, for example, do this using the command line secure copy command `scp`.

> ## Using `scp` (secure copy) to copy files between systems
>
> `scp` is a widely used tool that uses the SSH protocol to securely copy files between systems. As such, the syntax is similar to that of SSH.
>
> For example, if you want to copy the `my_image.sif` file from the current directory on your local system to your home directory (e.g. `/home/myuser/`) on a remote system (e.g. _hpc.myinstitution.ac.uk_) where an SSH private key is required for login, you would use a command similar to the following:
>
> ```
> scp -i /path/to/keyfile/id_mykey ./my_image.sif myuser@hpc.myinstitution.ac.uk:/home/myuser/
> ```
> Note that if you leave off the `/home/myuser` and just end the command with the `:`, the file will, by default, be copied to your home directory.
>
{: .callout}

We can now attempt to run a container from the image that we built:

~~~
$ singularity run my_test_image.sif
~~~
{: .language-bash}

If everything worked successfully, you should see the message printed by Python:

~~~
Hello World! Hello from Python 3.8.10 in our custom Singularity image!
~~~
{: .output}

> ## Using `singularity run` from within the Docker container
>
> It is strongly recommended that you don't use the Docker container for running Singularity images, only for creating them, since the Singularity command runs within the container as the root user.
> 
> However, for the purposes of this simple example, and potentially for testing/debugging purposes it is useful to know how to run a Singularity container within the Docker Singularity container. If you're using the same version (or a newer version) of the Docker Singularity image to the one being used in this material (_3.7.3-slim_), you should be able to run the container as we've done in previous examples in this section, telling it to use the `singularity run` command and passing the name of the image file. _Note that the name of the image file must be passed giving the path it will appear at **inside** the container._
> 
> ~~~
> docker run --privileged --rm --mount type=bind,source=${PWD},target=/home/singularity quay.io/singularity/singularity:v3.7.3-slim run /home/singularity/my_test_image.sif
> ~~~
> {: .output}
> 
> ### Using older versions of the Singularity Docker container
>
> _**NOTE:** The remainder of the content in this callout only applies if you're running an older version of the Docker Singularity container and experience issues..._
> 
> In the event that you're using an older version of the Singularity Docker container, you may encounter an error similar to the following:
>
> ~~~
> WARNING: skipping mount of /etc/localtime: no such file or directory
> FATAL:   container creation failed: mount /etc/localtime->/etc/localtime error: while mounting /etc/localtime: mount source /etc/localtime doesn't exist
> ~~~
> {: .output}
> 
> This occurs because the `/etc/localtime` file that provides timezone configuration is not present within the Singularity Docker container. If you want to use the Docker container to test that your newly created image runs, you can add the `--contain` switch to `singularity run`, or you can open a shell in the Docker container and add a timezone configuration as described in the [Alpine Linux documentation](https://wiki.alpinelinux.org/wiki/Setting_the_timezone):
>
> ~~~
> $ apk add tzdata
> $ cp /usr/share/zoneinfo/Europe/London /etc/localtime
> ~~~
> {: .language-bash}
> 
> The `singularity run` command should now work successfully without needing to use `--contain`. Bear in mind that once you exit the Docker Singularity container shell and shutdown the container, this configuration will not persist.
{: .callout}


### More advanced definition files

Here we've looked at a very simple example of how to create an image. At this stage, you might want to have a go at creating your own definition file for some code of your own or an application that you work with regularly. There are several definition file sections that were _not_ used in the above example, these are:

 - `%setup`
 - `%files`
 - `%environment`
 - `%startscript`
 - `%test`
 - `%labels`
 - `%help`

The [`Sections` part of the definition file documentation](https://apptainer.org/user-docs/3.7/definition_files.html#sections) details all the sections and provides an example definition file that makes use of all the sections.

### Additional Singularity features

Singularity has a wide range of features. You can find full details in the [Singularity User Guide](https://apptainer.org/user-docs/3.7/index.html) and we highlight a couple of key features here that may be of use/interest:

**Signing containers:** If you do want to share container image (`.sif`) files directly with colleagues or collaborators, how can the people you send an image to be sure that they have received the file without it being tampered with or suffering from corruption during transfer/storage? And how can you be sure that the same goes for any container image file you receive from others? Singularity supports signing containers. This allows a digital signature to be linked to an image file. This signature can be used to verify that an image file has been signed by the holder of a specific key and that the file is unchanged from when it was signed. You can find full details of how to use this functionality in the Singularity documentation on [Signing and Verifying Containers](https://apptainer.org/user-docs/3.7/signNverify.html).

