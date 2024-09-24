+++
title = "Simplify sharing your python application"
subtitle = "Why I started `box`?"
date = "2024-09-24"

[taxonomies]
tags=["coding", "python", "box"]

[extra]
repo_view = false
+++

The python ecosystem has changed a lot over the last year,
especially with respect to tooling.
The usage of [Rust](https://rust-lang.org) has opened new, very fast,
and by now very popular tools
that aid in linting ([ruff](https://astral.sh/ruff)) and 
package management ([uv](https://astral.sh/uv), [rye](https://rye.astral.sh)).
But what tools are there if you want to share your python applications
with colleagues that do not use python?

<!--more--> 

# Binary applications

The current ecosystem contains multiple tools that allow you
to turn your python script or program into an executable binary
that can be shared with others.
Some of these tools — by no means an exhaustive list — are:

- [PyInstaller](https://www.pyinstaller.org/): Probably one of the most used tools out there. Creates the standalone application. Can be complicated to use and ensure that all dependencies are included.
- [py2exe](https://www.py2exe.org/): Another tool to create a standalone execution, only works for Windows.
- [cx_Freeze](https://cx-freeze.readthedocs.io/en/latest/): One more tool to create standalone executions. Also seems to require extensive configuration.
- [PyOxidizer](https://pyoxidizer.readthedocs.io/en/latest/): A very interesting approach to creating python executables, now [in a zombie state](https://gregoryszorc.com/blog/2024/03/17/my-shifting-open-source-priorities/)/not maintained anymore.

All of the above tools have two drawbacks: 
They are not straightforward to use in involved projects 
and need extensive include files to ensure all dependencies are satisfied in the resulting binary. 
Furthermore, these programs "only" provide an executable 
and do not by themselves produces an installer most users would be familiar with.
The latter point seems to be especially interesting for graphical user interfaces (GUIs),
which are the main (but not sole) focus of this article.

Two python GUI packaging tools are currently available, 
[fbs](https://build-system.fman.io/) and [ppg](https://ppg.neuri.ai/).
While I have used fbs extensively in the past, 
the latest version requires a paid license 
and doesn't seem to be maintained anymore. 
In fact, this project seems much more in a zombie state than PyOxidizer,
as there is actually no communication from the maintainer about the status.
However, updates are lacking and a issues seem to pile up in the project repository,
making fbs an unattractive choice for new projects.
On the other hand I recently discovered [ppg](https://ppg.neuri.ai/),
which does basically the same as fbs. 
This tool seems be very actively maintained and is licenced under GPL-3.0.

So what do fbs and ppg actually provide?
Behind the scenes, both tools use PyInstaller to create standalone executables.
They both provide further installer creation solutions for your application 
and have been successfully used to package and release applications for multiple platforms
in CI workflows, see, e.g., [fbs with GitHub Actions](https://github.com/trappitsch/fbs-release-github-actions).
To use these tools you however need to follow a fairly strict folder structure 
and use PyQt or PySide for the GUI part of the project. 
Furthermore, the required folder structures do not necessarily align
with also creating a python package for upload to 
[PyPI](https://pypi.org/).

If these limitations are not an issue for you and you would like to ship a binary,
I recommend checking out ppg!
If you are like me and would like an alternative method for packaging,
read on!

# Non-binary standalone applications

Recently, [PyApp](https://ofek.dev/pyapp) came onto my radar. 
This tool also creates a binary executable of your python application or script,
however, it works very differently under the hood.

In brief, PyApp uses [Rust](https://rust-lang.org) to create a binary bundle
of your python code.
Upon first start, the program grabs/unpacks one of [Gregory Szorc's](https://github.com/indygreg)
[standalone python distributions](https://github.com/indygreg/python-build-standalone).
It then installs the specified dependencies, including your package/script/..., 
and runs your application. 
From the second execution onwards and since the standalone is already unpacked
and requirements installed, startup becomes very fast.

The huge difference to the other tools mentioned above is that your program
gets a standalone python environment in which your source code is simply loaded
as if the user would install your program into a virtual environment.
However, no virtual environment creation skills are required 
and python is shipped along with your code.
This makes PyApp very flexible and gives it, in my opinion, 
some huge advantages over the other tools mentioned above.

- You can create a distributable for a CLI tool, a GUI applications, a single script, and even for a jupyter notebook.
- You don't need to worry about a specific folder structure. If your program has a `[project.scripts]` or `[project.gui-scripts]` entry in the `pyproject.toml` file and gets uploaded to PyPI, you can package and distribute it with PyApp for your friends without python, all other people can simply install it via, e.g., `pipx` or `uv`.
- Your program ships with the source code in its own python environment. The user could go in and edit things if they wanted to, or at least have the possibility to do so!

Of course, PyApp is not perfect and this approach has some drawbacks. 
In order to create a standalone executable, 
you need to have a working Rust installation.
Furthermore, all specifications need to be set via environment variables
before you build the executable. 
This can be tedious, especially, if you have to do it over and over again.

*Note*: If you are using hatch for your python management,
the latter described drawbacks are not necessarily an issue for you.
You can use the 
[application builder](https://hatch.pypa.io/1.9/plugins/builder/app/)
to automatically create standalone executables for your python applications.

# Why [box](https://github.com/trappitsch/box)?

While I love the approach of PyApp to ship a standalone python distribution along with the source code,
I found the packaging process fairly tedious
and started wrapping all environment variables, etc., into a python script.
Ultimately,
I figured why not create a CLI tool that
works similar to fbs and ppg but packages with PyApp instead of PyInstaller and 
stores the PyApp configuration to apply inside the project.

Below I break the idea of box out into goals, non-goals, and maybe-goals, 
since I think these might give the best and quickest overview of what box should and should not be.

## Goals and non-goals of box

### Goals

- Initialize an existing project for future packaging with PyApp. This initialization should primarily be done by asking the user questions, i.e., guide the user through the process.
- Provide an opinionated basic configuration, but allow the user to configure PyApp further by saving user-defined environment variables.
- Provide a simple CLI interface that allows the creation of a standalone python program from your package using PyApp by simply typing `box package`.
- Provide a CLI command to take an existing PyApp standalone and wrap it into an installer.
- Work on Linux, Windows, and MacOS.
- Work with GUI and CLI programs.
- Provide some useful/opinionated standards for storing assets, e.g., icons etc., that are either based on recommended packaging practices and/or do not interfere with these practises.
- Run in CI workflows in order to, e.g., create installers for multiple OSes upon release of a package.
- Have good [documentation](https://box.rtfd.io)
- Always be open source and freely available.

### Non-goals

- Be universally configurable towards any possible configuration that PyApp supports.
- Become the de-facto packaging tool for standalone python projects.
- Allow for company specific branding, scaling, etc. (see below)

### Maybe-goals 

- Be able to create a completely self-contained standalone executable without the need of an internet connection (or maybe as [part of PyApp](https://github.com/ofek/pyapp/issues/117#issuecomment-2110405309))
- Be compatible with outdated / non-standard package environment in order to be broadly applicable ([see this discussion](https://github.com/trappitsch/box/discussions/56))


## How box basically works

In its current form, 
box can initialize a package using the `box init` command.
The project must already contain a `pyproject.toml` file 
that includes the basic package data.
Box adds a `[tool.box]` section to the `pyproject.toml` file where it stores
its own parameters on how to package a specific package.

Packaging then takes place by typing `box package`.
For this to succeed, Cargo must be installed.
After successful packaging, 
the `target/` directory contains a standalone executable
of the program.

Installers are finally created with `box installer`.
Depending on the operating system the user is on, 
this will require additional dependencies and result in different
operating system specific installer files.

For details on installation and usage, 
please check out the [documentation](https://box.rtfd.io).
Note that box is under active development and might still rapidly change.
Breaking changes are clearly labeled in every new release (if they occur).

## Philosophy

One of the non-goals states that allowing for company specific branding, scaling, etc. is a non-goal for box.
This does not mean that it will not be possible in the future,
just that is currently not an outcome I anticipate.
The (non-) goals are mostly what I currently think might happen,
but should be considered as a very flexible and fluid development plan that I have in mind.
Once box is good at its goals,
who knows where this might go.

Box is first and foremost created by a scientist (i.e., myself) 
with other scientists in mind.
This means that I want to distribute some software to my colleagues and the software should run,
they should be able to tinker with it,
but it might not necessarily look perfect on the first trial.
Remember: **Perfect is the enemy of good enough and functional!**
I want box to be my personal solution to this problem
and hope that other people might find it useful as well.

## Contribution

If you would like to contribute, 
one of the best ways currently is to test box on your projects and report back.
I am very open to feedback and suggestions on how to improve box.
Also: If you would like additional features, 
or if you would like to contribute an enhancement,
please reach out on [GitHub](https://github.com/trappitsch/box/issues)
and join the discussion there!


# Conclusions

This post outlines some of my thoughts and ideas behind starting box. 
However, these ideas are constantly developing and changing as I work on the project.
I am planning several follow up blog posts on more specific topics
over the next months
in order to dive into more detail on certain aspects and thoughts behind box.
While this might serve as an explanation on why certain features exist the way they do,
this hopefully also helps me with thinking things through in detail
and with becoming aware of potential pitfalls, etc.

If you would like to comment on this post, box, etc., 
please feel free to get in [touch by e-mail](https://galactic-forensics.space/contact/)
or by starting a [discussion](https://github.com/trappitsch/box/discussions)
on the box GitHub page.

