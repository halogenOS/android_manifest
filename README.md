<img src="https://raw.githubusercontent.com/halogenOS/android_manifest/XOS-9.0-old/halogenos-logo.png">

___________________________________________________________________________________


### Getting Started with XOS

#### __0. Preliminary Knowledge__

Before beginning this entire process, please ensure you have sufficient storage space. To carry out a single device build an excess of over 30 GB will be required. If building for more than one device, this amount of required storage can exponentially increase. For speedy builds it is highly recommended that you store the sources in a fast storage medium such as Solid State Drives (SSDs), with modern computers it often turns out to be a greater bottleneck than the processor itself when compiling Android.

It should also be noted that in order to build Android from source successfully, you will require GNU Make, git and the Open Java Development Kit as well as a few build and compiler centric packages, this will vary from distribution to distribution so it is recommended that you 'Google' the required packages for compiling Android for your distribution.

Before you continue, make sure you follow the [Setting up a Linux build environment](https://source.android.com/source/initializing.html#setting-up-a-linux-build-environment) guide.
We also recommend you to use CCache for faster builds:

In your environment (e. g. .bashrc):
```
CCACHE_DIR="/path/to/ccache"
USE_CCACHE=1
```

Run following command to set the size limit (minimum 80G recommended):
```
ccache -M 80G
```


#### __1. Getting Started__

To get started with XOS, you should first become familliar with the basics of the utilities named [git](http://rogerdudler.github.io/git-guide/) and [repo](https://source.android.com/source/using-repo.html), if using a development oriented distro or are already an actual developer working with source based Android ROMs or other similar projects you should more likely than not already have these obtained, if not here's an idea of what one should do below.

__Installing repo__

```bash
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

Now add the directory to your PATH variable in your environment (e. g. by appending it to your `~/.bashrc`)

```bash
export PATH="$HOME/bin:$PATH"
```

__Installing git__

Git would more likely than not probably be already installed in your distribution, if it is not then you should try one of the following terminal commands depending on your distribution:

```bash
Ubuntu, Debian (apt): apt-get install git
OpenSUSE: zypper install git
Fedora: yum install git-all
Gentoo: emerge --ask --verbose dev-vcs/git
Arch Linux: pacman -S git
```

The derivatives of these common distributions should also have the git package available, if you believe your distribution does not offer git in the default package repositories then you may consider [compiling and installing git from source](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git#Installing-from-Source).

#### __2. Initiating Repository and Acquiring Sources__

First, create a new empty directory of your choice, and `cd` into it:

```bash
mkdir xossrc
cd xossrc
```

Of course, you can use any directory name you desire. It is recommended to avoid spaces.

Now initialize a repo source tree, to do this please use following command:

```bash
repo init -u https://git.halogenos.org/halogenOS/android_manifest.git -b XOS-9.0
```

Then synchronize the source tree using repo, which will fetch the source of XOS. You should be warned that this is a procedure which downloads huge amounts (about 20-30 GB in total) of 
data, it may take hours to complete. Be prepared with something fun to do as will be waiting for a while or just listen to EDM.

```bash
repo sync -c --no-tags --no-clone-bundle -f build/make external/xos
source build/envsetup.sh
reposync
```

#### __3. Building__

First, in order to build XOS you should source the envsetup.sh script in your terminal/shell, this will set up and import all of the available device configurations for the ROM as well as giving you some fancy "macro" commands for 
your build enviroment. As such, in order to do this, run the command:

```bash
source build/envsetup.sh
```

By running

```bash
breakfast <device>
```

you can fetch the device tree and its dependencies of devices hosted by us.

Now, you should select and configure the build target by using the *lunch* command. Type 'lunch', and a list of the available devices and build targets will be offered, give it a whirl, it won't bite.

Additionally here's a list of build types for your target device that you will likely encounter while running 'lunch'.

| Build type	| Use |
|:----------|:----------|
| user	| The flavour usually for building final releases. We don't use this (at least not yet) because custom ROMs don't play very well with it. |
| userdebug |	Same as "user" but with adb enabled and more debuggable. This is the default. Don't be scared by the `debug` part. |
| eng	| Engineering build, enables shell root access, debuggability, adb, install engineering modules and is fully JIT enabled. Only use this for initial bringup and extensive debugging. |

Before you start building, make sure that you have all necessary device-specific trees.
Official trees, maintained by the team, can be retrieved using:

```bash
breakfast <device>
```

Example:

```bash
breakfast oneplus2
```

At this point you should be able to build the ROM freely, all that's left to do is enter...

```bash
build full XOS_<device>-userdebug
```

Example:

```bash
build full XOS_oneplus2-userdebug
```

This `build` command is a speciality made by the XOS team. It does everything for you, from lunching to breakfasting, and initiating a new build, as well as finding out which amount of threads are optimal for your machine. Hence you must not specify a thread count using `-j` on this command, as that will be done automatically for you.

in your terminal, the master chef that is 'Mr Compiler' (aka Ninja) will do the cooking of the ROM for you, this will take another while and depend on your storage speed and the capability of your CPU.

Once done you should find a cute flashable zip either in your directory or within "out/target/product/", this is your creation, you compiled it, she's yours, your own adorable pet... just make sure to treat her very nicely :).

_Additional build notes : If you're bringing up a new device, our [wiki](https://github.com/halogenOS/android_manifest/wiki) has some important info_

#### __4. Flashing__
If you do not know how to flash an Android ROM then you probably shouldn't have followed this guide in the first place, or have a case of amnesia, but in the case that you do need a briefing then here's a short guide on flashing XOS: https://goo.gl/BB53SU

<br />
