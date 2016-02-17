PEP: XXX
Title: An Abstract Build Pipeline for Wheels
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: DistUtils mailing list <distutils-sig@python.org>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 10-Feb-2016


Abstract
========

This PEP proposes a build pipeline that takes a Python package from an
arbitrary directory to a fully built Wheel. It does this not by describing how
the build process or by blessing a particular implementation, but by instead
describing the path in which a Python package goes through, and a set of APIs
by which any tool can be be used to implement each specific step, as long as it
adheres to the API in this PEP.


Rationale
=========

For a long time, Python packaging has been dominated by one of two build tools
and the assumption that you're using one of these two tools is baked into many
layers of the packaging toolchain. Within the last couple of years, the arrival
of the Wheel packaging format has started to form a path that allows us to
start to break the hard dependency on these two build tools. However, it is
still the case that there is an implicit assumption that when we're installing
from either a sdist or a arbitrary directory that distutils/setuptools is being
used.

An implicit, hard dependecy on distutils/setuptools has been problematic for a
number of reasons:

* The needs of a simple, pure python project are vastly different than the
  needs of a complex project like Numpy which needs to build against several
  languages.

* The design of the system makes it dificult to utilize reusable libraries in
  the build process, requiring that any imports be delayed until *during* the
  execution of the ``setup()`` function, else the import will fail when the
  build requirements are not satisified.

* The system itself lives in too many layers of the process, making it dificult
  for people to swap it out from one piece for another without having to
  reimplement the entire logic of what distutils/setuptools provides.

Further, there are numerous "paths" that any particular installation could have
taken to go from a directory on disk to a fully installed package. Each
different path creates a sort of combinatorial explosion of possible different
outcomes, which we hope all end up the same, but which often don't in practice.
This makes it difficult for a project to know exactly how their project is
going to get installed. By defining a clear and consistent pipeline, we can
eliminate many of these additional paths and make it easier to reason about
what will happen for any particular installation.


Build Pipeline
==============

This PEP defines the build pipeline, the process by which all conformant
packages must be able to go through, as::

    VCS/Arbitrary Directory -> Source Distribution -> Wheel -> Installed
                            \-> Editable Install

This represents a departure from the previous "pipeline" which aillowed::

    VCS/Arbitrary Directory -> Source Distribution -> Wheel -> Installed
                                                   \-> Installed
                            \-> Wheel -> Installed
                            \-> Installed
                            \-> Editable Install

Build Pipeline API
------------------


File Format
~~~~~~~~~~~

This PEP proposes a human editable file that is used to describe the tool that
should be invoked to handle each stage of the build pipeline.

This PEP chooses the TOML file format due to the fact it is easily readable and
writable by humans, but it still maintains sane parsing and real datatypes. An
example of this file is::

    [source]
    command = "mybuildtool.build.source"
    dependencies = [
        "mybuildtool",
    ]

    [build]
    command = "mybuildtool.build.wheel"
    dependencies = [
        "mybuildtool",
    ]


Source Wheel Creation
~~~~~~~~~~~~~~~~~~~~~

The source wheel creation is configured by specifying a command that will be
executed using ``python -m {command}``. Using the above example, the base
command to be executed will be ``python -m mybuildtool.build.source``.

This command must support two subcommands, ``metadata`` and ``create``. Both
of these commands must accept two positional arguments, the first of which is
the current location of our source tree in disk, and the second of which is the
location that we expect the command to write the data out onto disk at. Using
the example from above, we have::

    python -m mybuildtool.build.source {metadata,create} source destination

