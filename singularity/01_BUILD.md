## Building and Running Containers

In the second hour we will build the preceding container from scratch.

Simply typing `singularity` will give you an summary of all the commands you can use. Typing `singularity help <command>` will give you more detailed information about running an individual command.

### Building a basic container

To build a singularity container, you must use the `build` command.  The `build` command installs an OS, sets up your container's environment and installs the apps you need.  To use the `build` command, we need a **recipe file** (also called a definition file). A Singularity recipe file is a set of instructions telling Singularity what software to install in the container.

The Singularity source code contains several example definition files in the `/examples` subdirectory.  Let's copy the ubuntu example to our home directory and inspect it.

**Note:** You need to build containers on a file system where the sudo command can write files as root. This may not work in an HPC cluster setting if your home directory resides on a shared file server. If that's the case you may have to to `cd` to a local hard disk such as `/tmp`.

```
$ mkdir ../lolcow

$ cp examples/ubuntu/Singularity ../lolcow/

$ cd ../lolcow

$ vim Singularity
```

It should look something like this:

```
BootStrap: library
From: ubuntu:18.04

%runscript
    echo "This is what happens when you run the container..."

%post
    echo "Hello from inside the container"
    apt-get -y install vim

```

See the [Singularity docs](https://www.sylabs.io/guides/3.0/user-guide/definition_files.html) for an explanation of each of these sections.

Now let's use this recipe file as a starting point to build our `lolcow.img` container. Note that the build command requires `sudo` privileges, when used in combination with a recipe file.

```
$ sudo singularity build --sandbox lolcow Singularity
```

The `--sandbox` option in the command above tells Singularity that we want to build a special type of image (File system) for development purposes.

Singularity can build containers in several different file formats. The default is to build a [SIF](https://github.com/sylabs/sif) image. SIF is an open source implementation of the Singularity Container Image Format that makes it easy to create complete and encapsulated container enviroments stored in a single file.

But if you want to shell into a container and tinker with it (like we will do here), you should build a sandbox (which is really just a file system under a foler directory).  This is great when you are still developing your container and don't yet know what should be included in the recipe file.

When your build finishes, you will have a basic Ubuntu container saved in a local directory called `lolcow`.

### Using `shell` to explore and modify containers

Now let's enter our new container and look around.

```
$ singularity shell lolcow
```

Depending on the environment on your host system you may see your prompt change. Let's look at what OS is running inside the container.

```
Singularity lolcow:~> cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

No matter what OS is running on your host, your container is running Ubuntu 14.04!

Let's try a few more commands:

```
Singularity lolcow:~> whoami
eduardo

Singularity lolcow:~> hostname
eduardo-laptop
```

This is one of the core features of Singularity that makes it so attractive from a security standpoint.  The user remains the same inside and outside of the container.

Let's try installing some software. For building lolcow we first need the programs `fortune`, `cowsay`, and `lolcat` to produce the container that we saw in the first demo.

```
Singularity lolcow:~> sudo apt-get update && sudo apt-get -y install fortune cowsay lolcat
bash: sudo: command not found
```

Whoops!

Singularity complains that it can't find the `sudo` command.  But even if you try to install `sudo` or change to root using `su`, you will find it impossible to elevate your privileges within the container.

Once again, this is an important concept in Singularity.  If you enter a container without root privileges, you are unable to obtain root privileges within the container.  This insurance against privilege escalation is the reason that you will find Singularity installed in so many HPC environments.

Let's exit the container and re-enter as root.

```
Singularity lolcow:~> exit

$ sudo singularity shell --writable lolcow
```

Now we are the root user inside the container. Note also the addition of the `--writable` option.  This option allows us to modify the container.  The changes will actually be saved into the container and will persist across uses.

Let's try installing some software again.

```
Singularity lolcow:~> apt-get update && apt-get -y install fortune cowsay lolcat
```

Now you should see the programs successfully installed.  Let's try running the demo in this new container.

```
Singularity lolcow:~> fortune | cowsay | lolcat
bash: lolcat: command not found
bash: cowsay: command not found
bash: fortune: command not found
```

Drat! It looks like the programs were not added to our `$PATH`.  Let's add them and try again.

```
Singularity lolcow:~> export PATH=/usr/games:$PATH

Singularity lolcow:~> fortune | cowsay | lolcat
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
        LANGUAGE = (unset),
        LC_ALL = (unset),
        LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
 ________________________________________
/ Keep emotionally active. Cater to your \
\ favorite neurosis.                     /
 ----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We're making progress, but we are now receiving a warning from perl.  However, before we tackle that, let's think some more about the `$PATH` variable.

We changed our path in this session, but those changes will disappear as soon as we exit the container just like they will when you exit any other shell.  To make the changes permanent we should add them to the definition file and re-bootstrap the container.  We'll do that in a minute.

Now back to our perl warning.  Perl is complaining that the locale is not set properly.  Basically, perl wants to know where you are and what sort of language encoding it should use.  Should you encounter this warning you can  probably fix it with the `locale-gen` command or by setting `LC_ALL=C`.  Here we'll just set the environment variable.

```
Singularity lolcow:~> export LC_ALL=C

Singularity lolcow:~> fortune | cowsay | lolcat
 _________________________________________
/ FORTUNE PROVIDES QUESTIONS FOR THE      \
| GREAT ANSWERS: #19 A: To be or not to   |
\ be. Q: What is the square root of 4b^2? /
 -----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Great!  Things are working properly now.

Although it is fine to shell into your Singularity container and make changes while you are debugging, you ultimately want all of these changes to be reflected in your recipe file.  Otherwise if you need to reproduce it from scratch you will forget all of the changes you made.

Let's update our definition file with the changes we made to this container.

```
Singularity lolcow:~> exit

$ nano Singularity
```

Here is what our updated definition file should look like.

```
BootStrap: library
From: ubuntu:18.04

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%runscript
    fortune | cowsay | lolcat
```

Let's rebuild the container with the new definition file.

```
$ sudo singularity build lolcow.sif Singularity
```

Note that we changed the name of the container.  By omitting the `--sandbox` option, we are building our container in the standard Singularity squashfs file format.  We are denoting the file format with the (optional) `.sif` extension (Singularity image format).  A squashfs file is compressed and immutable making it a good choice for a production environment.

Singularity stores a lot of [useful metadata](https://www.sylabs.io/guides/3.0/user-guide/environment_and_metadata.html).  For instance, if you want to see the recipe file that was used to create the container you can use the `inspect` command like so:

```
$ singularity inspect --deffile lolcow.sif
BootStrap: library
From: ubuntu:18.04

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%runscript
    fortune | cowsay | lolcat
```

### Blurring the line between the container and the host system.

Singularity does not try to isolate your container completely from the host system.  This allows you to do some interesting things.

Using the exec command, we can run commands within the container from the host system.

```
$ singularity exec lolcow.sif cowsay 'How did you get out of the container?'
 _______________________________________
< How did you get out of the container? >
 ---------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

In this example, singularity entered the container, ran the `cowsay` command, displayed the standard output on our host system terminal, and then exited.

You can also use pipes and redirection to blur the lines between the container and the host system.

```
$ singularity exec lolcow.sif cowsay moo > cowsaid

$ cat cowsaid
 _____
< moo >
 -----
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We created a file called `cowsaid` in the current working directory with the output of a command that was executed within the container.

We can also pipe things _into_ the container.

```
$ cat cowsaid | singularity exec lolcow.sif cowsay -n
 ______________________________
/  _____                       \
| < moo >                      |
|  -----                       |
|         \   ^__^             |
|          \  (oo)\_______     |
|             (__)\       )\/\ |
|                 ||----w |    |
\                 ||     ||    /
 ------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We've created a meta-cow (a cow that talks about cows). :stuck_out_tongue_winking_eye:
