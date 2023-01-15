![](https://git.halogenos.org/halogenOS/android_manifest/-/raw/XOS-12.1/halogenos-logo.png)

### Getting Started with XOS

#### __0. Preliminary Knowledge__

Before beginning this entire process, please ensure you have sufficient storage space. To carry out a single device build an excess of over 150 GB will be required. If building for more than one device, this amount of required storage increases. For speedy builds it is highly recommended that you store the sources in a fast storage medium such as Solid State Drives (SSDs), with modern computers it often turns out to be a greater bottleneck than the processor itself when compiling Android. If you decide to build on a SSD with lower capacity (less than 500 GB), make sure that you use MLC or SLC SSDs only as the lifetime of TLC and QLC SSDs will be impacted as you build the OS.

It should also be noted that in order to build Android from source successfully, you will require a few build and compiler centric packages, this will vary from distribution to distribution. If you read on, you'll find more information as to what is necessary. Note that the tools needed depend on quite a few factors, some being out of our control, so please make sure you look the necessary packages up in case any is missing before contacting us about issues with the build.

Before you continue, make sure you follow the [Setting up a Linux build environment](https://source.android.com/source/initializing.html#setting-up-a-linux-build-environment) guide as it contains a lot of useful and important information regarding building AOSP.

#### Arch builders, ahoy!

We recommend building on Arch as that is what we use for daily building and development.
You can install all necessary packages using following commands:

```
sudo pacman -Syu --needed --noconfirm \
      base-devel bc ccache curl git gnupg \
      inetutils iputils net-tools libxslt ncurses \
      repo rsync squashfs-tools unzip \
      zip zlib ffmpeg lzop ninja pngcrush openssl \
      gradle maven libxcrypt-compat xmlstarlet \
      openssh gperf schedtool perl-switch ttf-dejavu \
      imagemagick jq
```

#### __1. Getting Started__

To get started with XOS, you should first become familiar with the basics of the utilities named [Git](http://rogerdudler.github.io/git-guide/) and [repo](https://source.android.com/source/using-repo.html), if using a development oriented distro or are already an actual developer working with source based Android ROMs or other similar projects you should more likely than not already have these obtained, if not here's an idea of what one should do below.

__Installing repo__

```bash
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

Now add the directory to your PATH variable in your environment (e. g. by appending it to your `~/.bashrc` and running the same command in your shell)

```bash
export PATH="$HOME/bin:$PATH"
```

__Installing git__

If you followed the Arch Linux steps above, you can go right to step 2.
Git would more likely than not probably be already installed in your distribution, if it is not then you should try one of the following terminal commands depending on your distribution:

```bash
Ubuntu, Debian (apt): apt-get install git
OpenSUSE: zypper install git
Fedora: yum install git-all
Gentoo: emerge --ask --verbose dev-vcs/git
Arch Linux: pacman -S git # this was listed above already, you can skip it
```

The derivatives of these common distributions should also have the git package available, if you believe your distribution does not offer git in the default package repositories then you may consider [compiling and installing git from source](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git#Installing-from-Source).

#### __2. Initiating Repository and Acquiring Sources__

First, create a new empty directory of your choice, and `cd` into it:

```bash
mkdir xossrc
cd xossrc
```

Of course, you can use any directory name you desire. It is recommended to avoid spaces.

If you're a curious geek and want to know what happens if you try to initialize the repo on your home folder, [see this GitHub issue](https://github.com/halogenOS/android_manifest/issues/18). It doesn't mean that it will never work - It might work, but you better avoid doing so.

Now initialize a repo source tree, to do this please use following command:

```bash
repo init -u https://git.halogenos.org/halogenOS/android_manifest.git -b XOS-12.1
```

Then synchronize the source tree using repo, which will fetch the source of XOS. You should be warned that this is a procedure which downloads huge amounts (about 30-60 GB in total) of data, it may take hours to complete. Be prepared with something fun to do as will be waiting for a while or just listen to EDM.

```bash
repo sync -j4 -c --no-tags --no-clone-bundle -f build/make external/xos vendor/halogenOS
source build/envsetup.sh
reposync
```

##### CCache

We also recommend you to use CCache for faster builds (if you don't know what CCache is, do some research about it or skip this step):

Create a file named "ccache.sh" in the source root of where you just ran reposync inside of (the example mentions "xossrc"):

```
export USE_CCACHE=1
export CCACHE_DIR=/path/to/your/ccache

# Specify your desired ccache size here. 80G is a good starting point.
ccache -M 80G
```

The `CCACHE_EXEC` variable will be automatically set based on the ccache installed on your host if you don't
set the variable yourself. Make sure you have installed ccache on your distribution.


#### __3. Building__

First, in order to build XOS you should source the `build/envsetup.sh` script in your shell, this will set up and import all of the available device configurations for the ROM as well as giving you some fancy "macro" commands for
your build environment. As such, in order to do this, run the command:

```bash
source build/envsetup.sh
```

Use following command to start a full build. You can also use `m`, `make` and sister commands to build.
If you use other commands make sure you have lunched before starting a build.

```bash
build full aosp_<device>-userdebug
```

Example:

```bash
build full aosp_cheeseburger-userdebug
```

This `build` command is a specialty made by the XOS team. It does everything for you, from lunching to initiating a new build, as well as finding out which amount of threads are optimal for your machine. Hence you must not specify a thread count using `-j` on this command, as that will be done automatically for you. **If you want to do a dirty build (i. e. skip `make clean`), simply add `noclean` to the end of your command like this:** `build full aosp_<device>-userdebug noclean`

##### Building for emulator

```
buildemu aosp_sdk_phone_x86_64-userdebug noclean
```