Using these two commands, a source wheel can be produced using::

    $ python -m mybuildtool.build.source metadata . build/
    $ python -m mybuildtool.build.source create . build/
    $ cp ./pypa.toml build/
    $ cd build/ && zip -r ../{name}-{verison}.src.whl ./*

The assumption is that a higher level command tool (such as twine) will be
created that encapsulates the above series of commands to make creating a
source wheel as easy as ``twine source-wheel .``.


metadata
````````

The metadata command must produce the ``DIST-INFO/`` directory from PEP XXXX
in the desintation directory.

create
``````

The create command must produce the ``SRC/`` directory from PEP XXXX in the
destination directory. This command cannot be called on a directory that has
not already had the metadata command called on it, and any metadata that it
needs in the creation of the ``SRC/`` directory *MUST* come from reading the
metadata files inside of the ``DIST-INFO/`` directory.


Wheel Creation
~~~~~~~~~~~~~~

The wheel creation is configured by specifying a command that will be executed
using ``python -m {command}``. Using the aboe example, the base command to be
executed will be ``python -m mybuildtool.build.wheel``.

This command must support two subcommands, ``metadata`` and ``build``. Both of
these commands must accept two positional arguments, the first of which is the
current location of an unpacked source wheel on disk, and the second of which
is the location that we expect the command to write the data out onto disk at.
Both commands must also take an option of either ``-b`` or ``--build-dir``
which, if present, tells the build tool which directory to use as the build
directory. Using the example froma bove, we have::

    python -m mybuildtool.build.wheel {metadata,build} [-b/--build-dir build] source destination

Using these two commands, a binary wheel can be produced using (roughly)::

    $ unzip {name}-{version}.src.whl -d /tmp/source-wheel/
    $ python -m mybuildtool.build.wheel metadata /tmp/source-wheel/ /tmp/wheel/
    $ python -m mybuildtool.build.wheel build /tmp/source-wheel/ /tmp/wheel/
    $ zip -r {name}-{version}-{tags}.whl /tmp/wheel/

As with the source wheel creation, the assumption is that a higher level tool
(such as twine) will be created that encapsulates the above commands to make
creating a wheel as easy as ``twine wheel ./foobar-1.0.src.whl``.


metadata
````````

The metadata command must produce the ``{name}-{version}.dist-info/`` directory
from PEP 427 in the destination directory. This command *MUST* produce metadata
where any value that is present in the source wheel *MUST* be equal to the
value that is present in the binary wheel.


build
`````

The build command must produce all of the non metadata files from PEP 427 in
the destination directory. This includes copying over any ``.py`` files,
compiling ``.c`` files into ``.so`` files, and similar. The final output of
this command should be an unpacked, but otherwise ready to install wheel. This
command cannot be called on a destination directory that has not already had
the metadata command called on it, and any metadata that the build command
needs *MUST* be read from the ``{name}-{version}.dist-info/`` directory.


Frequently Asked Questions
==========================

Why do none of these APIs produce ready to use Files?
-----------------------------------------------------

The APIs in this PEP all consume and produce *unpacked* distribution files.
They do not actually create a file that is ready to be uploaded. This allows
these tools to have a smaller focus, they don't need to worry about how to
format the filenames, what compression to use, etc. They only need to do the
bare minimum. It also makes it possible for tools to optimize the procedure a
bit by skipping needless pack+compress -> decompress/unpack cycles that would
otherwise be required. For instance, if someone is building a wheel from an
arbitrary directory, the tooling could generate an unpacked source distribution
in a temporary directory, then skip compressing it and just send that directly
into the wheel build stage.

Of course, this means that these APIs are not, on their own, enough to produce
a working file to ditribute. Instead it will be the job of a higher level tool
to handle this. The reference implementation will be implemented in the twine
tool, where authors will be able to do things like replace ``setup.py sdist``
with ``twine sdist .`` or similar. This higher level tool, ``twine`` in this
example, would be responsible for any compression/decompression and such that
are required. This allows end users to have a consistent tool that can operate
on all packages, without having to care or worry about what tooling is required
to actually operate on that package.


Why TOML instead of JSON/YAML/INI/etc?
--------------------------------------

Immediately, JSON is an attractive choice because it is included in the
standard library which makes it trivial to include support for it and it has
real data types which makes it nicer than INI. However, JSON is not a good
format for humans to have to write by hand. It lacks important things like
comments and it has issues like trailing commas in lists/dictionaries that make
it trivial for a human to accidently have invalid JSON.

YAML is a popular alternative to JSON for human writable files, made nicer by
the fact that JSON is actually a subset of YAML. However, it is not included in
the stdlib and the main library for parsing it includes (optional) C extensions
but more importantly, it does not use a single source for Python 2.x and 3.x
which means that projects like pip cannot utilize them.

INI is also supported in the standard library, however it's lack of real data
types makes it dificult to easily represent more complicated concepts in it.
At this time we don't have incredibly complicated needs, however it's expected
that other projects, particularly the build tools, may decide to reuse this
file to keep all of the build configuration in one location and those projects
are likely to have a more complex requirement.

TOML has real data types, and it's friendly enough that huamns can easily write
it and it has useful features like comments. Finally it is available in OSS
licensed, pure Python, single 2.x/3.x source libraries that projects like pip
can use. The choice of TOML represents a pragmatic compromise.


Why a CLI instead of a Python API?
----------------------------------

There are two general approaches that we could take to implementing this API.
The first is using a command line based approach, as we do now, and the second
is a Python API based approach.

This PEP uses a command line based approach for a few reasons:

* It removes any assumption about the build tool being able to monkeypatch the
  calling tools.

* It provides a nice, clean way for a build to provide incrimental updates as
  to the progress of the command.

* The process interface is well understood and simple, which makes it easier
  to handle than attempting to write a process shim that just calls a Python
  API when projects like pip need to call into it.


Why using python -m instead of a CLI?
-------------------------------------

An alternate method of specifying what command to run could be to allow the
project to execute any arbitrary command. This PEP instead chooses to have the
specified command be a python module that can be executed using ``python -m``.

Due to the nature of how the install process works, any command we're going to
execute needs to be something that is pip installable into our current Python
and we need a method to pass into the build tool what particular Python they
are building for. This is largely a subjective choice, but it is the opinion of
this PEP that by using the ``python -m`` mechanism we make it more obvious for
build tool authors that their dependency should be pip installable, and it
needs to be executed under the Python that we're installing into.


What about Editable Installs?
-----------------------------

.. note::

    Is this reasonable? Does our Pipeline prevent a reasonable editable
    install? What does an editable install entail?

This was mentioned earlier in the build pipeline, but this API doesn't actually
contain any mechanism for handling an editable install. This is on purpose. The
topic of editable installs is a complicated set of trade offs with a number of
edge cases. It is the opinion of this PEP that this deserves it's own PEP and
thus it defers sorting out the solution to a future PEP. This means that things
like ``pip install -e`` cannot (currently) be used for a project taking
advantage of this PEP.


Rejected Proposals
==================

Define the ``setup.py`` API contract
------------------------------------

A sort of "minimally invasive" option is to simply define the commands and
options that we require of the ``setup.py`` interface. This would allow us to
continue to use distutils/setuptools as we do now, but still allow people to
write their own build tool implementations.

However, this isn't all we would need to do, because part of the problem with
the current toolchain is that there is no mechanism to declare the things that
need to be installed prior to executing the ``setup.py`` file. The only thing
close is the ``setup_requires`` option in setuptools. This has a number of
problems though, since the item being invoked is in control of trying to
satisify the dependencies, options like which repositories should be installed
from and the like cannot easily be passed down into that invocation.
Additionally, since setuptools doesn't have control of the process until the
``setup()`` function is being called, anything that relies on those libraries
being installed must be delayed until later on in the execution.

Thus it is the opinion of this PEP that to make this solution palatable it
would require defining a seperate, static file, that can be used to indicate
what dependencies need to be installed prior to executing the ``setup.py``.

Additionally, the current system has the problem that the pipeline that a
project goes through isn't well defined which has, in real world situations,
caused problems where invoking it in one way would cause different outcomes.

Additionally, for projects that are not distutils/esque, the outcome will be to
have a tiny shim ``setup.py`` that does nothing but invoke the real build
system. It is the opinion of this PEP that it is better to just invoke the real
build system directly instead of going through the ``setup.py`` shim.


Allow replacing setup\.y invocations with something else
-------------------------------------------------------

PEP XX proposes another alternative which combines the ability to specify
"bootstrap" dependencies in a static file and then it simply replaces the
invocations that pip would do to ``setup.py egg_info`` and
``setup.py bdist_wheel`` with another invocation.

It is the opinion of this PEP that this does not go far enough in what it hopes
to accomplish, and infact it represents a regression in some forms.

First, it does not mandate any sort of ability to create a source distribution,
however not creating a source distribution is something that should be frowned
upon in the general case since it prevents downstream distributors like Debian
from being able to redistribute the code provided by that project. PEP XX
punts on this decision since it's not strictly related to the concept of
building, however since distutils/setuptools currently handles both situations
you cannot replace distutils/setuptools for one part of that operation without
also replacing it for the other, so it is the opinion of this PEP that any
method of replacing setuptools for building wheels *must* also include a method
for replacing setuptools for building sdists. To do otherwise would incentivize
people to either not use the new system, or more likely, to not produce source
distributions at all.

Secondly, it does not specify any particular kind of pipeline that a build must
go through. It is the opinion of this PEP that it should. The reasons for this
are:

* If you can go from an arbitrary directory straight to a wheel or to an
  install than all layers of the process need to know how to do things like
  generate a version from a VCS revision (for projects that wish that) instead
  of that needing to be something that only exists in the sdist stage. This
  makes it hard to have two different tools handling the actual build process
  of the wheel, and the process of creating the sdist. By defining a set
  pipeline each tool

* Every additional "path" that an installation can take through the process
  complicates the tooling and the mental model required to handle the
  ecosystem. Limiting the paths instead makes it much easier to reason about.



Copyright
=========

This document has been placed in the public domain.


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8

