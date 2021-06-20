---
layout: post
title: "How to Compile and Load onto a Teensy"
date: 2021-06-20
categories: microcontrollers, rust
---

To me, one of the most interesting things about embedded devices is the breadth of hardware available. Unfortunately, the variaty of hardware makes for a less ergonomic experience. One set of devices that caught my attention were the [Teensy](https://www.pjrc.com/teensy/) family of microcontrollers. In my small amount of embedded experience, they're quite powerful, relatively inexpensive, easy to procure, and small (makes sense given the name is "Teensy"). Teensy microcontrollers are Arduinio compatible, so you can use the Arduino IDE and libraries with them, but given my general interest in Rust, I was curious how to load custom, non Arduino programs onto one. The process is actually quite easy, especially given that Teensy has both a [GUI loader](https://www.pjrc.com/teensy/loader.html) and a [CLI tool](https://www.pjrc.com/teensy/loader_cli.html). This post will walkthrough the necessary tools to build and load a Rust program on Mac for a Teensy 4, but should be general enough for any language, give you know how to compile to the right processor.

The strategy for loading a program to the Teensy is: compile program, convert program to Intel hex format, load file onto device using CLI (or GUI). Compiling for a device is easy, loading the file should also be easy, but what is [Intel hex](https://en.wikipedia.org/wiki/Intel_HEX), why are we converting to it, and how do we do it? Intel hex (or HEX for short) is just an ASCII representation of binary, orginally developed for Intel. I just think of it as "more readable" version  of machine code, which I'm guessing is somewhat of a relic of a time ago, when people wrote programs in assembly and such. We need to convert to HEX as the loader requires it. Now, I'm not too sure why the loader requires HEX instead of just the compiled binary, but this is just how it is.

So we now have some understanding of what HEX is, now how do we get it? Thankfully, it's actually quite easy! There's a GNU/LLVM utility called `objcopy`, which makes a copy of a file, with the ability to change the format of that file. Threrefore, we can give `objcopy` a binary file and it can create a copy of that file in HEX format. Before we can use it, we need to install it. I chose to go with the LLVM version, which requires us to just download LLVM. Given that I'm on Mac, I used Homebrew for this process. After install, you just need to add the path to your Bash/Zsh rc file (and then `source` it).

```bash
$ brew install llvm
$ echo 'export PATH="/usr/local/opt/llvm/bin:$PATH"' >> ~/.zshrc
$ source ~/.zshrc
```

Now you have `llvm-objcopy` installed and in your path, ready to use!

The next step is to install the loader, which is as easy as cloning the repo and then building it.

```bash
$ git clone https://github.com/PaulStoffregen/teensy_loader_cli.git
$ cd teensy_loader_cli
```

Now one "gotcha" that I ran into was not uncommenting the correct platform in the makefile (but this is probably more a me problem than anything else). The issue I ran into was building the loader for Linux (instead of Mac) and therefore was running into `Unable to claim interface, check USB permissions`, but this was easy enough to fix. 

```makefile
# Makefile

# OS ?= LINUX
#OS ?= WINDOWS
OS ?= MACOSX
#OS ?= BSD
```

Now you're ready to build the loader and then move it into a visible path.

```bash
$ make
$ mv teensy_loader_cli /usr/local/bin
```

See, easy enough!

Now we just need a program to load. I used the [`teensy4-rs-template`](https://github.com/mciantyre/teensy4-rs-template), which contains a simple program that just blinks the onboard LED in one second intervals. Note: You may need to install `cargo genearte` with `cargo install cargo-generate`.

```bash
$ cargo generate --git https://github.com/mciantyre/teensy4-rs-template --name teensy-4-hello-world
```

Now all you need to do is build it. If you haven't already, you will need to install the correct target platform, which in this case is ARM Cortex-M7. The Rust ecosystem makes this easy as you can  just run `rustup target add thumbv7em-none-eabihf`. You can see all the available Rust targets [here](https://doc.rust-lang.org/nightly/rustc/platform-support.html). Now we can build and load the program!

```bash
$ cargo build --release
$ llvm-objcopy --output-target=ihex target/thumbv7em-none-eabihf/release/teensy-4-hello-world teensy-4-hello-world.hex
$ teensy_loader_cli --mcu=TEENSY40 -v -w teensy-4-hello-world.hex
```

As you can see, we can specify a variaty of formats for [`llvm-objcopy`](https://llvm.org/docs/CommandGuide/llvm-objcopy.html), therefore we specify `--output-target=ihex`. Similarly, `teensy_loader_cli` is used for any of the Teensy microcontrollers, therefore we specify that we're loading to a Teensy 4. You'll probably need to press the reset button on the microcontroller once you run the above commands, but that's it! What's great is that most of this is not specific to Rust; if you know how to compile for the correct platform, you now have the tools to load it onto a Teensy!
 
