![](https://git.halogenos.org/halogenOS/android_manifest/-/raw/XOS-15.2/halogenos-logo.png)

### Getting Started with XOS

#### __0. Preliminary Knowledge__

Before beginning this entire process, please ensure you have sufficient storage space and RAM.
You should expect to need at least 200 GB of storage for sources and a full build, although a minimum of 500 GB of
available space is recommended. Avoid using SSDs with short lifetime, such as TLC/QLC SSDs that have little capacity.

Before you continue, make sure you read the [Setting up a Linux build environment](https://source.android.com/source/initializing.html#setting-up-a-linux-build-environment) guide as it contains a lot of useful and important information regarding building AOSP.

You should familiarize yourself with all the AOSP basics on the [AOSP documentation](https://source.android.com/docs) page.

#### __1. Getting Started__

Make sure you have an environment suitable for building AOSP. If you use NixOS or Nix, you can follow the guidance
in step 3.

For the sake of brevity and avoiding outdated information, we will no longer include instructions on how to set up
a build environment on distributions like Debian, Ubuntu, Fedora, Arch, etc.

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
repo init -u https://git.halogenos.org/halogenOS/android_manifest.git -b XOS-15.2
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

We use NixOS and Nix for setting up a development and build environment suitable for AOSP.
To use this, you can run `nix develop path:external/xos/devshell` to enter said environment which will already
have everything installed so you can get started right away, batteries included.

In case you have `direnv` installed and set up in your shell, you can `direnv allow` our `.envrc` and then
simply run `aosp-env` which will be equivalent to running the `nix` command mentioned previously.

First, in order to build XOS, you should source the `build/envsetup.sh` script in your shell.
This will set up your environment so that you can start building.

```bash
source build/envsetup.sh
```

Use following command to start a full build. You can also use `m`, `make` and sister commands to build.
If you use other commands make sure you have lunched before starting a build.

```bash
build full aosp_<device>-bp1a-userdebug
```

Example:

```bash
build full aosp_Pong-bp1a-userdebug
```

This `build` command is a specialty made by the XOS team. It does everything for you, from lunching to initiating a new build, as well as finding out which amount of threads are optimal for your machine. Hence you must not specify a thread count using `-j` on this command, as that will be done automatically for you. **If you want to do a dirty build (i. e. skip `make clean`), simply add `noclean` to the end of your command like this:** `build full aosp_<device>-bp1a-userdebug noclean`
