+++
author = ""
comments = true
date = "2016-10-06T10:12:22+02:00"
description = "This post will explain how to make terminal applications in Rust for new beginners."
draft = false
featured = false
image = ""
menu = ""
share = true
slug = "making-terminal-applications-in-rust-with-termion"
tags = ["rust", "termion", "tui"]
title = "Making Terminal Applications in Rust with Termion"
+++

This post will walk through the basics of implementing a terminal (TTY) application for both new beginners and experienced users of Rust.

# Introduction

Terminal applications play an important role in many programmers' toolchain, from text editors to minigames while your code is compiling. And it's great to know and understand how to make these yourself, so you can create a customized TUI application for your needs.

Escape codes and TTY I/O is messy, but fortunately there are libraries for this. We will use [Termion](https://github.com/ticki/termion), which is the most feature-complete TUI library in pure Rust.

Termion is pretty simple and straight-forward. This "tutorial" or guide is going to walk through these in a manner that even Rust new beginners can understand.

# Understanding the TTY

Ignoring historical facts, the TTY is the name of the virtual device that takes some stream of text and presents it to the user. As opposed to sophisticated UIs and graphics, it is incredibly simple to get started with.

The terminal emulator keeps a grid of characters, and a cursor. When you write to the standard output the cell is overwritten with the new character and the cursor moves respectively.

Take the code,

```rust
println!("Text here.");
```

All this does is writing some text to the standard output, and when you run this program, "Text here." should appear before the TTY cursor.

If this is all we can do, how can we create interactive TTY applications? Well, it turns out that there is a whole lot more, we can do.

Certain sequences represents some operations to the TTY. These are called "escape sequences" and can do things like changing the color of the text, change the background, moving the cursor, clearing the screen, and so on. Writing these codes by hand quickly gets messy, so we let Termion do it for us:

# Setting up Termion

Start by making sure `cargo` is installed, then do

    # Initialize a new cargo repository.
    cargo new --bin my-tui-app
    # Cd into it
    cd my-tui-app

Then open the `Cargo.toml` file with your favorite text editor, and add

    termion = "1"

To the file under the section `[dependencies]`.

Then open up `src/lib.rs` and add

```rust
extern crate termion;
```

Now everything is ready to start!

> For documentation, see [here](https://github.com/ticki/termion).

# The structure of Termion

Termion is divided into 8 different modules each providing different functions and primitives:

1. `clear`: For clearing the screen or parts of the screen.
2. `color`: For changing the foreground or background color of the text.
3. `cursor`: For moving the cursor around.
4. `event`: For handling mouse cursor or modifiers.
5. `input`: For getting more advanced user input (like asynchronous user input).
6. `raw`: Switching to raw mode (we will get back to this later)
7. `scroll`: Scrolling up or down the text stream.
8. `style`: Changing the text style or formatting.

## Color

Since escapes really are nothing but just another text output, we use the `std::fmt::Display` to generate the escape codes. This means that we can use it with macros like `write!` or `println!`. If we want red text for example, we can do simply:

```rust
extern crate termion;

// Import the color module.
use termion::color;

fn main() {
    println!("{red}more red than any comrade{reset}",
             red   = color::Fg(color::Red),
             reset = color::Fg(color::Reset));
}
```

`color::Fg` specifies that we want to change the _foreground color_ (i.e. the color of the text), `color::Fg(color::Reset)` means that we _reset_ the foreground color.

## Clear

Clearing the screen allows you to remove text which is already written without overwriting it manually with spaces. For example, I can easily implement the `clear` command:

```rust
extern crate termion;

// Import the `clear` module.
use termion::clear;

fn main() {
    println!("{}", clear::All);
}
```

It should be pretty obvious that `clear::All` clears the whole grid, but what if we only want to clear the screen partially?

- `clear::CurrentLine` will leave the current line empty.
- `clear::AfterCursor` clears from the cursor to the end of the grid.
- `clear::BeforeCursor` clears from the cursor to the beginning of the grid.

[and so on...](https://docs.rs/termion/1.1.1/termion/clear/index.html)

## Cursor

What if I want to jump back and overwrite what I just wrote? The easy way is to use `\r`, which will jump back to the start of the line:

```rust
extern crate termion;

use termion::{color, clear};
use std::time::Duration;
use std::thread;

fn main() {
    println!("{red}more red than any comrade{reset}",
             red   = color::Fg(color::Red),
             reset = color::Fg(color::Reset));
    // Sleep for a short period of time.
    thread::sleep(Duration::from_millis(300));
    // Go back;
    println!("\r");
    // Clear the line and print some new stuff
    print!("{clear}{red}g{blue}a{green}y{red} space communism{reset}",
            clear = clear::CurrentLine,
            red   = color::Fg(color::Red),
            blue  = color::Fg(color::Blue),
            green = color::Fg(color::Green),
            reset = color::Fg(color::Reset));
}
```
But actually, `\r` is pretty limited, because it only allows us to jump to the start of the line. What if we want to jump to an arbitrary cell in the text grid?

Well, we can do that with `cursor::Goto`, say we want to print the text at (4,2):

```rust
extern crate termion;

use termion::{color, cursor, clear};

fn main() {
    println!("{clear}{goto}{red}more red than any comrade{reset}",
             // Full screen clear.
             clear = clear::All,
             // Goto the cell.
             goto  = cursor::Goto(4, 2),
             red   = color::Fg(color::Red),
             reset = color::Fg(color::Reset));
}
```

## Style

What if I want my gay space communism to have style?

The `style` module provides escape codes for that. For example, let's print it in bold (`style::Bold`):

```rust
extern crate termion;

use termion::{color, clear, style};

fn main() {
    println!("{bold}{red}g{blue}a{green}y{red} space communism{reset}",
            bold  = style::Bold,
            red   = color::Fg(color::Red),
            blue  = color::Fg(color::Blue),
            green = color::Fg(color::Green),
            reset = style::Reset);
}
```

Neat. Now we can control the cursor, clear stuff, set color, and set style. That should be good enough to get us started.

# Entering raw mode

Without raw mode, you cannot write a proper interactive TTY application. Raw mode gives you complete control over the TTY:

1. It disables the line buffering: As you might notice, your command-line application tends to behave like the command-line. The programs will first get the input when the user types `\n`. Raw mode makes the program get the input after every key stroke.
2. It disables displaying the input: Without raw mode, the things you type appear on the screen, making it insufficient for most interactive TTY applications, where keys can represent controls and not textual input.
3. It disables canonicalization of the output: For example, `\n` represents "go one cell down" not "break the line", for line breaks `\n\r` is needed.
4. It disables scrolling.

So, how do we enter raw mode?

It's not that hard:

```rust
use termion::raw::IntoRawMode;
use std::io::{Write, stdout};

fn main() {
    // Enter raw mode.
    let mut stdout = stdout().into_raw_mode().unwrap();

    // Write to stdout (note that we don't use `println!`)
    writeln!(stdout, "Hey there.").unwrap();

    // Here the destructor is automatically called, and the terminal state is restored.
}
```

# Inputs

Keys and modifiers are somewhat oddly encoded in the ANSI standards, and fortunately Termion parses those for you. If you take a look at the [`TermRead`](https://docs.rs/termion/1.1.1/termion/input/trait.TermRead.html) trait, you'll see the method called `keys`. This returns an iterator over [`Key`](https://docs.rs/termion/1.1.1/termion/event/enum.Key.html), an enum which contains the parsed keys.

```rust
extern crate termion;

use termion::event::Key;
use termion::input::TermRead;
use termion::raw::IntoRawMode;
use std::io::{Write, stdout, stdin};

fn main() {
    // Get the standard input stream.
    let stdin = stdin();
    // Get the standard output stream and go to raw mode.
    let mut stdout = stdout().into_raw_mode().unwrap();

    write!(stdout, "{}{}q to exit. Type stuff, use alt, and so on.{}",
           // Clear the screen.
           termion::clear::All,
           // Goto (1,1).
           termion::cursor::Goto(1, 1),
           // Hide the cursor.
           termion::cursor::Hide).unwrap();
    // Flush stdout (i.e. make the output appear).
    stdout.flush().unwrap();

    for c in stdin.keys() {
        // Clear the current line.
        write!(stdout, "{}{}", termion::cursor::Goto(1, 1), termion::clear::CurrentLine).unwrap();

        // Print the key we type...
        match c.unwrap() {
            // Exit.
            Key::Char('q') => break,
            Key::Char(c)   => println!("{}", c),
            Key::Alt(c)    => println!("Alt-{}", c),
            Key::Ctrl(c)   => println!("Ctrl-{}", c),
            Key::Left      => println!("<left>"),
            Key::Right     => println!("<right>"),
            Key::Up        => println!("<up>"),
            Key::Down      => println!("<down>"),
            _              => println!("Other"),
        }

        // Flush again.
        stdout.flush().unwrap();
    }

    // Show the cursor again before we exit.
    write!(stdout, "{}", termion::cursor::Show).unwrap();
}
```

What the above snippet does is to open a blank screen, where it informs you what keys and modifiers you type as you press keys.

# Asynchronized stdin

One interesting problem you will run into, while writing certain terminal application is that the stdin is blocking, and you need to wait to the user giving the input. This potentially could block your application from doing work while waiting for user input (e.g. you freeze the graphics).

Fortunately, Termion has a solution to that [`termion::async_stdin()`](https://docs.rs/termion/1.1.1/termion/fn.async_stdin.html). In principle, it is really simple. It works around the limitation to TTYs by using another thread to read from the stdin, and when your main thread needs to read from the stream, it pops from a concurrent queue to read the bytes. It doesn't scale to things like byte streams, but it works seamlessly with user input.

# Mouse

You can read mouse clicks etc. by converting your stdin stream to [`termion::input::MouseTerminal`](https://docs.rs/termion/1.1.1/termion/input/struct.MouseTerminal.html):

```rust
extern crate termion;

use termion::event::*;
use termion::cursor;
use termion::input::{TermRead, MouseTerminal};
use termion::raw::IntoRawMode;
use std::io::{self, Write};

fn main() {
    let stdin = io::stdin();
    let mut stdout = MouseTerminal::from(io::stdout().into_raw_mode().unwrap());
    // ...
```

Then we can clear the screen:

```rust
    writeln!(stdout,
             "{}{}q to exit. Type stuff, use alt, click around...",
             termion::clear::All,
             termion::cursor::Goto(1, 1))
        .unwrap();
```

Then you can read mouse inputs through the `events()` function:

```rust
    for c in stdin.events() {
        let evt = c.unwrap();
        match evt {
            Event::Key(Key::Char('q')) => break,
            Event::Mouse(me) => {
                match me {
                    MouseEvent::Press(_, a, b) |
                    MouseEvent::Release(a, b) |
                    MouseEvent::Hold(a, b) => {
                        write!(stdout, "{}", cursor::Goto(a, b)).unwrap();
                    }
                }
            }
            _ => {}
        }
        stdout.flush().unwrap();
    }
}
```

Now, if you click around or hold your your mouse, the TTY cursor should follow.

# A few extra tricks

## The terminal size

Sometimes you might want to center or align things. This need the terminal size, which can be obtained by [`termion::terminal_size()`](https://docs.rs/termion/1.1.1/termion/fn.terminal_size.html).

## Bypassing piped input

Sometimes you might want to pipe some input to your program while controling the TTY. This is actually not that hard. With [`termion::get_tty()`](https://docs.rs/termion/1.1.1/termion/fn.get_tty.html), you can read and write from the TTY, while still being able to read or write to stdin/stdout via `std::io`.

## Truecolor

[`termion::color::Rgb(r, g, b)`](https://docs.rs/termion/1.1.1/termion/color/struct.Rgb.html) allows you to use full 24-bit truecolor.

# Trying all this out yourself

There's a lot of things you can do as well:

1. Writing a simple nano clone.
2. Writing a TUI music player.
3. Writing a TODO list manager.
4. Writing an interactive TUI file manager.
5. Writing a game.

# Reference programs and examples

If you need a hands-on reference or examples on using termion, you can check out one of the following:

1. [The termion examples](https://github.com/ticki/termion/tree/master/examples)* (easy/overview)
1. [An utility to set countdowns/reminders in the terminal](https://github.com/ticki/rem/blob/master/src/main.rs)* (easy)
1. [An utility to get the battery status from command line](https://github.com/hoodie/battery-rs)* (easy)
1. [Pokemon-style ice sliding puzzle for terminal](https://github.com/redox-os/games-for-redox/blob/master/src/ice/main.rs)* (medium)
1. [Minesweeper implementation](https://github.com/redox-os/games-for-redox/blob/master/src/minesweeper/main.rs)* (medium)
1. [Snake implementation](https://github.com/redox-os/games-for-redox/blob/master/src/snake/main.rs) (medium)
1. [An IRC client](https://github.com/ca1ek/ircim) (medium)
1. [A line-editing library](https://github.com/MovingtoMars/liner) (medium)
1. [A standalone editor](https://github.com/IGI-111/Smith) (hard)
1. [A more high-level TTY library built on top of Termion](https://github.com/Munksgaard/inquirer-rs) (hard)
1. [A Termion Xi-editor frontend](https://github.com/little-dude/xi-tui) (hard)

* = Recommended as reference or example.

If you want your program added, just contact me.
