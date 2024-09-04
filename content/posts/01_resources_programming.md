+++
title = "Resources for Programming"
date = "2022-01-27"

[taxonomies]
tags=["coding"]

[extra]
repo_view = false
+++


Sooner or later, most scientist have to automate certain tasks.
It is generally advisable to leave such repetitive things that can be 
automated to the computer. Here we have a look at tools that might help with
this task.

<!--more--> 

First things first: this is by no means going to be a comprehensive article,
but will rather be my own, fairly biased view, which heavily focuses on
[`python`](https://www.python.org/) and [`git`](https://git-scm.com/).
It hopefully starts simple, and then gets more advanced fast.

## Python

Python is a scripted computer language that allows you to do many scientific
tasks, e.g., data evaluation, automatically. It is a fairly simple language to
learn, and you can get going fast, however, be open to learn more advanced 
techniques later on. This means: Never stop learning! Python has many advanced
capabilities that can make your life easier and faster.

### Learning Python

[Software Carpentry](https://software-carpentry.org/lessons/) has two great
lessons on `python`, which are especially suited for beginners. These lessons
should give you an introduction and overview of the language, but also teach
you how to plot figures using `matplotlib`.

A great book to introduce you python was written by Allen Downy and can 
be found [here](https://greenteapress.com/wp/think-python-2e/). It is available
for free. The book gives an in-depth introduction into `python`, but also into
the basics behind the language. Such knowledge is always helpful later on, 
since it gives you a deeper understanding of certain behavior. 

### Working with Python

The official `python` distribution is distributed from 
[python.org](https://www.python.org).
If you install this distribution, you can add further packages using
[`pip`](https://pip.pypa.io/en/stable/).
However, this is not always the most straight forward way to work with `python`.

#### Virtual environments 
Virtual environments should be considered for all your projects. Depending
on what `python` environment you are using, these environments will be
created different. It is worth looking into it.

#### Anaconda
For data processing and scientific python, check out 
[Anaconda](https://www.anaconda.com/products/individual). This gives you a 
`conda` environment for `python`. Furthermore, Anaconda comes with many 
packages pre-installed and has a graphical manager to handle your distribution.
This can be advantageous, depending on what you need.

#### `pyenv` 
If you need to handle multiple different python versions and want
to easily manage virtual environments, check out 
[`pyenv`](https://github.com/pyenv/pyenv).

#### Jupyter
Finally, Jupyter Notebooks give a straight forward interface 
to `python` that allow you to play and develop code in a browser, as well as
to document it in [Markdown](https://www.markdownguide.org/).
Googles flavor of Jupyter is called 
[Google Collab](https://colab.research.google.com/) and run on the web.
JetBrains, the creator of PyCharm, also has an online Jupyter Notebook 
Server with is especially great for developing at the same time. It is 
called [Datalore](https://datalore.jetbrains.com/).

For Astrophysics, especially if you are interested in 
[NuGrid](https://nugrid.github.io/) data, check out the 
[Astrohub](https://astrohub.uvic.ca/). You can log into the public outreach
server with your GitHub account and then use a JupyterLab environment to run
your astrophysics models.

#### Editors 
Many good python editors exist. Personally, I prefer 
[PyCharm](https://www.jetbrains.com/pycharm/), however, many other options.
PyCharm is a fully integrated developer environment (IDE) and comes with many 
more tools than you need in the beginning, it can therefore be overwhelming at
first.

Other notable editors are [Spyder](https://www.spyder-ide.org/), which is a full
IDE that comes pre-installed with Anaconda. Also notable is 
[Sublime Text](https://www.sublimetext.com/), which works in my workflow
especially well for scripting. 
[Here](https://www.tabnine.com/blog/python-on-sublime-text/)
is a great article on how to set up Sublime Text for python.

### GUIs

If you are interested in creating graphical user interfaces for your programs,
it is worth looking into GUIs. Two potential GUI creation packages that you 
might want to consider using are 
[`PyQt`](https://www.riverbankcomputing.com/software/pyqt/intro/) or 
[`PySide`](https://wiki.qt.io/PySide2).
Great tutorials on Qt can be found [here](https://www.pythonguis.com/pyqt5-tutorial/).

If you want to package your GUIs with installers, 
check out [`fbs`](https://build-system.fman.io/). Note that the open / free
version only supports `python-3.6` and is restricted to `PyQt5`. If you want
to dabble with the pro version, let me know.


### Advanced Python

#### Auto-formatting
Formatting `python` code should adhere - for readability - to certain rules.
These are often also referred as linting requirements. While it is tedious to
format code by hand, automatic formatters are very helpful. I generally use 
[`black`](https://github.com/psf/black) to format my `python` code.
The beauty of this is that it there are not many possibilities to format your
code, therefore, most of the decisions are already made, and it always looks 
awesome. Various plugins exist that can be used in editors and IDEs. 
[Search engines](https://www.duckduckgo.com) are useful to find them.

#### Test your code!
Testing of code is crucial, since you generally want to make sure that your
scripts, functions, classes, etc., do what you want them to do. 
An amazing package to test your python code is 
[`pytest`](https://docs.pytest.org/en/6.2.x/). If you are interested in 
learning testing with python, check out Brian Okken's book 
[here](https://pythontest.com/pytest-book/).


## Git

When working with code, version control should be an integral part of your
workflow. One way of controlling versions is with `git`.
The Galactic Forensics Laboratory has its `git` repositories hosted 
[here on GitHub](https://github.com/galactic-forensics). Lab members get
access to all repositories of the lab.

### Learning `git`

The best way to learn `git` and / or to review your skill is by going through
[this course on Software Carpentry](https://swcarpentry.github.io/git-novice/).
The next step is then to use `git` and, if you want to keep your repos online,
a service such as [GitHub](https://github.com). You can also browse the `git`
book, which is available for free [here](https://git-scm.com/book/en/v2).

The beauty of `git` is that you can most of the time go back in time if you 
made a mistake. So don't worry if something happens! A good resource for
these weird cases is [Dangit Git](https://dangitgit.com/).


### Good practices

If you want to contribute code to a repository of which you are not a
maintainer, you should fork the repository to your own GitHub account. 
Then create a branch with an appropriate name for the feature you want to 
contribute. Add your changes, push your branch to your fork, and then create
a pull request where you describe what you have changed and why. Keep it short,
but descriptive. 

Most projects, e.g., the [`iniabu`](https://github.com/galactic-forensics/iniabu)
project have a developers guide that gives you additional information on 
how to contribute. For `iniabu`, you can find the guide, e.g., 
[here in the docs](https://iniabu.readthedocs.io/en/latest/dev/index.html).

### Advanced `git` and GitHub

#### `pre-commit`

If you are using `git` regularly, especially on public projects, `pre-commit` 
can help you to automate tasks. You can install so-called `hooks` that help
you perform various tasks, e.g., formatting, etc. Check out the
[`pre-commit` website](https://pre-commit.com/).

#### GitHub Actions

For automatic testing on GitHub, consider using
[GitHub Actions](https://github.com/features/actions). These actions can 
especially help when using continuous integration (CI).

## Some more advanced resources

### Code coverage

When testing your python code, it is useful to know how many lines of your 
code are actually tested by your test suite. To automate this process,
you can, e.g., use GitHub hooks for [`coveralls`](https://coveralls.io/).

### Documentation

Last, but surely not least, you will likely make extensive use of great
documentations that you can find online. For `python` code, automatic 
documentation using your doc strings can be really helpful. One tool to do so
is [`sphinx`](https://www.sphinx-doc.org/en/master/). 
This is especially powerful in combination with 
[ReadTheDocs](https://readthedocs.org/), which can also be implemented with 
a GitHub hook.

### Hypermodern Python

Finally, if you want to code in `python` using many bells and whistles, 
check out the blog articles on hypermodern python by Claudio Jolowicz. 
The series can be found [here](https://cjolowicz.github.io/posts/), the 
first article 
[here](https://cjolowicz.github.io/posts/hypermodern-python-01-setup/).