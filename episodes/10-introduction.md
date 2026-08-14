---
title: "Introduction"
teaching: 15
exercises: 0
---

:::::::::::::::::::::::::::::::::::::: questions

- What is a `pyproject.toml` file?
- What information do I need to include in the file for my project?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Be able to create a `pyproject.toml` file for a small project.

::::::::::::::::::::::::::::::::::::::::::::::::

# Packaging Python Code


## The Problem of Packaging

Once you get beyond the most basic of programs you are going to start depending on other code to do things. At the most fundamental level it may be calling out to system functions, or standard libraries of the language you are working with, but in the real world there are likely to be many packages and libraries of code that you end up using to a greater and lesser degree.

This may be acceptable when you are the only consumer of your code, but as soon as you start wanting to share your code with others, you have the problem of how to ensure that the code that your program depends on is available.

There are a number of ways of approaching this:

- include everything with your code (possibly up to and including the hardware it runs on!).  This is particularly applicable for *applications* which may rely on operating system functionality and nothing else.

  Examples of this approach include Docker/Containers, C/C++ static builds, MSI installers on Windows, and tools like pyinstaller or briefcase in the Python world.

- include information about the dependencies alongside the code and rely on installation technologies to understand these dependencies and install them. This is common for code *libraries* that are themselves used by other code.

  Examples of this approach includes Linux packaging technologies like APT or YUM, systems like Homebrew on Mac and Nuget on Windows, and the dependency resolution system used by pip/PyPI.

In either case you need to be able to *specify* those dependencies, either because your build system needs to know what to include in the build, or because your installer needs to know what to add to the system.

::: callout

### The History of Packaging on Python

Originally Python had no packaging system: you simply downloaded and copied Python files into somewhere that Python would find them (usually via the `PYTHONPATH` environment variable).  Eventually there was a website called "The Vaults of Parnassus" where archives of Python libraries were stored and where you could download the source (this was well before GitHub existed).  This worked OK for pure Python code, but for Python extensions written in C/C++ there was typically a compilation step which used typical Unix tooling---like configure scripts and Make---and which didn't work on other platforms.

As a result, around 2000, Python introduced the `distutils` library that gave a cross-platform way of building Python packages and extensions, and introduced the `setup.py` script. While this worked well for pure Python packages, C/C++ extensions required you to have a C/C++ compiler available (generally not a problem on Unix, but potentially expensive on Mac and Windows), and even then often it needed to be exactly the right compiler.  Packages like SciPy were notoriously difficult to build as you needed both C and Fortran compilers.

So shortly after the introduction of distutils, tooling was developed for binary packaging: `easy_install`, `setuptools` and the `.egg` format (as in what snakes hatch from); and companies like ActiveState, Enthought and (eventually) Continuum/Anaconda started to build businesses around packaging.

The `setup.py` and `.egg` systems worked OK, but had the problem that they relied on *scripts*: a `setup.py` file could contain arbitrary code, and eggs could specify scripts to run as part of installation.  Great for flexibility, but poor for discoverability.

The next major development was the creation of the "cheese shop", now better known as the Python Package Index (or PyPI) as a standard repository of Python packages and evetually, after a *lot* of politics, the creation of the "wheel" format (as in a "wheel" of cheese), `pip`, and `pyproject.toml` to replace eggs, `easy_install` and `setup.py` respectively.  The primary advantage of wheels and `pyproject.toml` is that they are *not* scripts, and so can be reliably analysed and installers do not need to run arbitrary code that can literally do *anything* to your computer.

The point of this discussion is that you may still find the remains of this history when you look into old research codebases: now you know what `easy_install`, `setup.py` and "egg" files are if you encounter references to them.

And also that packaging is **hard** to do, and even harder to get right. It's taken Python over 30 years.

:::

## Packaging Python Scripts

