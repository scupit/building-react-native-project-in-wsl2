# Building a React Native Project in WSL2

This is the list of steps and resources I used to get a React Native project build working on WSL2 Ubuntu
using Windows 11 running an Android emulator. There are quite a few little annoying nuances to getting
this set up, so I decided to write this document detailing the process in case I ever need to do the
same setup steps again.

For reference, the working build was using *react-native 0.70.0* and *react 18.2.0*.

## Resource List

### Guides and Intros

- [**General mostly comprehensive guide**](https://gist.github.com/bergmannjg/461958db03c6ae41a66d264ae6504ade)
- [Improving build speed](https://reactnative.dev/docs/next/build-speed)
- [New architecture info](https://reactnative.dev/docs/new-architecture-intro)
- [Configuring adb server socket](https://stackoverflow.com/questions/72078857/fix-android-studio-react-native-wsl-wont-launch-emulator-with-more-errors)

### General

- [React Native new architecture info](https://reactnative.dev/docs/new-architecture-intro)
- [React Native environment setup](https://reactnative.dev/docs/environment-setup)
- [Android Studio command line tools](https://developer.android.com/studio#command-tools)
- [OpenJDK General-Availability release archive](https://jdk.java.net/archive/)

### Fixes

- [EACCES error fix](https://stackoverflow.com/questions/54541734/spawnsync-gradlew-eacces-error-when-running-react-native-project-on-emulator)

## Configuring Windows

1. Make sure Android Studio is installed and the
  [React Native Windows development environment is set up correctly](https://reactnative.dev/docs/environment-setup).

2. Add a firewall rule which allows WSL to connect to servers hosted on the Windows host machine.
  See <https://github.com/microsoft/WSL/issues/4585> for details.

``` powershell
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow
```

Also **make sure inbound adb requests aren't blocked for public networks** on Windows.
**WSL's ethernet adapter (`vEthernet (WSL)` on the Windows machine) is considered a public network**,
so any requests from WSL2 to a server running on Windows will fail if the server executable
(node.exe, adb.exe, etc.) has an inbound firewall rule blocking inbound requests from public networks.
**Same goes for all servers such as docker, node, etc.**

## Configuring WSL2

1. [Set up CCache](https://reactnative.dev/docs/build-speed#use-a-compiler-cache). A clean build takes a long
  time. ccache helps alleviate some of that time.
2. Make sure `unzip` and `socat` are installed using `sudo apt install unzip socat`
3. Download the [Linux command line tools](https://developer.android.com/studio#command-tools)
4. Unpack the archive using `unzip`
5. `mkdir -p ~/Android/Sdk/cmdline-tools`
6. Move the unpacked directory to `~/Android/Sdk/cmdline-tools/latest`
7. Download OpenJDK 11+ from the [OpenJDK archive](https://jdk.java.net/archive/).
8. Unpack and move the unpacked jdk directory to `/opt/jdk-11.0.2`. (You can name the directory anything,
  however naming it `jdk-<the jdk version>` is easiest to understand).
  [React Native environment setup](https://reactnative.dev/docs/environment-setup)
  recommends JDK 11, however later JDKs usually work fine. `/opt/<your-jdk-dirname>/bin` should contain java
  and the rest of the jdk tools if you moved the correct directory.
9. Run `~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager "platform-tools"` to install **adb** and other necessary tools.
10. Add these lines to *~/.bashrc*:

``` bash
# WSL2 is run in a VM, so its localhost is unable to access the Windows host machine's
# localhost. Instead, WSL_HOST will store the IP address which WSL2 uses to point to
# the host Windows machine.
# See https://stackoverflow.com/questions/72078857/fix-android-studio-react-native-wsl-wont-launch-emulator-with-more-errors
export WSL_HOST=$(tail -1 /etc/resolv.conf | cut -d' ' -f2)
export ADB_SERVER_SOCKET=tcp:$WSL_HOST:5037

export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_SDK_ROOT="$HOME/Android/Sdk"
# This assumes you downloaded OpenJDK 11.0.2 and moved it
# into /opt/jdk-11.0.2 .
# Use the directory name you moved your downloaded jdk into, if different.
export JAVA_HOME="/opt/jdk-11.0.2"
export PATH=$PATH:$ANDROID_SDK_ROOT/platform-tools
export PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin
```

## Configuring the project

If using the [new React Native architecture](https://reactnative.dev/docs/new-architecture-intro),
increase the amount of memory given to the gradle build by adding `org.gradle.jvmargs=-Xmx8G` or
some other large amount of memory to the build. If you don't do this, the build will likely
fail with an *OutOfMemoryError* at the very last step. Super annoying.

## Running the build

1. Start emulator on windows
2. On Windows, `adb kill-server` then `adb -a nodaemon server start`
3. In WSL2, `socat -d -d TCP-LISTEN:5037,reuseaddr,fork TCP:$(cat /etc/resolv.conf | tail -n1 | cut -d " " -f 2):5037` ([this page](https://gist.github.com/bergmannjg/461958db03c6ae41a66d264ae6504ade) saves us again)
4. In WSL2, start the Metro server with `yarn react-native start --host 127.0.0.0`
5. In WSL2, start the build with `yarn react-native run-android --variant=debug --active-arch-only`

## Build Issue FAQ

### adb version mismatch

The `adb` executable must be the same version in both Windows and WSL2. To get the latest version,
run `sdkmanager "platform-tools"`.

For example, if the first line of output from running `adb version` is

> Android Debug Bridge version **1.0.41**

then both your Windows and WSL2 adb executables must output that same version. Differing versions will
result in an error.

### Build fails with EACCES when starting gradlew

Try running `chmod 755 android/gradlew`. The gradle wrapper probably just has the wrong permissions.

See [this stackoverflow answer](https://stackoverflow.com/questions/54541734/spawnsync-gradlew-eacces-error-when-running-react-native-project-on-emulator)
for info.

### OutOfMemoryError

When using *React Native 0.68.0+*, both React Native and the Hermes engine are built from source as
part of the app build process. For some reason, this is causing an OutOfMemoryError on WSL when the default
amount of memory is allocated to the gradle build.

**Solution:** Add more memory to the gradle build. Allocating a large amount of memory
(ex: `org.gradle.jvmargs=-Xmx8G`) for the gradle build in *YOUR_PROJECT_ROOT/android/gradle.properties*
should do the trick.

### The build failed with a weird Gradle Error

Until the build works, it sometimes just fails for no reason. For errors like "cannot get peer dependency
due to lack of authentication", just try running the build again.
