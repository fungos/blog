---
title: "Setting Up a Rust Environment on Windows"
date: 2017-08-12T13:00:00-05:00
draft: false
tags: ["rust", "windows", "vscode", "setup"]
categories: ["programming"]
toc: true
---

When I talk about Rust, one question that comes the most is about tooling and debugging. Online this can be seen a lot, mainly about debugging on Windows. 

So I've decided to write a little guide mostly to practice my written English and bootstrap this blog, but also to document the current steps needed to set a good Windows Rust environment.

Be warned that it may miss some details or be too superficial for some people.

The setup described here is the one I've been using Rust on Windows (and approximately equivalent on GNU/Linux) for some time and I believe it to be the ideal setup for myself, at least for the moment.

I also believe that this setup is already better than current C or C++ (take it with a grain of salt anyway), first because of Cargo, but also due the recent improvements with [Rust Language Server (RLS)](https://github.com/rust-lang-nursery/rls) and in part due to Rust use of LLVM and the compatibility with C ABI, it can piggyback on C/C++ tools like GDB, Valgrind, profilers and lots more. 

This makes setting up projects, developing and debugging a breeze if compared to C/C++, even if compared with Visual Studio and Visual Assist. Only Debugging is not as clean as debugging C code, its more like an equivalent of debugging a heavy STL C++ code base and some people may find this hard.

I hope this may be helpfull in some way to these getting started with Rust on Windows.

> Note: My background is with C and C++, so I tend to compare things with these languages and probably use terminology from this background.

## C++ Build Tools

The first and most important thing to a working Rust environment is to have one C++ toolchain available so Rust can build C or C++ ("-sys") crates and link against Windows libraries.

This can be achieved in two ways:

- Having a Visual Studio C++ installation, or;
- Installing the smaller [Visual C++ Build Tools](http://landinghub.visualstudio.com/visual-cpp-build-tools) for a standalone toolchain;


## Rustup

Rustup is the responsible for managing Rust compilers, Cargo, documentation, and so on. This tool is the entry point for a complete Rust environment and as Cargo, is a central part of Rust tooling.

> Note: To avoid any confusion with older Rust installations, if you'd already tried Rust before without Rustup, I suggest to uninstall everything (Rust, Cargo and anything related).

Installing Rustup is straightforward, download it [here](https://www.rustup.rs/) and then run it. A console window will launch as shown here:

![rustup-init.exe](/images/1/rustup-console.png)

Then go with the default installation (option 1). As shown in the screenshot, it will install a stable Rust for MSVC ABI.

Or by command line:
```dos
> rustup-init.exe -y
```


### Rust Toolchain

With Rustup installed, most of the toolchain management is done in the command line with the Rustup command. Getting used to it will surely help in the future.

First, lets update everything to be sure (required if it is not a fresh install):

```dos
> rustup update
```

Then, adding the nightly version of the compiler for trying experimental features and to be able to compile an useful tool (Clippy):

```dos
> rustup install nightly
```


### RLS and Components

RLS is essential and the main component, but it comes with its own requirements: the Rust source code, documentation and the analysis component.
The source and documentation will be used by analysis component to feed RLS with required data about existing crates and code. Read more about what RLS can do [here](http://www.ncameron.org/blog/what-the-rls-can-do/).

Basically, RLS enable auto-completion, documentation tooltips and related code editing features to any IDE supporting the Microsoft Language Server protocol. It is in a way equivalent to C++ Intellisense or Visual Assist. Although it is still under heavy development, I think it is already more usable than both of C++ alternatives on Visual Studio.

```dos
> rustup default nightly
> rustup component add rust-src
> rustup component add rust-docs
> rustup component add rust-analysis
> rustup component add rls
> rustup update
```

> Note: Both `rust-analysis` and `rls` can only be installed by using the nightly toolchain. Nightly can be set as the default toolchain or by passing the flag `--toolchain nightly` to the Rustup command line.

### Rustfmt

Rustfmt will automatically format the code accordingly to the current code style standard. Opposed to languages as C or C++ where each code base has its own style, Rust (and for the matter, most recent languages) enforce a standard style on the code. You may think it is too intrusive, but it is better for everyone by the end. More information about styling can be found on rustfmt [RFC repository](https://github.com/rust-lang-nursery/fmt-rfcs/blob/master/guide/guide.md).

To install it, suffice to use Cargo:

```dos
> cargo install rustfmt
```

It will be used by VSCode Rust Extension to automatically format Rust code being edited. 

> Note: Rustfmt will be asked to be installed by Visual Studio Code if it isn't already installed anyway.

But rustfmt is in active development and the recommended way is by using the nightly version:

```dos
> rustup run nightly cargo install rustfmt-nightly
```

Now, formating code should be done by issuing the command bellow, indicating it uses the nightly toolchain otherwise and error loading its dependencies will happen:

```dos
> cargo +nightly fmt
```


### Clippy

Clippy is another jewel of Rust, it is the second best friend (after Rust compiler) on helping learning Rust and improving the code. It is a linter that will catch issues and suggest improvements, it should be used regularly and will help learning some tricks and writing idiomatic code. The downside if we can say so is that it requires a nightly compiler to install/compile it.

Again, to install, use Cargo (and be sure to have the nightly toolchain in use):

```dos
> rustup run nightly cargo install clippy
```

> Note: Installing these tools with cargo means compiling them, and Rust compile times are not really the best. So, go take a coffee.

> Note: Clippy will be asked to be installed by Visual Studio Code if it isn't already installed anyway.

To check a Rust project using Clippy:

```dos
> cargo clippy
```

## Visual Studio Code

I've tried some different IDEs (take a look [here](https://areweideyet.com/) for some alternatives and their current support state), but I believe VSCode (at the moment) is the best and most complete mostly due to the [Language Server Protocol](https://github.com/Microsoft/language-server-protocol)/RLS. 

> Note: The second best is [IntelliJ Rust](https://intellij-rust.github.io/) and now it is [officially supported by JetBrains](https://blog.jetbrains.com/blog/2017/08/04/official-support-for-open-source-rust-plugin-for-intellij-idea-clion-and-other-jetbrains-ides/). The problem is that it is heavy. I wouldn't recommend it unless you're already have it. Also, it does not support debugging, but this is being worked on.

Before configuring a Rust project in Visual Studio Code, some extensions are needed as these will help to edit code and debugging, and they're pretty much all it is needed to have a working Rust IDE.


### Extensions

All VSCode extensions here may be installed via the Extensions Panel, or by using `CTRL+P` and typing `ext install <name>` or by command line:

```dos
> code --install-extension <name>
```


#### Rust (rls)

[Rust (rls)](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust) is a new extension (the third Rust extension already) but is already very usable, also this is the official one by the Rust Tool Team and is being worked on actively.

Extension name: `rust-lang.rust`

```dos
> code --install-extension rust-lang.rust
```


#### C/C++

This extension is needed to be able to debug native binaries using GDB, LLDB or Visual Studio Debugger.

Extension name: `ms-vscode.cpptools`

```dos
> code --install-extension ms-vscode.cpptools
```

It will also download and install any required dependency.

#### Native Debug

This one is not really needed, but I recommend use this to automatically enable VSCode to set breakpoints on Rust code. Otherwise, it would require tuning user settings (`CTRL+,`) and setting `debug.allowBreakpointsEverywhere`.

Extension name: `webfreak.debug`

```dos
> code --install-extension webfreak.debug
```


### Visualizers

> Note: This section will soon be deprecated. A [recent commit](https://github.com/rust-lang/rust/pull/43221) on Rust, is now embedding visualization information inside PDBs. This is already valid for Nightly toolchain, but Stable still requires this manual step.

With Rust MSVC toolchain comes some type visualizers for Visual Studio Debugger (Natvis), these can be used to ease the visualization of some very few Rust types within the Debugger. Sadly, there are still limitations of what can be specified in a Natvis file.

The downside is that the installation of the visualization files are a bit manual at the moment, so this is the hardest part of all the Rust environment setup.

The files in question are in `%HOMEPATH%\.rustup\toolchains\stable-x86_64-pc-windows-msvc\lib\rustlib\etc` and must be copied to `%HOMEPATH%\.vscode\extensions\ms-vscode.cpptools-0.12.2\debugAdapters\vsdbg\bin\Visualizers` (note the version on ms-vscode.cpptools path, this can be different for your case).

If a custom visualizer is needed, an entry `"visualizerFile"` can be set inside `launch.json` (environment variables can be referenced as `${env:VAR}`), so for example, one can set: `"visualizerFile": "${env:HOMEPATH}/.rustup/toolchains/stable-x86_64-pc-windows-msvc/lib/rustlib/etc/libcore.natvis"`. For the moment, only one file can be used, an [issue](https://github.com/Microsoft/vscode-cpptools/issues/925) is open to make it work with multiple files.


## Debugging

Now that everything is in place, the last thing is configuring a Rust project to be debugged.

Create a debug configuration by going to the `(1) Debug Panel`, then clicking on the `(2) Debug Configuration Dropdown` and then select `(3) C++ (Windows)` option as shown below:

![debug](/images/1/vscode-debug-cfg.png#center)

![debug](/images/1/vscode-debug-msvc.png#center)

This will open a new file in the editor, called `launch.json` where all launch settings are managed.

The initial contents of the file aren't good because VSCode does not know details about the loaded Rust project (and I don't believe extensions can do anything here to help with it, at least for the current version), so a little tweak is required.

The current file should look as this:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(Windows) Launch",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "enter program name, for example ${workspaceRoot}/a.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "environment": [],
            "externalConsole": true
        }
    ]
}
```

Some important things to note here are the `type` and `program`.

The `type` should specify `cppvsdbg` to use the MSVC Debugger and have Natvis support.

The `program` here is the path to the executable generated by rustc/Cargo. When using Cargo, normally the debug executable will be found in a path as `<project>/target/debug/<project>.exe`. With this in mind, a good default should be to point the `program` to `${workspaceRoot}/target/debug/${workspaceRootFolderName}.exe`.

My preferred `launch.json` is this:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceRoot}/target/debug/${workspaceRootFolderName}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "environment": [],
            "externalConsole": true
        },
        {
            "name": "Release",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceRoot}/target/release/${workspaceRootFolderName}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "environment": [],
            "externalConsole": true
        }
    ]
}
```

With this, a way to debug release binaries is already in hand if needed for some reason.

## Wishlist

The missing key step here would be to have a better integration from Rust Language Extension to manipulate/fix/generate `launch.json` for Rust projects or even better, a `cargo ide-setup <ide>` command or something similar, where `<ide>` could be `vscode`.

This command would then be able to install the required vscode extensions if needed, and copy the natvis files to the right place, if in a windows environment.

### `rust-env-setup` Script

This is a quick power shell script I hacked together in a gist for automatizing a working Rust environment with VSCode.

> Note: I'm not a PS guy, so do not take this for a solution.

{{< gist fungos 5c94a2b7ed4a5b7ea4572e54c0c96a34 "rust-env-setup.ps1" >}}

## Feedback

Help me improving this text by submitting fixes for wrong wording/phrasing or typos in [this repo issue tracker](https://github.com/fungos/fungos.github.io/issues) or by commenting on twitter or elsewhere!

## Updates

- Fixed script to consider the fresh VSCode 64bits. Thanks @colemickens (Reddit).
- Fixed missing analysis component and rustfmt-nightly on text and script. Thanks @dodheim (Reddit).
- Added more information about how to add a natvis file into `launch.json` and mention that soon it will not be required, as it will be automatically embedded onto PDB files.
- Add information and fix typo. Closes [issue #1](https://github.com/fungos/fungos.github.io/issues/1) by @TheCycoONE (Github).