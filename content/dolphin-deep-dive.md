+++
title = "A deep dive with Dolphin"
date = 2019-12-11
draft = false

[taxonomies]
tags = ["linux", "manjaro", "systemd"]
categories = ["linux"]

[extra]
toc = true
+++

or, how systemd broke my file manager.

### TL;DR

_Systemd_ broke my file manager and I lost about 4 hours investigating it. But I suffered less than
if I was using Windows. Jump to the [conclusion](#conclusion) to see why.

### A bit about my linux background

I'm a long time linux desktop user (since 1998) and other than a few times where I did adventure into
linux internals (for learning, programming or for fun), I can't remember the last time I actually had to
do it to solve anything.

I did have moments where I had fun tuning my system for no sensible reason, including custom configured
kernels with minor personal patches here and there. I also did have my Gentoo years where everything was
statically built with custom compiler flags and tweaks for "performance". 

As a system administrator, I did maintain some servers (as work and at home). But maybe in the last 10
years my adventures on linux land where basically as a desktop user and the very few times where I needed
to deal with services, my muscle memory for _init_ always got me in trouble with _systemd_. I know almost 
nothing about _systemd_ and I have to [ddg](https://duckduckgo.com/) it every time.

Currently I use [Manjaro Linux](https://manjaro.org/), a distro based on [Arch Linux](https://www.archlinux.org/),
and I've always been a [KDE](https://kde.org/) user. And I know for a fact that _KDE_ and [Qt](https://www.qt.io/download-open-source)
are some incredible and unique pieces of technology.

Unfortunately I'm in a weird position where I'm required to use Windows professionally as I'm a game
developer. ¯\\\_(ツ)\_/¯

In all my years as a home linux desktop user, I barely had any issues. Until recently.

### What is this about

I decided to write this post to myself and to document my steps investigating a problem that have been
occurring to me in the last few weeks, which I've been delaying to check. After reading the
[O(n^2), again, now in WMI](https://randomascii.wordpress.com/2019/12/08/on2-again-now-in-wmi/) post
by **Bruce Dawson** on how he investigated and found the reason of multi-minute delays on his workstation,
I thought "why not try the same with my issue and see where this takes me? It could be interesting...". 
Here is it, and I hope it is.

> As I work with windows and my work deal in part with performance optimizations and debugging,
> Bruce Dawson's blog is a must-read.

### The problem

I've been hit for a long delay opening [Dolphin](https://kde.org/applications/system/org.kde.dolphin)
(the _KDE_'s file manager) which could take up to one minute to show up. As being something essential
as it is, this started to really annoy me and a bit of searching didn't reveal anything obvious but
some similarly looking cases. I believe this started since my last distro update which was recent (2019-11).

### Taking the dive

Even with similar _Dolphin_ issues reported over different distros forums, I was unable to have any actual
hints about my own issue. So I did go search directly into _KDE_ bugzilla for some similarly looking bugs, but
only a few had the same symptoms and not much more information. What most had in common was that the
responder always asked for a backtrace, obviously. 

But before trying that, I did the simpler stuff first.
Running `dolphin` directly from _Konsole_ to see if some log output appears.

```bash
➜  ~ dolphin
QObject::connect: No such signal QDBusAbstractInterface::DeviceAdded(QString)
QObject::connect: No such signal QDBusAbstractInterface::DeviceRemoved(QString)
kf5.kio.core: "Could not enter folder tags:/."
```

this only reveals two signal/slot signature mismatch. This is the _Qt_ way to tell
that a callback _Dolphin_ is trying to setup has a bad function signature somewhere, 
which can cause some functionality to not work correctly.

This also means that _Dolphin_ try to listen when a device is added or removed using `QDBus`, which is _Qt_'s 
interface to the [dbus](https://www.freedesktop.org/wiki/Software/dbus/) system used for [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication).

My first reaction was to ask myself if it could be locked waiting a reply from some storage device?
Let's see what `strace` has to say about it:

```bash
➜  ~ strace dolphin
<lots of output>
statx(AT_FDCWD, "/etc/fstab", AT_STATX_SYNC_AS_STAT, STATX_ALL, 0x7ffc3c621890) = -1 ENOSYS (Function not implemented)
newfstatat(AT_FDCWD, "/etc/fstab", {st_mode=S_IFREG|0644, st_size=820, ...}, 0) = 0
inotify_add_watch(12, "/etc/fstab", IN_MODIFY|IN_ATTRIB|IN_MOVED_FROM|IN_MOVED_TO|IN_DELETE_SELF|IN_MOVE_SELF) = 1
access("/run/udev/control", F_OK)       = 0
socket(AF_NETLINK, SOCK_RAW|SOCK_CLOEXEC|SOCK_NONBLOCK, NETLINK_KOBJECT_UEVENT) = 14
getpid()                                = 29328
gettid()                                = 29328
getrandom("\x0d\xd4\x7d\xd7\xce\xab\xa3\xb8\x3b\x75\x8a\x49\xa8\xd8\xe9\xb7", 16, GRND_NONBLOCK) = 16
getrandom("\xcf\xfb\xfe\xd6\xf2\x00\x03\x4d\xbe\xa0\xbc\xb9\x4a\x4a\x52\xbb", 16, GRND_NONBLOCK) = 16
getrandom("\xec\x43\xc1\x92\xa2\x3e\xcb\xfd\x5b\x34\x10\x54\xb2\x67\xf7\x25", 16, GRND_NONBLOCK) = 16
setsockopt(14, SOL_SOCKET, SO_ATTACH_FILTER, {len=29, filter=0x7ffc3c620aa0}, 16) = 0
setsockopt(14, SOL_SOCKET, SO_PASSCRED, [1], 4) = 0
bind(14, {sa_family=AF_NETLINK, nl_pid=0, nl_groups=0x000002}, 12) = 0
getsockname(14, {sa_family=AF_NETLINK, nl_pid=29328, nl_groups=0x000002}, [12]) = 0
write(5, "\1\0\0\0\0\0\0\0", 8)         = 8
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x7ffc3c6219d0, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
write(5, "\1\0\0\0\0\0\0\0", 8)         = 8
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x7fb56853f420, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7ffc3c6217d0, FUTEX_WAIT_PRIVATE, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x55e72722f180, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
futex(0x55e72722f130, FUTEX_WAKE_PRIVATE, 1) = 0
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x7fb56853f420, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7ffc3c621620, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x7fb56853f420, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0x7ffc3c621620, FUTEX_WAIT_PRIVATE, 0, NULL) = -1 EAGAIN (Resource temporarily unavailable)
write(7, "\1\0\0\0\0\0\0\0", 8)         = 8
futex(0x55e72722ac90, FUTEX_WAIT_PRIVATE, 0, NULL^C) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
<delay, ctrl+c>
```

This show us that we're waiting for something.
Most of the output shows some communication happening between user-space and kernel-space, which could be _dbus_ working.
_futex_ is a syscall for user space lock used for shared-memory synchronization, _IPC_, etc.

Alright, so it is time to grab that backtrace. Lets launch within _GDB_ and see what we can get from it. It may give some further hints at
what exactly it is waiting for:

```bash
➜  ~ gdb dolphin
GNU gdb (GDB) 8.3.1

<...output...>

Reading symbols from dolphin...
(No debugging symbols found in dolphin)
(gdb) r
Starting program: /usr/bin/dolphin 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
[New Thread 0x7fffefb45700 (LWP 31973)]
[New Thread 0x7fffeed96700 (LWP 31974)]

<wait a bit and hit ctrl+c to get back at gdb>
^C
Thread 1 "dolphin" received signal SIGINT, Interrupt.
0x00007ffff438ac45 in pthread_cond_wait@@GLIBC_2.3.2 () from /usr/lib/libpthread.so.0
(gdb) bt
#0  0x00007ffff438ac45 in pthread_cond_wait@@GLIBC_2.3.2 () from /usr/lib/libpthread.so.0
#1  0x00007ffff5b3e610 in QWaitCondition::wait(QMutex*, QDeadlineTimer) () from /usr/lib/libQt5Core.so.5
#2  0x00007ffff5b3e702 in QWaitCondition::wait(QMutex*, unsigned long) () from /usr/lib/libQt5Core.so.5
#3  0x00007ffff5ffd5cd in ?? () from /usr/lib/libQt5DBus.so.5
#4  0x00007ffff5fadfa3 in ?? () from /usr/lib/libQt5DBus.so.5
#5  0x00007ffff5fae7ce in ?? () from /usr/lib/libQt5DBus.so.5
#6  0x00007ffff5fb9e1e in ?? () from /usr/lib/libQt5DBus.so.5
#7  0x00007ffff5fb9f86 in QDBusInterface::QDBusInterface(QString const&, QString const&, QString const&, QDBusConnection const&, QObject*) () from /usr/lib/libQt5DBus.so.5
#8  0x00007ffff753c8a7 in ?? () from /usr/lib/libKF5Solid.so.5
#9  0x00007ffff74bf918 in ?? () from /usr/lib/libKF5Solid.so.5
#10 0x00007ffff74c23ed in ?? () from /usr/lib/libKF5Solid.so.5
#11 0x00007ffff74c2536 in ?? () from /usr/lib/libKF5Solid.so.5
#12 0x00007ffff74c261b in Solid::DeviceNotifier::instance() () from /usr/lib/libKF5Solid.so.5
#13 0x00007ffff74c0d69 in Solid::Device::Device(QString const&) () from /usr/lib/libKF5Solid.so.5
#14 0x00007ffff7b06e0c in ?? () from /usr/lib/libKF5KIOFileWidgets.so.5
#15 0x00007ffff7b0e2ab in ?? () from /usr/lib/libKF5KIOFileWidgets.so.5
#16 0x00007ffff7b0e3ed in ?? () from /usr/lib/libKF5KIOFileWidgets.so.5
#17 0x00007ffff7b0fe73 in KFilePlacesModel::KFilePlacesModel(QString const&, QObject*) () from /usr/lib/libKF5KIOFileWidgets.so.5
#18 0x00007ffff7ee75dc in ?? () from /usr/lib/libkdeinit5_dolphin.so
#19 0x00007ffff7ee76fc in ?? () from /usr/lib/libkdeinit5_dolphin.so
#20 0x00007ffff7ee10f3 in ?? () from /usr/lib/libkdeinit5_dolphin.so
#21 0x00007ffff7ee89f3 in ?? () from /usr/lib/libkdeinit5_dolphin.so
#22 0x00007ffff7ee8c48 in ?? () from /usr/lib/libkdeinit5_dolphin.so
#23 0x00007ffff7eea934 in ?? () from /usr/lib/libkdeinit5_dolphin.so
#24 0x00007ffff7eeab7e in ?? () from /usr/lib/libkdeinit5_dolphin.so
#25 0x00007ffff7eeaf8f in ?? () from /usr/lib/libkdeinit5_dolphin.so
#26 0x00007ffff7ecba55 in kdemain () from /usr/lib/libkdeinit5_dolphin.so
#27 0x00007ffff7ceb153 in __libc_start_main () from /usr/lib/libc.so.6
#28 0x000055555555505e in _start ()
(gdb) 
```

Oh well, not much useful right? No debug symbols available, but at least we can see a few more
interesting things there.

1. Looks like _Dolphin_ is trying to create a `KFilePlacesModel`;
2. _Dolphin_ handed the work over to something called _Solid_ in a _libKF5Solid_;
3. which is creating a `DeviceNotifier` instance;
4. We're in fact inside a wait condition inside `QDBusInterface` call;

Still not much useful, but looks like while _Dolphin_ is booting up it will fill the UI _FilePlaces_ with _Devices_.
This is the left side panel in _Dolphin_, where it will show your favorite places and storage devices.
It kind of makes sense, _Dolphin_ needs to query which devices are available to show up, so maybe I 
have a faulty device that is locking on a _dbus_ query?

With these new information, I checked my storage and any network shares via _Konsole_, but everything
looked fine and working.

My next step would be to try to have more information on the stack trace and if I was lucky it could
tell me at least which kind of device I was having problem with.

I looked around my distro for KDE's debug symbols and couldn't find where or if they are available.
Luckily linux, _KDE_, _Qt_, etc. are open source, it would _only_ need to get to build it myself. :)

## Intermission: C and C++ build systems trauma

At this moment I was cold sweating and thinking where this would take me and if it wouldn't be better
to turn back and accept that one minute loading was my new reality until... the next update?

I was burnt out because just one week before I was trying to get a [GTK](https://www.gtk.org/) [vcpkg](https://github.com/Microsoft/vcpkg)
port up-to-date **on linux** and failed miserably. Previously _GTK_ was using _Makefiles_, but they 
migrated in recent versions to an even worse thing called _Meson_. Which basically made everything 
way more complicated than the normal C build system story already is.

Logically, if building _GTK_ was hard then building an entire desktop environment like _KDE_ would
be a lot harder right?

Luckily the people at _KDE_ (and _Qt_) decided to use _CMake_ as their build system and did a nice 
bootstrap setup. It is not perfect and other than a minor _perl_ hiccup, everything worked like a charm!

> If you're curious, check the _KDE_ [build instructions](https://community.kde.org/Get_Involved/development#One-time_setup:_your_development_environment).

Seriously, _CMake_ is not perfect but if you don't use it you're making things worse for **everyone**.

The build itself was long (I believe about ~1h as I didn't time it) to get `dolphin` and `kio` ready.

## Debug symbols, the solution?

With a freshly build _Dolphin_ with debug symbols ready, what can _GDB_ reveal?

```bash
➜  ~ source ~/kde/build/kio/prefix.sh
➜  ~ gdb ~/kde/usr/bin/dolphin
GNU gdb (GDB) 8.3.1

<...output...>

Reading symbols from /home/fungos/kde/usr/bin/dolphin...
(gdb) r
Starting program: /home/fungos/kde/usr/bin/dolphin 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
[New Thread 0x7fffef0c7700 (LWP 2812)]
[New Thread 0x7fffee51e700 (LWP 2813)]

<wait a bit and hit ctrl+c to get back at gdb>
^C
Thread 1 "dolphin" received signal SIGINT, Interrupt.
0x00007ffff4beac45 in pthread_cond_wait@@GLIBC_2.3.2 () from /usr/lib/libpthread.so.0
(gdb) bt
#0  0x00007ffff4beac45 in pthread_cond_wait@@GLIBC_2.3.2 () from /usr/lib/libpthread.so.0
#1  0x00007ffff562b610 in QWaitCondition::wait(QMutex*, QDeadlineTimer) () from /usr/lib/libQt5Core.so.5
#2  0x00007ffff562b702 in QWaitCondition::wait(QMutex*, unsigned long) () from /usr/lib/libQt5Core.so.5
#3  0x00007ffff68f25cd in ?? () from /usr/lib/libQt5DBus.so.5
#4  0x00007ffff68a2fa3 in ?? () from /usr/lib/libQt5DBus.so.5
#5  0x00007ffff68a37ce in ?? () from /usr/lib/libQt5DBus.so.5
#6  0x00007ffff68aee1e in ?? () from /usr/lib/libQt5DBus.so.5
#7  0x00007ffff68aef86 in QDBusInterface::QDBusInterface(QString const&, QString const&, QString const&, QDBusConnection const&, QObject*) () from /usr/lib/libQt5DBus.so.5
#8  0x00007ffff6c81468 in Solid::Backends::UPower::UPowerManager::UPowerManager (this=0x555555946340, parent=0x0) at /home/fungos/kde/src/solid/src/solid/devices/backends/upower/upowermanager.cpp:41
#9  0x00007ffff6c293f1 in Solid::ManagerBasePrivate::loadBackends (this=0x555555944060) at /home/fungos/kde/src/solid/src/solid/devices/managerbase.cpp:90
#10 0x00007ffff6c2d6d2 in Solid::DeviceManagerPrivate::DeviceManagerPrivate (this=0x555555944050) at /home/fungos/kde/src/solid/src/solid/devices/frontend/devicemanager.cpp:38
#11 0x00007ffff6c2ecaf in Solid::DeviceManagerStorage::ensureManagerCreated (this=0x7ffff6cd354c <(anonymous namespace)::Q_QGS_globalDeviceStorage::innerFunction()::holder>)
    at /home/fungos/kde/src/solid/src/solid/devices/frontend/devicemanager.cpp:301
#12 0x00007ffff6c2ec62 in Solid::DeviceManagerStorage::notifier (this=0x7ffff6cd354c <(anonymous namespace)::Q_QGS_globalDeviceStorage::innerFunction()::holder>)
    at /home/fungos/kde/src/solid/src/solid/devices/frontend/devicemanager.cpp:294
#13 0x00007ffff6c2e5da in Solid::DeviceNotifier::instance () at /home/fungos/kde/src/solid/src/solid/devices/frontend/devicemanager.cpp:182
#14 0x00007ffff6c2ae85 in Solid::Device::Device (this=0x555555697cb0, udi=...) at /home/fungos/kde/src/solid/src/solid/devices/frontend/device.cpp:59
#15 0x00007ffff7bece5e in KFilePlacesItem::KFilePlacesItem (this=0x555555697c80, manager=0x5555559dc5d0, address=..., udi=...) at /home/fungos/kde/src/kio/src/filewidgets/kfileplacesitem.cpp:48
#16 0x00007ffff7bf670d in KFilePlacesModel::Private::loadBookmarkList (this=0x5555556cbb00) at /home/fungos/kde/src/kio/src/filewidgets/kfileplacesmodel.cpp:764
#17 0x00007ffff7bf559d in KFilePlacesModel::Private::_k_reloadBookmarks (this=0x5555556cbb00) at /home/fungos/kde/src/kio/src/filewidgets/kfileplacesmodel.cpp:650
#18 0x00007ffff7bf443c in KFilePlacesModel::KFilePlacesModel (this=0x5555559f2070, alternativeApplicationName=..., parent=0x0) at /home/fungos/kde/src/kio/src/filewidgets/kfileplacesmodel.cpp:403
#19 0x00007ffff7ef7564 in DolphinPlacesModelSingleton::DolphinPlacesModelSingleton (this=0x7ffff7fcadb8 <DolphinPlacesModelSingleton::instance()::s_self>) at /home/fungos/kde/src/dolphin/src/dolphinplacesmodelsingleton.cpp:26
#20 0x00007ffff7ef75f5 in DolphinPlacesModelSingleton::instance () at /home/fungos/kde/src/dolphin/src/dolphinplacesmodelsingleton.cpp:33
#21 0x00007ffff7eea8fe in DolphinViewContainer::DolphinViewContainer (this=0x5555559e4540, url=..., parent=0x5555559e8310) at /home/fungos/kde/src/dolphin/src/dolphinviewcontainer.cpp:96
#22 0x00007ffff7ef9a92 in DolphinTabPage::createViewContainer (this=0x55555594c6a0, url=...) at /home/fungos/kde/src/dolphin/src/dolphintabpage.cpp:370
#23 0x00007ffff7ef85e2 in DolphinTabPage::DolphinTabPage (this=0x55555594c6a0, primaryUrl=..., secondaryUrl=..., parent=0x55555566ff60) at /home/fungos/kde/src/dolphin/src/dolphintabpage.cpp:43
#24 0x00007ffff7efb432 in DolphinTabWidget::openNewTab (this=0x55555566ff60, primaryUrl=..., secondaryUrl=..., tabPlacement=DolphinTabWidget::AfterLastTab) at /home/fungos/kde/src/dolphin/src/dolphintabwidget.cpp:171
#25 0x00007ffff7efb3b0 in DolphinTabWidget::openNewActivatedTab (this=0x55555566ff60, primaryUrl=..., secondaryUrl=...) at /home/fungos/kde/src/dolphin/src/dolphintabwidget.cpp:163
#26 0x00007ffff7efb7cc in DolphinTabWidget::openDirectories (this=0x55555566ff60, dirs=..., splitView=false) at /home/fungos/kde/src/dolphin/src/dolphintabwidget.cpp:215
#27 0x00007ffff7ec7c90 in DolphinMainWindow::openDirectories (this=0x555555631210, dirs=..., splitView=false) at /home/fungos/kde/src/dolphin/src/dolphinmainwindow.cpp:221
#28 0x00007ffff7ec4821 in kdemain (argc=1, argv=0x7fffffffdcf8) at /home/fungos/kde/src/dolphin/src/main.cpp:171
#29 0x000055555555517b in main (argc=1, argv=0x7fffffffdcf8) at /home/fungos/kde/build/dolphin/src/dolphin_dummy.cpp:3
(gdb) 
```

Now we have a nice stack. This reveals a lot of stuff we missed in the first try, mostly are uninsteresting details.
The real nice thing there is that now we know *which* device is being queried via _dbus_, which is..._UPower_?
Wait, what is happening here? **WHY** _Dolphin_ would try to query some _power_ related device white querying
up storage stuff?

Let's take a step back.
We never asked ourselves what is this _Solid_ thing. Searching about it leads to [Solid page](https://techbase.kde.org/Development/Tutorials/Solid/Introduction)
on _KDE TechBase_ wiki. That page explains that _Solid_ is a kind of _Hardware Discovery Layer_ and 
some features it provides is _Listing Devices_. In this page there is a tutorial on how your program
can use it, but mostly important, there is information about a `solid-hardware` tool to use on command line.

Lets try it:

```bash
➜  ~ solid-hardware list
Object::connect: No such signal QDBusAbstractInterface::DeviceAdded(QString)
Object::connect: No such signal QDBusAbstractInterface::DeviceRemoved(QString)
virtual QStringList Solid::Backends::UPower::UPowerManager::allDevices()  error:  "org.freedesktop.DBus.Error.TimedOut" 
udi = '/org/kde/solid/udev/sys/devices/LNXSYSTM:00/LNXPWRBN:00/input/input5'
<a lot of other devices>
```

BINGO?
Surely it had the exact same delay as _Dolphin_ before any output was shown and then as soon as it unblocked
I got the same output messages about unknown signals and a new interesting error about _UPower dbus_
timeout error.

Checking again the stack trace we see that _Solid_ device manager frontend will request to `loadBackends()`
in `ManagerBasePrivate`, which in turn [process all kind of backends](https://cgit.kde.org/solid.git/tree/src/solid/devices/managerbase.cpp?id=e33da0b273312877770d14ee9b6906acfacba8d0#n65) 
indiscriminately as it is agnostic about its clients intentions.

Now we know that:

1. _Dolphin_ needs to query storage devices using _Solid_;
2. _Solid_ loads all kind of backends independenty of _Dolphin_ real needs;
3. When a backend loads it will query _dbus_ in some or another way;
4. _UPower_ debus service is timing out;
5. _Solid_ blocks and in turn _Dolphin_ waits until the timeout fire to continue working;

So what is [UPower](https://upower.freedesktop.org/)? It seems to be an(other) abstraction layer to 
deal with hardware power management and it seems to be in part responsable to manage suspending and
waking up things.

So why does it timeout? `man upower` gives a few options, lets try some commands and see what we get:

```bash
➜  ~ upower -e

(upower:10725): UPower-WARNING **: 21:04:33.690: Cannot connect to upowerd: Error calling StartServiceByName for org.freedesktop.UPower: Timeout was reached

➜  ~ upower --monitor

(upower:6427): UPower-WARNING **: 22:22:36.230: Cannot connect to upowerd: Error calling StartServiceByName for org.freedesktop.UPower: Failed to activate service 'org.freedesktop.UPower': timed out (service_start_timeout=25000ms)
```

Sure. So _upowerd_ daemon is unreachable for some reason. If we ask _systemd_ to start it:

```bash
➜  ~ sudo systemctl start upower
Job for upower.service failed because the control process exited with error code.
See "systemctl status upower.service" and "journalctl -xe" for details.
```

`journalctl` doesn't say much useful, only that we got an `exit code 217`.
And this link to [Support](https://forum.manjaro.org/c/technical-issues-and-assistance).

But if we launch `upowerd` service ourselves, it works:

```bash
➜  ~ sudo /usr/lib/upowerd --verbose
[sudo] password for fungos: 
TI:21:05:00     Acquired inhibitor lock (7)
TI:21:05:00     Starting upowerd version 0.99.11
```

And magically _Dolphin_ becomes alive again and we can confirm with `solid-hardware list`.

Great, at least we have a workaround! And we know that the problem is actually _systemd_.
Now we have two questions to answer:

1. Why _systemd_ is unable to start it?
2. How to correctly fix this?

Going to support link from _systemd_ journal output and searching for _upower_ brings us to
this [question](https://forum.manjaro.org/t/upower-daemon-failed-to-start/108539). Which is basically
the issue I was having.

### Conclusion

A _systemd_ [change](https://github.com/systemd/systemd/blob/v243/README#L77) that makes use of a kernel 
feature unsupported on my kernel `4.9.202` broke my file manager.

Manjaro delivered a _systemd_ upgrade without **enforcing** a minimal kernel version. 
In Manjaro defense, they clearly stated this on their changelog, but we shouldn't expect people to 
read it.

This made me realize that I really need to update my kernel as it is 2 years old already.
And, that I have a lot less issues with linux than with Windows on my daily work, take that Microsoft.

Now an upgrade to `5.4` is required, see you soon...

### Epilogue (...5 minutes later)

... and it is done. After all this I'm up and running on `5.4` and most importantly, _Dolphin_ is 
alive.

