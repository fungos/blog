+++
title = "cr.h: A Simple C Hot Reload Header-only Library"
date = 2017-11-20
draft = false
aliases = ["/blog/2017/11/20/cr.h-a-simple-c-hot-reload-header-only-library/"]

[taxonomies]
tags = ["c/c++", "live", "reload"]
categories = ["programming", "gamedev"]

[extra]
toc = true
+++

Recently I've been back to hobby coding simple C stuff, and one project that I'm doing with a friend tries to simple emulate some old game. The idea is really basic, but we want to do it in the C-style without over engineering or losing track of the hobby feeling.

But! It is really hard to not care at least a bit, even if it is just hobby stuff. I got literally side-tracked at one point and here I describe why and the resulting product of this.

![ImGui with cr.h](https://camo.githubusercontent.com/08a269b19513df8325a177ff96e4e387f36b5bbf/68747470733a2f2f692e696d6775722e636f6d2f4e7136733047502e676966)

## The "Problem"

While prototyping a functionality in this project, I wanted to be able to quickly iterate trying some ideas. My first reaction was that having a scripting language from the start would be a benefit for testing out ideas in a fast-paced way, but at the same time that would easily get in the way (by requiring bindings and maintenance) considering the time available for some hobby coding. On the other side, something that wouldn't get in the way would be doing it in a way as close as possible of the C and with less boilerplate possible, helping us keep the focus on the project itself.

If what we wanted is something close to C, it could be done in at least two ways:

- a script based on a C subset (preferable compilable to C); or,
- run-time hot-reload for live coding;

<br/>
Independent of which way we decided to go with, I had already some expectations about it, I wanted something that respected most of the following requirements, loosely based on my priorities for a hobby project:

- easy to use;
- requires the minimum amount of boilerplate;
- easy to maintain;
- fast;
- independent of build system and not requiring custom build steps or anything;
- closest possible to the C-family syntax (ie. Lua was a big NO);

## The "Options"

I've checked some scripting languages and none of them fit most of the listed requirements. The better ones are slow or complex and the faster ones (as Lua) have annoying syntax. I mostly based my evaluation on [this][0] very nice listing with some benchmarks and code samples.

In the other hand, creating a kind of C script is not that simple, it requires much more code to achieve something usable than integrating some ready-to-use scripting language. There is a bunch of libraries and tools that can help, like embedding a simple compiler as [TCC][1] or one complex as [libclang/libtooling][2], or even maybe something that already embedded the compiler for us, as [C-Toy][3] or [CINT][4]. But that adds a lot of dependency code, requires too much fiddling with build systems, still requires writing bindings and aren't really _easy to use_.

The other option would be doing hot-reloading of the runtime code as we change it. Even if this may appear complex, it is at least not as complex as to write a simple language. One downside is that opposed to scripts, this area does not have much public content in both articles and source code forms. Luckily enough, this idea fits with my concept of hobby stuff and is doable in my free time.

### Hot Reloading

One very known solution is [RuntimeCompiledCPlusPlus (RCC++)][5], other than that, there is nothing else **ready-to-use** even if this is a somewhat common practice privately. So first, lets thanks [Doug Binks][6] and [enkisoftware][12] for publishing RCC++ with an open source license, this is a much required improvement over the situation.

RCC++ is a full featured solution, and this comes with its own amount of complexity. On my case, I didn't need all features it offers, but I strongly recommend evaluating it when looking for a solution, as each one has its pros and cons. To know more about its design and usage, I recommend reading [this article][13].

Another good thing about RCC++ is that it has listing of some [alternatives][7] solutions on code hot-reloading, including some nice posts by people that use it for actual development like [this post][8] from [Our Machinery][9]. Sadly, none of the projects with source code seems ready to use, as they look more like experimentation projects and most of them if not all, don't have multi platform support or are simple barely usable at all.

So, I decided to write one that I hope to be simple but also usable by anyone:

## cr.h: the c reloader

Considering the requirements for our hobby project listed before, in a public and open source project these requirements would become:

- simple to use, but not basic;
- less intrusive possible;
- reusable to anyone, not only specific to my needs;
- avoid build system customization or dependency;
- linux and windows at least (macosx is a bonus);

<br/>
The first four points are related to user experience and the last one is a minimum expected from any meaningful public project.

Being simple and reusable comes with not being too intrusive and having a simple public API. If anyone other than me decide to use it, it should not require learning a lot of details of how it works nor requiring deep changes in existing code. But also, not requiring complex changes to existing building system or scripts to do some magic in the background.

### Overview and Usage

Before implementing `cr.h`, I read everything I could find about how people deal with this and what the most frequent problems and issues. I will try to explain how my implementation differ from others and how I've solved some of the more common issues.

The core of the system is really basic and do not differ from most of the home grown solutions. The idea is to split the code into a thin host application executable and the core of the program into a dynamically loadable binary (shared object or dll) guest.

The less the host needs to know the better and easier it becomes. Ideally it should just be able to load the binary, monitor for new updates, unload the current one saving any required state then loading the new up-to-date binary and passing over the saved state, repeating the process until terminated by the user.

The usage is really simple, the very first thing is to initialize the system with `cr_plugin_init`, a `cr_plugin` context and the fullpath to the loadable object (ie. a .so or .dll). Once initialized, the main function `cr_plugin_update` must be frequently called, as it will call the real application and it will deal with all the reloading and monitoring stuff. Finally, when the application wants to exit, `cr_plugin_close` will do all the required cleanup. This is all the public API when using `cr.h`.

The `cr_plugin` context contains some internal private stuff, but also some information useful to the application itself. One is the `version` field, a value incremented each time a reload is successful or decremented in case of a rollback. Rollbacks may happen when a crash or an issue is detected, the system will try to safely unload the problematic binary and reload a previous working one. In case of rollback, a `failure` code will be set in the plugin context and the new loaded binary may use this information to give some useful feedback or dealing with it in an appropriate fashion for the application.

Once up and running, each time the loadable binary is rebuilt, `cr.h` will trigger a reload as it is monitoring for file changes based on the file time stamp. Each time an update, a load or an unload happens, `cr.h` will pass the info down to the application by using the `cr_op` operation flag: `CR_LOAD`, `CR_STEP` or `CR_UNLOAD`. For example, in case of unload the application may be able to intercept and deal with something before the binary is fully unloaded (like saving some internal state).

This is everything needed to live code reload using `cr.h`!

However, how does `cr.h` deal with the issues cited over the other articles about hot-reloading? How to manage state between reloads? What about the common PDB locking that people frequently have on windows? `cr.h` tries to solve these problems without any workaround or tricks with build systems.

#### Problem: PDB Lock

One recurring theme when doing hot-reloading on windows with a MSVC toolchain is the PDB lock while debugging. One instance of this issue can be seen [here][8] and [here][14]. These posts lists some possible solutions and problems with each solution, and it goes way down to trying to force unlock the file handles on windows as seen [here][15].

`cr.h` solves this in a pragmatic and simple way, first it starts by copying the `.dll` and the `.pdb` with a new versioned name and then we fix the real issue, that is not the file lock.

When debugging, the debugger needs a way to find for the debug symbols, and in the MSVC toolchain this comes in the form of a second file, the `.pdb` that may exist elsewhere in the filesystem. So when generating the executable it will write the path to the matching PDB file inside the binary to be debugged. It literally contains the full path to the `.pdb` hardcoded inside it and the debugger will load this file causing a lock. You can check this yourself by doing a `strings` on the `.dll` or opening it with a hex editor and searching for the substring `.pdb`.

The only thing needed to do to avoid the lock to happen, is to literally change this string to point to our own copy of `.pdb` and thus guaranteeing that the debugger will lock our copy and not the original file while debugging.

This can be done the brute-force but brittle way (searching and replacing the string), or the right way by doing what the debugger does: correctly opening and parsing the binary structures and modifying it. There are a lot of documentation on the [Portable Executable format][16] and how to parse it, or more specifically on how to find the [debugging information][17] as reference for more details on the subject.

With this simple solution we can debug and rebuild at the same time even during reloads, the debugger will find the correct debug info and be up-to-date with the current debugged binary as we modify it.

#### Problem: Crashes

While live coding, the chances to introduce problems are high as we get into a faster development flow. So if we can avoid crashing due to erroneous code, it will help keeping with this faster flow. Hence, `cr.h` tries to be helpful and to detect crashes and safely continuing the execution from a previous working version. This system is not unbreakable as at the end we are dealing with C/C++ and there are just too many ways to shoot yourself.

In practice, `cr.h` tries to emulate the debugger here too. On windows it will use [structured exception handling][18] to detect some common problems as illegal instruction, access violation and some others. In which case, `cr.h` will catch it and try to unload the problematic binary and revert back to the previous working one, effectively doing a rollback.
Over Linux, the same happens but it is managed using the OS signal handlers.

All this enables a seamsly development flow that is pleasing to use.

#### Problem: State Transfer

A very common way to keep state between reload is to use the heap and pass pointers to objects so the host can hand it over the next reload. This requires that the plugin instances share the same allocator, it may be managed by the host or via a common crt (dynamic crt on MSVC). One limitation of this approach is with global and local static states.

For the first case, using the heap model the user may decide to manage its own states by filling a struct with pointers to objects and handing it over so `cr.h` can hold it between reloads using the `userdata` pointer in the `cr_plugin` context. Other than the same allocator being required, care should be taken with destructors called during unload.

The second case is when dealing with static state (both global and local), it would be really annoying and highly error prone to do it by hand like copying over to the heap and restoring (and dealing with a lot of issues this may cause). In this situation, `cr.h` will magically do all the hard work with all necessary static data tagged with a macro `CR_STATE`. These states will get saved and copied over to the newly loaded instance and everything will just work.. most of the time. The catch here is that depending on how much your binary changes, things may not work as we're dealing with opaque memory copying and addresses that may change.

Here some things to be aware when using `CR_STATE`:

- Do not save objects that have pointers to anything that is not in the heap;
- Do not save objects that have non trivial constructors and destructors, they may or may not work;

<br/>
All this is subject to change as I'll be hardening it while using in my projects. I have some more ideas in the back of my mind on how to improve all this by using more debug info, but not sure if it is worth the effort. Enough yak shaving.

### Development Stats

Finally, some approximated development time statistics (as my free time is mostly in spans of 30 minutes to 1h).

- Base implementation: 1h
    - Windows specific: 3h
    - Linux specific: 4h
- Samples: 3h
- Tests: 2h
- Documentation: 2h
- This Post: 7h

<br/>
Total: 22h (5 weeks, ~4h/week)

### Feedback

Please, post corrections and suggestions about this post by opening an issue [here](https://github.com/fungos/fungos.github.io/issues/). Any help/improvement is appreciated.

[0]: https://github.com/r-lyeh/scriptorium
[1]: https://bellard.org/tcc/
[2]: https://clang.llvm.org/docs/Tooling.html
[3]: https://github.com/anael-seghezzi/CToy
[4]: https://root.cern.ch/cint
[5]: https://github.com/RuntimeCompiledCPlusPlus/RuntimeCompiledCPlusPlus
[6]: https://www.enkisoftware.com/about#doug
[7]: https://github.com/RuntimeCompiledCPlusPlus/RuntimeCompiledCPlusPlus/wiki/Alternatives
[8]: http://ourmachinery.com/post/dll-hot-reloading-in-theory-and-practice/
[9]: http://ourmachinery.com/
[12]: https://www.enkisoftware.com
[13]: http://runtimecompiledcplusplus.blogspot.ca/2016/04/runtime-compiled-c-article-available.html
[14]: http://ourmachinery.com/post/little-machines-working-together-part-2/
[15]: https://blog.molecular-matters.com/2017/05/09/deleting-pdb-files-locked-by-visual-studio/
[16]: https://msdn.microsoft.com/en-us/library/ms809762.aspx
[17]: http://www.debuginfo.com/articles/debuginfomatch.html
[18]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx
