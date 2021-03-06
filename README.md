# How to setup icecream to compile mozilla-central on mac

All the files/scripts below can be found in this repository.

At time of writing, brew installs llvm/clang 5.0 (clang version 5.0.0 (tags/RELEASE_500/final))

## Make sure you are running the latest version of icecream

Ubuntu ships an old icecream version released in 2013: version 1.0.1. You will find the latest release (1.1) packages for ubuntu/debian in the packages directory.

## Getting icecream installed and running on Mac

```bash
$ brew install icecream
```

start iceccd manually with:
```bash
$ sudo /usr/local/sbin/iceccd -d -s HOSTNAME
```

Replace HOSTNAME with the hostname the icecc scheduler is running on.

Alternatively, to get the icecream iceccd daemon running at boot time
copy the file
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>Icecc Daemon</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/Cellar/icecream/1.1/sbin/iceccd</string>
        <string>-s</string>
        <string>HOSTNAME_SCHEDULER</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>UserName</key>
    <string>root</string>
</dict>
</plist>
```

into /Library/LaunchDaemons/com.mozilla.iceccd.plist
There too, replace HOSTNAME with the hostname the icecc scheduler is running on.

then run:
```bash
$ sudo launchctl load /Library/LaunchDaemons/com.mozilla.iceccd.plist
```

## Getting llvm-5.0 installed and running on Mac

```bash
$ brew install llvm
```

## Getting toolchain packaged for icecream

You can directly download the archives here and skip this section.
https://github.com/jyavenard/mozilla-icecream/raw/master/clang-5.0.0-Darwin17_x86_64.tar.gz
and:
https://github.com/jyavenard/mozilla-icecream/raw/master/clang-5.0.0-x86_64.tar.gz


Get the latest mac clang build at http://releases.llvm.org/download.html

Current it’s 5.0.0 available at:
mac: http://releases.llvm.org/5.0.0/clang+llvm-5.0.0-x86_64-apple-darwin.tar.xz
linux 64 bits: http://releases.llvm.org/5.0.0/clang+llvm-5.0.0-linux-x86_64-ubuntu16.04.tar.xz
(tested with ubuntu 16.04 and 17.10 only)

In the following instructions, I placed everything into ~/clang for ease of documentation, you can of course place it in any location.

### On mac:
```bash
$ mkdir ~/clang
$ cd ~/clang
```

Extract the archive and create the icecream toolchain archive
```bash
$ wget http://releases.llvm.org/5.0.0/clang+llvm-5.0.0-x86_64-apple-darwin.tar.xz
$ tar xvf clang+llvm-5.0.0-x86_64-apple-darwin.tar.xz
$ /usr/local/Cellar/icecream/1.1/libexec/icecc/icecc-create-env --clang ~/clang/clang+llvm-5.0.0-x86_64-apple-darwin/bin/clang /usr/local/Cellar/icecream/1.1/libexec/icecc/compilerwrapper
```

this will create a file such ce3126a6676d8b9cd58ea709615bee97.tar.gz
rename it into clang-5.0.0-Darwin17_x86_64.tar.gz

### On Linux
```bash
$ mkdir ~/clang
$ cd ~/clang
```

Extract the archive and create the icecream toolchain archive
```bash
$ wget http://releases.llvm.org/5.0.0/clang+llvm-5.0.0-linux-x86_64-ubuntu16.04.tar.xz
$ tar xvf ~/Download/clang+llvm-5.0.0-linux-x86_64-ubuntu16.04.tar.xz
$ /usr/lib/icecc/icecc-create-env --clang ~/clang/clang+llvm-5.0.0-linux-x86_64-ubuntu16.04/bin/clang /usr/lib/icecc/compilerwrapper
```

import the generated tar.gz into your mac and rename it as clang-5.0.0-x86_64.tar.gz, place it into ~/clang

## Prepare your central tree

You need to apply the following patch for now:
```
diff --git a/media/ffvpx/ffvpxcommon.mozbuild b/media/ffvpx/ffvpxcommon.mozbuild
index fdca987f49ef..4577d1440b9d 100644
--- a/media/ffvpx/ffvpxcommon.mozbuild
+++ b/media/ffvpx/ffvpxcommon.mozbuild
@@ -66,6 +66,7 @@ if CONFIG['GNU_CC']:
             '-Wno-incompatible-pointer-types-discards-qualifiers',
             '-Wno-string-conversion',
             '-Wno-visibility',
+            '-ffreestanding',
         ]
     else:
         CFLAGS += [
diff --git a/old-configure.in b/old-configure.in
index 849146516063..f99ceb3a16cd 100644
--- a/old-configure.in
+++ b/old-configure.in
@@ -4790,10 +4790,10 @@ AC_SUBST(MOZ_SYSTEM_NSPR)
 
 AC_SUBST(MOZ_SYSTEM_NSS)
 
-HOST_CMFLAGS=-fobjc-exceptions
-HOST_CMMFLAGS=-fobjc-exceptions
-OS_COMPILE_CMFLAGS=-fobjc-exceptions
-OS_COMPILE_CMMFLAGS=-fobjc-exceptions
+HOST_CMFLAGS="-x objective-c -fobjc-exceptions"
+HOST_CMMFLAGS="-x objective-c++ -fobjc-exceptions"
+OS_COMPILE_CMFLAGS="-x objective-c -fobjc-exceptions"
+OS_COMPILE_CMMFLAGS="-x objective-c++ -fobjc-exceptions"
 if test "$MOZ_WIDGET_TOOLKIT" = uikit; then
   OS_COMPILE_CMFLAGS="$OS_COMPILE_CMFLAGS -fobjc-abi-version=2 -fobjc-legacy-dispatch"
   OS_COMPILE_CMMFLAGS="$OS_COMPILE_CMMFLAGS -fobjc-abi-version=2 -fobjc-legacy-dispatch"
```

As of central 2017-11-28 you no longer need to modify old-configure.in

## Configure your mozbuild for using icecream

This is the .mozconfig that I use:

```
mk_add_options AUTOCLOBBER=1

ac_add_options --enable-debug-symbols

# Enable debug builds
ac_add_options --enable-debug

# Turn off compiler optimization. This will make applications run slower,
# will allow you to debug the applications under a debugger, like GDB.
ac_add_options --disable-optimize

#ac_add_options --with-ccache=/opt/local/bin/ccache

CC="/usr/local/opt/llvm/bin/clang --target=x86_64-apple-darwin16.0.0 -mmacosx-version-min=10.11"
CXX="/usr/local/opt/llvm/bin/clang++ --target=x86_64-apple-darwin16.0.0 -mmacosx-version-min=10.11"

ac_add_options --with-compiler-wrapper="/usr/local/bin/icecc"
```

## Start the build

For some reasons, when using icecream remote host, configure takes over 4.5 minutes to run. As such I run configure on the local host only first with:
```bash
ICECC_VERSION="Darwin17_x86_64:$HOME/clang/clang-5.0.0-Darwin17_x86_64.tar.gz" ./mach configure
```

then we can continue the build:
```bash
ICECC_VERSION="x86_64:$HOME/clang/clang-5.0.0-x86_64.tar.gz,Darwin17_x86_64:$HOME/clang/clang-5.0.0-Darwin17_x86_64.tar.gz" ./mach build -j32
```

You may find that only using the remote linux host makes your compilation much quicker, to do so, remove the Darwin17_x86_64 entry from ICECC_VERSION:
```bash
ICECC_VERSION="x86_64:$HOME/clang/clang-5.0.0-x86_64.tar.gz" ./mach build -j32
```

A note of caution, the paths in ICECC_VERSION can't be symbolic links.
