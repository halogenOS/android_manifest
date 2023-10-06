![](https://git.halogenos.org/halogenOS/android_manifest/-/raw/XOS-15.0/halogenos-logo.png)

### Getting Started with XOS

#### __0. Preliminary Knowledge__

Before beginning this entire process, please ensure you have sufficient storage space and RAM.
You should expect to need at least 200 GB of storage for sources and a full build, although a minimum of 500 GB of
available space is recommended. Avoid using SSDs with short lifetime, such as TLC/QLC SSDs that have little capacity.

Before you continue, make sure you follow the [Setting up a Linux build environment](https://source.android.com/source/initializing.html#setting-up-a-linux-build-environment) guide as it contains a lot of useful and important information regarding building AOSP.

You should familiarize yourself with all the AOSP basics on the [Android OS Documentation](https://source.android.com/docs) page.

#### Arch builders, ahoy!

We recommend building on Arch as that is what we use for daily building and development.
You can install all necessary packages using following commands (let us know if there's anything missing):

```
sudo pacman -Syu --needed --noconfirm \
      base-devel bc ccache curl git gnupg \
      inetutils iputils net-tools libxslt ncurses \
      repo rsync squashfs-tools unzip \
      zip zlib ffmpeg lzop ninja pngcrush openssl \
      gradle maven libxcrypt-compat xmlstarlet \
      openssh imagemagick jq
```

Also you have to install `ncurses5-compat-libs` from AUR using your favorite package manager  (E.g. `yay`, `aura`).

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

On Arch, you can also use the `repo` package.

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

#### __2. Initiating Repository and Acquiring Sources__

First, create a new empty directory of your choice, and `cd` into it:

```bash
mkdir xossrc
cd xossrc
```

Of course, you can use any directory name you desire. It is recommended to avoid spaces.

Please, do **not** initialize repo in your home directory. **Always** use a subfolder, like mentioned above.

Now initialize a repo source tree, to do this please use following command:

```bash
repo init -u https://git.halogenos.org/halogenOS/android_manifest.git -b XOS-15.0
```

Then synchronize the source tree using repo, which will fetch the source of XOS. You should be warned that this is a
procedure which downloads a lot (about 30-60 GB in total) of data, so it may take hours to complete.

```bash
repo sync -j4 -c --no-tags --no-clone-bundle build/make external/xos product/halogenOS
source build/envsetup.sh
reposync
```

##### CCache

We also recommend you to use CCache for faster builds (if you don't know what CCache is, do some research about it or skip this step):

Create a file named "ccache.sh" in the source root of where you just ran reposync inside of (the example mentions `xossrc`):

```
export USE_CCACHE=1
export CCACHE_DIR=/path/to/your/ccache

# Specify your desired ccache size here. 80G is a good starting point.
ccache -M 80G
```

The `CCACHE_EXEC` variable will be automatically set based on the ccache installed on your host if you don't
set the variable yourself. Make sure you have installed ccache on your distribution.

It's recommended to place CCache on a separate SSD to take advantage of the full speed such a separate drive provides.

#### __3. Building__

First, in order to build XOS, you should source the `build/envsetup.sh` script in your shell.
This will set up your environment so that you can start building.

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