A common problem in research software is that you have a Python script that performs some analysis that you want to share.  A new way to do this is via ["Inline Script Metadata"](https://packaging.python.org/en/latest/specifications/inline-script-metadata).

If you add a specially formatted comment like the following:
``` python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "pytorch>2.11",
#   "pillow",
# ]
# ///
```
then tools like pip can use this information to install the dependencies into a virtual environment using the [`--requirements-from-script`](https://pip.pypa.io/en/latest/cli/pip_install/#cmdoption-requirements-from-script) command-line option.

This is great for *replication*: research scripts can specify under what conditions they expect to run, and then other tooling can ensure that there is an appropriate environment for the script to run as designed. If needed, precise versions can be specified.

::: callout

### What Packages are Your Dependencies?

A good rule of thumb is that if you directly import code from a package, then it should be a dependency.  In some cases this isn't strictly necessary because of chained dependencies: for example, if you depend on `torchvision`, then `torchvision` depends on `torch` and so you will automatically get `torch` installed and don't need to specify it as a separate dependency.

However it's good practice to specify `torch` as a dependency as well if you directly import it as a form of defensive programming: if at a later point you remove the use of `torchvision` but keep the usage of `torch` you're less likely to have problems with your dependencies.

The exception to this is when publishing code in support of research—when you care about the ability to replicate—you may want to specify your dependencies exactly and completely. You can get a list of all currently installed packages in an environment with `pip freeze`.  The result of this should allow others to reproduce your Python environment, assuming the same Python version and OS.

:::

## Specifying Dependencies

The simplest way to specify a dependency is just by its name: `"numpy"` or `"pytorch"`.  This name is the name of the *distribution* and not the name of the *package* it installs (for example, `pytorch` installs the `torch` package).  This is because a single distribution can install multiple packages. To find the correct distribution name, search on the [Python Package Index](pypi.org).

Installers, like `pip` or `uv` will use this information to find a compatible set of dependency versions and install them. While this is very flexible, it runs the risk of installing an older or newer version of your dependency that isn't compatible with your code. To get around this, you can add additional data that indicates which versions of your dependencies your code is compatible with.

You can use standard comparison operators (`<=`, `>=`, `==`, `!=` and so on) to specify ranges of versions of your dependencies, speparated by commas, so `pytorch >= 2.10,<2.15,!=2.11.4` means any version of pytorch from 2.10.0 up to, but not including 2.15.0, but also excluding the exact version 2.11.4. You can also use wildcards in your expression, such as `pytorch==2.12.*` which means any version of PyTorch 2.12. The operator `~=` means any version compatible with the specified version, so for example `pytorch ~= 2.11` means any version from 2.11.0 up to, but not including, 3.0 (if/when it comes out).

Sometimes you may need to specify a dependency which is only needed for a particular platform or python version. In this case you can follow the version information with additional filters after a semicolon such as `typing-extensions>=4.13; python_version < 3.14` - this will install `typing-extensions` only if your package is installed on Python 3.13 or earlier.  You can use `sys_platform` to detect windows vs. linux vs. mmacOS, and there are similar identifiers to detect things like machine architectures (eg. ARM vs. Intel).  The full set of options can be found in the [Python Packaging User Guide](https://packaging.python.org/en/latest/specifications/dependency-specifiers/#defined-environment-marker-fields)

Finally, if absolutely necessary, you can specify a URL which contains the distribution that you want to install.  This can be a link to a binary wheel, a zip file with the code, or even to a git repository.  This allows you to do things like depending on non-open code in a private repository that you have access to.

::: callout

### Dependencies on the Command-line

You can use comparison operators when installing packages from the command-line using tools like `pip` or `uv`, but you need to take care because `>` and `<` have special meanings to console and other similar command shells. If you need to specify version ranges on the command line, it's generally a good idea to put quotes around them, eg.:
``` console
pip install "pytorch >= 2.11"
```

:::

::: callout

### Supply-Chain Attacks

When you are using other people's code, you are implicitly *trusting* other people's code. This trust is something which can be exploited by hackers.  There are two main ways that this trust can be exploited:

- **"typo-squatting"** - it's easy to make mistakes when typing in the names of distributions, particularly when the name of the distribution doesn't match the name of the package (eg. is it `torch` or `pytorch`? Is it `scikit-learn` or `scikit_learn` or `sklearn`?). Bad actors can create malicious libraries which have names close to or easily mistaken for popular packages which can then run arbitrary code on your system with your privileges.  Most package repositories monitor for bad packages like this, but you should always take care that your dependencies are what you think they are.

- **supply-chain compromises** - if attackers can gain the credentials to write to a popular package they can potentially inject malicious code into the package which then is run by users of that package.  This can either be via compromising developer accounts, or using social engineering to gain the trust of the core developers of a package and then gaining commit rights to the project.  In at least one prominent case, a state-level actor took control of an open-source project.

The main defence against these sorts of attack is vigilance: take care with dependencies, and make sure that you keep some visibility on the state of those projects (eg. subscribing to announcements, or reading news sources which report on these sorts of issues). If projects are compromised, they are usually up-front about the problem.

:::

## Packaging Python Libraries

The current standard for specifying dependencies and build information in Python is the `pyproject.toml` file.  There are other, older ways of doing this that you may still see, such as a `requirements.txt` file, but for new code it is best to use `pyproject.toml`.

However for packaging you need to specify more than dependencies in most cases. Particularly if you are going to share your code on GitHub or PyPI, you will need to specify the name of your package, the version, the authors, contact information, descriptions of what the package does, where to find the original source code repositories, and so on.

A minimal `pyproject.toml` might look something like this:
``` toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "example-project"
version = "0.0.1"
dependencies = [
    "numpy",
    "pillow",
    "torch",
    "torchvision",
]
requires-python = ">=3.13"
authors = [
  {name = "A person", email = "A.Person@soton.ac.uk"}
  {name = "Another person", email = "A.N.Other@soton.ac.uk"}
]
maintainers = [
  {name = "A person", email = "A.Person@soton.ac.uk"}
]
description = "An example project."
readme = "README.rst"
license = "MIT"
license-files = ["LICEN[CS]E.*"]
keywords = ["example", "coffee"]
classifiers = [
  "Development Status :: 4 - Beta",
  "Programming Language :: Python"
]
```
Not all of these are strictly needed for packaging, but are a good idea.

There are additional optional sections that you can include, such as specifying optional dependencies (eg. for a GUI or for development work), command-line scripts and configuration options for tools such as linters.

::: callout

### TOML

TOML, or "Tom's Markup Language", is a human-readable data format along the lines of XML, JSON, YAML and similar, but designed to be somewhat easier for humans to *write*. It is somewhat inspired by the `.ini` configuration file format used in older Windows apps, but less complex and a more modern.

:::

### Build-system Options

The `build-system` section tells Python tooling how to build the package, both for installation from source and for producing wheels for distribution.

See [PyOpenSci's discussion for how you might go about choosing a build system](https://www.pyopensci.org/python-package-guide/package-structure-code/python-package-build-tools.html). Unless you are doing something complex (such as wrapping a large C/C++ library), the following are good choices:

- **SetupTools**: legacy, but battle-tested and it generally just works.
- **Hatch/Hatchling**: Hatch is a comprehensive solution. It can set up a project for you, create environments, and run tests. However it is opinionated about the tooling it uses to do this (eg. it is some work if you don't want to use `pytest` or `uv`).  Hatchling is Hatch's build backend, but it can be used independently.

A typical build-system section might look like this:
``` toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```
If you depend on other packages when you are *building* your code (eg. Cython), those should be added to the `requires` section and follow the same rules for specifying dependencies as described above.

### Dependencies and Optional Dependencies

The `[project]` section of the `pyproject.toml` contains the project's metadata. A lot of this is information to help PyPI get information about the package to help with searching anc categorising.  But some of the data is used by tools like `pip`. In particular, you need a `name` and `version` field to be able to install your package with `pip` or `uv`.  Depending on what language and standard library features you use, you may want to specify the Python version that your code requires.

But most importantly, it includes the `dependencies`, which is a list of your project runtime dependencies.  These dependencies are dpecified in the same format that was discussed earlier.  In general, for library code, it is better to specify just the few top-level dependencies.  At a minimum this should be every 3rd party library that you directly import. You can generally assume that your dependencies will specify what they need through their own `pyproject.toml` but you may need to specify a few more projects depending on the circumstances: some Python code may test to see whether a package is installed before trying to use it rather than having a direct dependency on it.

There is a mechanism for distributions to specify these sorts of "optional dependencies" or "extras".  These allow you to give a name to additional sets of dependencies.  Common use-cases include specifying extra dependencies that are needed for developers on the project (eg. tools for linting, compilation, testing, and documenatation), optional additonal features (such as a command-line or GUI front-end), or support for particular less-common use cases (such as reading from certain file formats, or support for particular databases).

These are specified as their own `[project.optional-dependencies]` section which includes a mapping of extra names to the corresponding dependencies:
``` toml
[project.optional-dependencies]
dev = [
  "pytest",
  "black",
  "pylint",
]
gui = ["pyside6"]
cli = ["click"]
```
The dependency names are up to you (there is nothing special about `dev`, `gui` or `cli` in the above example), and the dependencies themselves follow the same rules as other dependencies discussed earlier.

To install a distribution and one or more of its extras, you add the extras you want in square brackets after the name of the package.  For example to be able to read and write Excel spreadsheets from the Pandas library, you need to include the `excel` extra, and for reading and writing HTML pages you need the `html` extra.  At the command-line installing both would look like:
``` console
pip install "pandas[excel,html]"
```
The quotes are needed as console and other shells consider square brackets to have special meaning.

To specify in a `pyproject.toml` it would be similar:
``` toml
dependencies = [
  "pandas[excel,html]",
]
```

::: callout

### Dependency Groups

A recent way of specifying additional dependencies are "dependency groups", which are specified similarly to extras, but using the top-level `[dependency-groups]` section, and which have slightly different behaviour:

- they don't require the package itself to be installed (which saves time for installing build tools, linters or similar tools that operate *on* the code);
- dependency groups can include other dependency groups; and
- they are installed via tools like `pip` with the `--group` argument.

This can be convenient for projects which aren't libraries or single scripts, such as projects with Jupyter notebooks, as a way to specify dependencies without having to install a library.

For example, to install requirements for a jupyter notebook which uses pandas, you might have the following in your `pyproject.toml`:
``` toml
[dependency-groups]
notebooks = ["notebook", "pandas"]
```
and then you could install the dependencies into your virtual environment with:
``` console
pip install --group notebooks
```

:::

### Installing for Development

Once you have a `pyproject.toml` you can install your project in "editable" form into your working virtual environment using `pip` or similar tools. At the command-line, change directory into the top-level of your project where your `pyproject.toml` is, and run:
``` console
pip install -e .
```
This installs your library and its dependencies into your current virtual environment, but in a way so that if you edit your library then the changes are picked up when you restart Python and import your package.

If you change your dependencies you will need to re-run the command to pick them up.

### Data files

Often you need to distribute more than just Python code. Maybe it's the wieights for a deep learning model, maybe its a small database, maybe its images, HTML templates and Javascript for a web app. Whatever it is, you need to tell your build system to include those files when it builds the distribution.

There isn't a standard way to do this: different build backends have different ways to specify additional files that should be included in the built distribution.

#### Setuptools

Setuptools accepts a number of different ways of specifying additional files.

The easiest way is to use a `[tool.setuptools.package-data]` section to give patterns for files to include:
``` toml
[tool.setuptools.package-data]
"*" = ["*.txt", "*.safetensors"]
```
will add all text files and safetensor weights in any package to your distribution.  There are also options to specify files to exclude, and the directories to search for files in.

#### Hatchling

Hatchling understands `.gitignore` files, and so by default it includes all files in your packages that are under version control, which is often good enough.  You can turn this off, and you can also add or remove files from inclusion, but this is done separately for source distributions and wheels.

Exact details can be found in the [Hatch documentation](https://hatch.pypa.io/latest/config/build/#file-selection).


## Building Distributions

Python has two common ways of distributing packages: source distributions and binary wheels.  For pure Python packages there isn't a lot of difference between the two formats, but binary wheels are required if you want to distribute extension modules without requiring that that user has a compiler on their system.

It's a good idea to build your library regularly as part of continuous integration to make sure that nothing is broken.

The standard Python tool for building both wheels and source distributions is `build`. You can install it into your environment using `pip`:
``` console
pip install build
```
and then you can use the `build` command to build wheels, source distributions, or both:
``` console
python -m build
```
The build artifacts produced will be located in the `dist` directory by default.

### Building on Other Platforms

The `build` tool has the limitation that it can only build either pure Python wheels or wheels for the current platform and Python version.  If you need to build wheels which include extension modules then the [`cibuildwheel`](https://cibuildwheel.pypa.io/en/stable/) project provides a CI tool (available for GitHub and GitLab in particular) which will automatically set up virtual machines and run `build` on a wide variety of platforms. Take care when doing this with private repositories: the build process is time-consuming and may eat your free CI time, particularly if you enable some of the more exotic platforms.

## Publishing Distributions to PyPI

When you are ready to release your code, after you have built it, you can use the Twine tool to publish your code to your project on PyPI.  You don't have to use Twine: you *can* manually upload files, but Twine significantly improves the experience.

Before using Twine, you will need to create an account on PyPI, and then reserve your distribution

To use Twine, you install it with pip:
``` console
pip install twine
```
You can then check that your distribution metadata renders correctly using `twine check`
``` console
twine check
```
PyPI provides a "Test PyPI" instance that you can use to verify that everything looks correct:
``` console
twine upload -r testpypi dist/*
```
When you are happy that everything is OK, you can then upload with:
``` console
twine upload dist/*
```

## Packaging Applications





