Copyright (c) 2004-13 Simon Peter <probono@puredarwin.org>
All rights reserved. 
Redistribution of this document is permitted only in unchanged form.
Version 2013-04-19

Building
========

Use an old system for building (at least 2-3 years old) to ensure the binaries run on older systems too.

```bash
sudo apt-get update ; sudo apt-get -y install libfuse-dev libglib2.0-dev cmake
cd AppImageKit-master
cmake .
make
```

Once you have built AppImageKit, try making an AppImage, e.g., of XChat:
    ./apt-appdir/apt-appdir xchat && ./AppImageAssistant xchat.AppDir XChat.AppImage
(This is just a proof-of-concept, of in reality you should use AppDirAssistant to create proper AppDirs)

TODO
====

* Include AppDirAssistant tool
* Update http://www.portablelinuxapps.org/docs/1.0/ to remove all references to elficon; include changelog section

Changelog
=========

-11
* Builds on x86-64
* Follow http://specifications.freedesktop.org/thumbnail-spec/thumbnail-spec-latest.html (Version 0.8.0 introduced new location for the thumbnails)
* Remove elficon as no one uses it; use thumbnails instead
* Include AppImageExtract tool
* Update AppImageAssistant.AppDir so that it runs on latest Ubuntu
* Use AppDir suffix
* Factor out dependency binaries to be bundled into ./bundled-dependencies-i386

-9
* Runtime extracts .DirIcon to $HOME/.thumbnails/normal/
* When called with "--icon", only extracts the icon, prints a message and exits
* Runtime sets environment variables APPIMAGE, APPDIR that the "inside" app may use
* AppRun appends to environment variables XDG_DATA_DIRS, QT_PLUGIN_PATH, PERLLIB

AppImage format definition
==========================

The AppImage format has the following properties:
* The AppImage is an ISO9660 file
* The contents of the ISO9660 file can be compressed with zf
* In the first 32k of the ISO9660 file, an ELF executable is embedded
  which mounts the AppImage, executes the application from the 
  mounted AppImage, and afterwards unmounts the AppImage again
* The ISO9660 file contains an AppDir as per the ROX AppDir specification
* The AppDir contains one desktop file as per the freedesktop.org specification

```
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
+                                                                                  +
+   ISO9660                                                                        +
+                                                                                  +
+   ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++   +
+   +                              +                                           +   +
+   +                              +                AppDir                     +   +
+   +                              +                 |_ .DirIcon               +   +
+   +                              +                 |_ appname.desktop        +   +
+   +             ELF              +                 |_ AppRun                 +   +
+   +                              +             (   |_ usr/                   +   +
+   +                              +                     |_ bin/               +   +
+   +                              +                     |_ lib/               +   +
+   +                              +                     |_ share/  )          +   +
+   +                              +                                           +   +
+   ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++   +
+                                                                                  +
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

Creating portable AppImages
===========================

For an AppImage to run on most systems, the following conditions need to be met:
1. The AppImage needs to include all libraries and other dependencies that are not part of all of the base systems that the AppImage is intended to run on
2. The binaries contained in the AppImage need to be compiled on a system not newer than the oldest base system that the AppImage is intended to run on
3. The AppImage should actually be tested on the base systems that it is intended to run on

Libraries and other dependencies
--------------------------------

Binaries compiled on old enough base system
--------------------------------------------

The ingredients used in your AppImage should not be built on a more recent base system than the oldest base system your AppImage is intended to run on. Some core libaries, such as glibc, tend to break compatibility with older base systems quite frequently, which means that binaries will run on newer, but not on older base systems than the one the binaries were compiled on.

If you run into errors like this

    failed to initialize: /lib/tls/i686/cmov/libc.so.6: version `GLIBC_2.11' not found

then the binary is compiled on a newer system than the one you are trying to run it on. You should use a binary that has been compiled on an older system. Unfortunately, the complication is that distributions usually compile the latest versions of applications only on the latest systems, which means that you will have a hard time finding binaries of bleeding-edge softwares that run on older systems.

Testing
-------

To ensure that the AppImage runs on the intended base systems, it should be thoroughly tested on each of them. The following testing procedure is both efficient and effective: Get the previous version of Ubuntu, Fedora, and openSUSE Live CDs and test your AppImage there. Using the three largest distributions increases the chances that your AppImage will run on other distributions as well. Using the previous (current minus one) version ensures that your end users who might not have upgraded to the latest version yet can still run your AppImage. Using Live CDs has the advantage that unlike installed systems, you always have a system that is in a factory-fresh condition that can be easily reproduced. Most developers just test their software on their main working systems, which tend to be heavily customized through the installation of additional packages. By testing on Live CDs, you can be sure that end users will get the best experience possible.

I have a setup where I use the compressed filesystems of Live CDs, loop-mount them, chroot into them, and run the contents of the AppImage there. This way, I need approximately 700 MB per supported base system (distribution) and can easily upgrade to never versions by just exchanging one file.

Bundling shell scripts
======================

Shell scripts occasionally hardcode paths to other files. This will break if these other files are relocated. Hence, you should refer to outside files always relative to the position of the shell script itself. This can be achieved with the following code:

```bash
#!/bin/sh
HERE=$(dirname $(readlink -f "${0}"))
echo "${HERE}"
```

This will always return the absolute path to the directory the script is located in, regardless of how it was called.

Assume you have a script /usr/bin/foo which wants to call /usr/bin/bar. 

To make it relocateable, you need to change /usr/bin/foo so that instead of calling
    /usr/bin/bar --dosomething
it contains
    "${HERE}/bar" --dosomething

Bundling Python apps
====================

Due to the way Python imports work and due to the way Python packages are installed on Debian and Ubuntu, it is not trivial to create working AppDirs from them. 

To find out what Python files a Python app (let's say "SpiderOak") accesses, you can use

    strace -eopen -f ./SpiderOak 2>&1 | grep / | grep -v ENOENT | cut -d "\"" -f 2 | sort | uniq > openedfiles

Instead, it is advised that you use workingenv.py, a script that sets up an isolated environment into which you can install your Python app and its dependencies that are not part of the standard library. 

```bash
wget http://svn.colorstudy.com/home/ianb/workingenv/workingenv.py
python workingenv.py MyNewEnvironment
cd MyNewEnvironment/
SITEPY=$(find -name site.py | head -n 1)
cat >> $SITEPY.new <<EOF
global USER_BASE, USER_SITE, ENABLE_USER_SITE
USER_BASE = None
USER_SITE = None
EOF
cat $SITEPY >> $SITEPY.new
mv $SITEPY.new $SITEPY
source easy_install bin/activate
cd src/
# wget ...
tar xzf * ; cd * ; python setup.py install
# or
easy_install *.egg
```

Then, in your AppRun file, set $PYTHONPATH to point at MyNewEnvironment/lib/python2.6

Please let me know if there is an easier way to bundle Python apps properly.

Bundling Perl apps
====================

If a Perl app refuses to run, the first thing is to increase verbosity by adding the following code at the top of the Perl script:

    use strict;
    use warnings;

Then watch out for messages like "Can't locate ... in @INC"

Then set $PERLLIB so that Perl can find its includes.

Distribution specific notes
===========================

 * openSUSE lacks libpcre.so.3
 * Fedora lacks libXp.so.6
 * Fedora and openSUSE lack libtiff.so.4 (but can be symlinked cleverly)

Support
=======

I support open source projects that wish to distribute their software as an AppImage. For closed source applications, I offer AppImage packaging and testing as a service.
