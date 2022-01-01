# masspkg
The upcoming package manager of MassOS which is currently IN HEAVY DEVELOPMENT.
# IN HEAVY DEVELOPMENT - NOT PRODUCTION READY
**masspkg is in HEAVY development, it is not ready yet, please do not try to use it ~~on a production system~~ at all.**
# About masspkg.
`masspkg` is the (upcoming) official package manager of the [MassOS](https://github.com/TheSonicMaster/MassOS) operating system.

It is written in Bash from the ground-up, and has syntax similar (but not identical) to `apt`.

It is designed to be flexible and uncomplicated to use for both users and package maintainers.
# Usage.
```
masspkg install - Install one or more package(s).
masspkg update  - Scan for and install package update(s).
masspkg remove  - Remove one or more package(s).
masspkg list    - List installed package(s).
masspkg info    - Get information about a particular package.
```
# Installing masspkg.
`masspkg` is designed to be heavily integrated with MassOS, and therefore is not designed to be installed by users, and you should **NEVER** install it on another distribution which uses its own package manager. With that being said, if you are a developer who wants to hack on `masspkg`, you can install it like this:

First, retrieve the source code:
```
git clone https://github.com/TheSonicMaster/masspkg.git
cd masspkg
```
Then install the program and configuration file (AS ROOT). This assumes you are using the default locations, `/usr/bin` and `/etc`.
```
install -m755 masspkg /usr/bin/masspkg
install -m644 masspkg.conf /etc/masspkg.conf
```
**NOTE:** The `masspkg` program currently has a hard-coded location to look for the configuration file: `/etc/masspkg.conf`. This will be resolved in the future, when a proper Makefile and configuration system is set up.

You can edit the configuration file `masspkg.conf` if you want to change the default package repository data directory, or builtins file. More about that below.
# How masspkg works.
The concept of `masspkg` is quite straightforward. Basically it retrieves information about a package by reading the package's **manifest** file (more about the **manifest** format below) from a remote repository, then uses that to handle things like dependency resolution, before actually downloading and installing the actual package from the same remote repository.

Despite it's straightforward nature, it supports handling of multiple arguments, automatic (and recursive) dependency resolution, knowing whether installed packages are dependencies of other packages or not (to prevent breakages), and more.

For each package, only two files are really needed; the **manifest** file (read about that below), and the actual package tarball itself, which unlike other package managers, contains no metadata of its own; only the actual binaries and libraries of the package. Package information is handled through **manifest** files, which are kept separate to the package tarball.
# The manifest format.
The information about each package is defined in its **manifest** file. A manifest file is titled **_package-name_.manifest**. Inside the manifest file are variables which define information about the package. An example manifest file (`hello.manifest`) may look like this:
```
pkgname="example-package"
pkgver="1.0.0"
description="Example program for demonstrating masspkg manifest syntax."
deps="bash glibc"
arch="x86_64"
homepage="https://github.com/TheSonicMaster/MassOS"
maintainer="John Smith <john@example.org>"
license="GPL-3.0-or-later"
```
The purpose of most variables defined in here should be quite obvious. For more in-depth details about each variable, take a look at the [example.manifest](example.manifest) file.

**It is important that manifest files do not contain any syntax errors, otherwise it could prevent masspkg from working.**
## What about programs already on the system?
Good question! There should be a file named `builtins` somewhere on your system (`/usr/share/massos/builtins` by default), which contains a list of all packages which are already pre-installed as part of the MassOS system (if you don't have one, it looks something like [this](https://github.com/TheSonicMaster/MassOS/blob/development/utils/builtins)). If your package depends on something you know is included in MassOS, you can exclude it from the `deps=` line of your manifest file if you wish; it makes no difference to `masspkg` since `masspkg` checks this `builtins` file to check if the dependency is already satisified, and in this case, it doesn't try to resolve it. On the other hand, if your package depends on something which is definitely not in the MassOS system, or may be in a newer version of MassOS but is not an older one and you are *extremely* obsessed with backwards compatibility, then it's probably better to include it as a dependency in the manifest anyway.
# How to create a package for masspkg.
## Compiling and preparing your package binary.
First, compile the package or otherwise obtain it as you normally would. When configuring, prefer the prefix to be `/usr`. When installing it, use a DESTDIR-based installation, if possible. e.g. if installing from `make`, do something like `make DESTDIR="${PWD}/pkg-out" install`. The DESTDIR directory can be anywhere you want, however `"${PWD}/pkg-out"` will install it to the **pkg-out** folder in the current working directory.

It is good practice to install all binaries and libraries to `/usr/bin`, `/usr/sbin`, or `/usr/lib` respectively. This is because MassOS uses a unified filesystem structure, where `/bin`, `/sbin`, and `/lib` are symlinks to their `/usr` counterpart. Having `/bin`, `/sbin`, and `/lib` as real directories in your DESTDIR-installed folder will probably break masspkg.

MassOS also prefers to avoid installation of libtool archives (`.la` files) and static libraries (`.a` files), unless it is absolutely necessary. This is because the majority of the MassOS system is dynamically linked. Unless you really need the libtool archives and/or static libraries of your package to be preserved, you can remove them with the following command (assuming your libraries are installed to `pkg-out/usr/lib` (the following commands will need to be modified as appropriate if these directories differ)):
```
find pkg-out/usr/lib -name \*.la -delete
find pkg-out/usr/lib -name \*.a -delete
```
It may also be benifical to strip your newly compiled and installed binaries/libraries. This removes unnecessary debugging symbols to save up disk space. Strip binaries and libraries like this (assuming your binaries are installed to `pkg-out/usr/bin` and your libraries are installed to `pkg-out/usr/lib` (the following commands will need to be modified as appropriate if these directories differ)):
```
find pkg-out/usr/bin -type f -exec strip --strip-all {} ';'
find pkg-out/usr/lib -type f -name \*.so\* -exec strip --strip-unneeded {} ';'
```
Take care **NEVER** to use `--strip-all` on the shared libraries in `/usr/lib`. It will break their ability to be dynamically linked to, and therefore effectively render them useless. `--strip-all` can, however, be safely used on the binaries in `/usr/bin` without issues.

There is a good chance that a large number of these files will complain about their file format not being recognised. This is perfectly normal, and nothing is wrong; it simply indicates that these files are scripts instead of binaries.
## Creating the binary package tarball.
Package tarballs must be in the `.tar.xz` (XZ-compressed) format. The package tarball must contain **ONLY** the installation directories and files relative to `/`, e.g. the files inside the `pkg-out` directory, if this was your DESTDIR-installed directory. You must **NOT** include the `pkg-out` directory itself, or any other containing directory. Assuming your package is DESTDIR-installed to the `pkg-out` directory, the following command will correctly create the package tarball (once again, you'll need to modify this command if your directory differs to `pkg-out`):
```
pushd pkg-out
tar -cJf ../<PACKAGE-NAME>-<PACKAGE-VERSION>-<CPU-ARCH>.tar.xz *
popd
```
The package name, version, and CPU arch in there are PLACEHOLDERS. Replace them with the actual package name, e.g. `example-package-1.0.0-x86_64.tar.xz`
## Writing the manifest file.
Now your package binary is ready, the final thing to do is write the manifest file for it. It should be named **_package-name_.manifest**, e.g. `example-package.manifest`. For writing the manifest file, follow the section above entitled "The manifest format", which explains how a manifest file is created and the syntax it uses. This ["hello" manifest file](https://massos.thesonicmaster.net/repo/x86_64/manifest/hello.manifest) file is also a good template you could modify, if you need a starting point.

Once you've written the manifest and your package is ready to be published on your repository, follow the section below entitled "How to host masspkg packages in a repository".
# How to host masspkg packages in a repository.
## Introduction to hosting.
This section will explain how to set up the filesystem structure for hosting masspkg packages in a repository. It is assumed that your web server itself is already set up and you know how to manipulate files within it. The specific protocol your web server uses can be `http`, `https` (preferred), `ftp`, or any other protocol which `curl` supports.

We'll assume your web server is called `https://repo.example.org`, and the masspkg section of the repository is at the path `/massos` (it can be a name of your choice, as long as you edit the commands below appropriately) (you could omit `/massos` entirely and just use the root path of your web server, however if you use this web server for other purposes, it is unpractical, and a dedicated directory such as `/massos` is recommended).

If it's not already, you must define the repo in `masspkg.conf`, like this:
```
repo="https://repo.example.org/massos"
```
## The repository structure and layout.
The commands in this section assume that `/massos` is the path of the masspkg repository on your web server. The below commands will need modifying slightly if this differs.

You must create a separate directory for each CPU architecture you wish to provide packages for. For example, if you are only targetting the `x86_64` architecture, you'd create only the `x86_64` directory within `/massos`.

**NOTE:** If your packages support any architecture (e.g. the architecture is set to `any` in the manifest file), you still need to put it in a specific architecture directory. You could create multiple architecture directories such as `x86_64` and `i686`, and put the package in each of them, however only `x86_64` is required, since this is the only architecture MassOS officially supports right now.

It is also optional, but not mandatory, to include a `sources` directory, where the source packages of your released binaries are stored. Source package functionality has not yet been implemented in MassOS, however it probably will be implemented in the future.

Inside the `x86_64` folder (or whatever architecture you are targetting), you need to create two directories; a `manifest` directory, and an `archives` directory. It's quite self-explanatory that `manifest` contains package manifest files, and `archives` contains the actual package tarballs.
## Publishing your packages.
Now your filesystem structure is set up, you are ready to publish your packages.

You should have the two files you created earlier, the package tarball and the manifest file. It is assumed that you have not changed the names of these files since creating them.

Place the manifest file in the `manifest` directory, and place the tarball in the `archives` directory.

That's it! Now `masspkg` should be able to install your newly created package.
## Publishing updates.
To update packages, you simply need to upload the newer package tarball (which can optionally coexist with the old one if you wish, since tarball names are versioned), and then update the manifest file as appropriate; changing the package version and any other information or dependencies which have changed.
