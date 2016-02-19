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

An implicit, hard dependecy on distutils/setuptools is problematic for a number
of reasons:

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

This represents a departure from the previous "pipeline" which allowed::

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

    [build]
    builder = "mybuildtool"
    dependencies = [
      "mybuildtool",
    ]


API Entrypoint
~~~~~~~~~~~~~~

The entrypoint to the pipeline API is defined using a key ``builder`` inside of
the ``build`` table in ``pypa.toml``. The value of this key is a string using
the format ``path.to.import:object.to.use``. This string can be resolved to an
object or module using a functional equivalent to::

  import path.to.import
  builder = path.to.import.object.to.use

The ``:object.to.use`` portion may be omitted, in which case it is functionally
equivalent to::

  import path.to.import
  builder = path.to.import

Each of the functions defined as part of the pipeline API must be an attribute
of the final resolved object or module.


Source Wheel Creation
~~~~~~~~~~~~~~~~~~~~~

Building a source wheel is handled using two functions, ``source_metadata`` and
``source_wheel``. These two functions can be utilized using something like::

    import email.parser
    import os.path
    import re
    import tempfile
    import zipfile

    SOURCE = "."
    DESTINATION_DIR = "."

    def zipdir(ziph, path):
        for root, dirs, files in os.walk(path):
            for file in files:
                filepath = os.path.join(root, file)
                ziph.write(filepath, os.path.relpath(filepath, path))


    with tempfile.TemporaryDirectory() as tmpdir:
        swhl_dir = os.path.join(tmpdir, "swhl")
        build_dir = os.path.join(tmpdir, "build")

        builder.source_metadata(SOURCE, swhl_dir, build_dir=build_dir)
        builder.source_wheel(SOURCE, swhl_dir, build_dir=build_dir)

        with open(os.path.join(swhl_dir, "DIST-INFO", "METADATA"), "rb") as fp:
            metadata = email.parser.Parser().parsestr(fp.read().decode("utf8"))

        name, version = metadata["Name"], metadata["Version"]

        filename = "{}-{}.swhl".format(
            re.sub(r"[-_.]+", "-", name).lower(),
            re.sub(r"[-_.]+", "-", version).lower(),
        )
        destination = os.path.join(DESTINATION_DIR, filename)

        with zipfile.ZipFile(destination, "x", compression=zipfile.ZIP_DEFLATED) as zf:
            zipdir(zf, swhl_dir)
            zf.write(os.path.join(SOURCE, "pypa.toml"), "pypa.toml")


source_metadata
```````````````

The ``source_metadata`` function's purpose is to take a directory of files and
build the metadata for the source wheel and place it on disk. Its signature
is::


    def source_metadata(source: str, destination: str, *, build_dir: str = None):
        pass

The ``source`` argument is the location on disk of the directory of files for
which we are computing the metadata for. The ``destination`` is the directory
where the PEP XXXX ``DIST-INFO/`` directory for the source wheel should be
written to.


source_wheel
````````````

The ``source_wheel`` function's purpose is to take a directory of files, and
build the actual contents of a source wheel, sans the metadata, and place it
on disk. Its signature is::

    def source_wheel(source: str, destination: str, *, build_dir: str = None):
        pass

Much like the ``source_metadata`` function, the ``source`` argument is the
location on disk of the directory of files that we are building the source
wheel of and the ``destination`` argument is the directory on disk where the
source wheel's PEP XXXX ``SRC/`` directory should be written to.

This function *MUST* always be called with a ``destination`` that has already
had the ``source_metadata`` function called on it. Implementations of this
function *MAY* verify that the metadata is accurate but *MUST NOT* modify any
files outside of the ``SRC/`` directory.


Binary Wheel Creation
~~~~~~~~~~~~~~~~~~~~~

.. note::

    The name "Binary" Wheel refers to the traditional ``.whl`` files and it is
    not specific to wheels that contain compiled code.


Similarly to building a source wheel, building a binary wheel is handled using
two functions, ``binary_metadata`` and ``binary_wheel``. These two functions
can be utilized using a similar chunk of code to building a source wheel.


binary_metadata
```````````````

The ``binary_metadata`` function's purpose is to take an unpacked source wheel
and build the metadata for the binary wheel and place it on disk. Its signature
is::

    def binary_metadata(source: str, destination: str, *, build_dir: str = None):
        pass

The ``source`` argument is the location on disk of the unpacked source wheel
that we are building the binary wheel metadata for. The ``destination`` is the
directory where the PEP 427 ``{name}-{version}.dist-info/`` directory must be
written to.

Any metadata that exists in the source wheel must be exactly equal when it has
been moved into the binary wheel. It is invalid for a binary wheel to, for
example, have a different version than the source wheel that it was created
from.


binary_wheel
````````````

The ``binary_wheel`` function's purpose is to take an unpacked source wheel and
build the actual contents of the binary wheel, sans the metadata, and place it
on disk. Its signature is::

    def binary_wheel(source: str, destination: str, *, build_dir: str = None):
        pass

Much like the ``binary_metadata`` function, the ``source`` argument is the
location on disk of the unpacked source wheel that we are building a binary
wheel for. The ``destination`` argument is the directory on disk where the
PEP 427 wheel contents must be written to. This will include copying over any
``.py`` files that should be installed, compiling ``.c`` files into ``.so``
files, etc.

This function *MUST* always be called with a ``destination`` that has already
had the ``binary_metadata`` function called on it. Implementations of this
function *MAY* verify that the metadata is accurate but *MUST NOT* modify the
metadata itself.


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


Why a Python API instead of a CLI Based API?
--------------------------------------------

.. note::

    Steal the explanation from PEP 517.


What about Editable Installs?
-----------------------------

.. note::

    Is this reasonable? Does our Pipeline prevent a reasonable editable
    install? What does an editable install entail? Arguably breaking
    ``pip install -e`` for projects that decide to switch to this feature is
    a negative.

    The current editable install code *typically* does an inplace build, drops
    a .egg-info metadata into the directory and then creates a egg-link
    (similar to a symlink). This means that for projects that are pure python,
    you can edit your .py files and changes will be reflected immediately,
    however for projects that have any sort of build step, it requires running
    another inplace build to regenerate the files. In addition, if a project
    needs to modify the .py files at all (such as with 2to3) this cannot be
    done in place, and instead the current code builds to a build directory and
    then uses that instead. Editable installs *also* install any scripts that
    need to be generated as part of the installation.

    In addition, editable installs currently have a problem with stale
    metadata. If you use a system that say, uses a git derived version, then
    it's possible that between the point that the metadata file was generated
    during the editable install, and when the code has run, commits have
    occured and the version has increased. This can lead to problems if the
    software checks the version at all.

    All in all, coming up with a signifcantly better editable install is
    absolutely outside of the scope of this PEP, however trying to get
    something similar to the status quo may not be. It comes down to whether
    baking in a "broken" but status quo API into the pipeline API that allows
    us to not break ``pip install -e`` is a better choice than

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

PEP 516 and PEP 517 propose other alternatives which combines the ability to
specify "bootstrap" dependencies in a static file and then it simply replaces
the invocations that pip would do to ``setup.py egg_info`` and
``setup.py bdist_wheel`` with another invocation.

It is the opinion of this PEP that this does not go far enough in what it hopes
to accomplish, and infact it represents a regression in some forms.

First, it does not mandate any sort of ability to create a source distribution,
however not creating a source distribution is something that should be frowned
upon in the general case since it prevents downstream distributors like Debian
from being able to redistribute the code provided by that project.

PEP 516 punts on this decision since it's not strictly related to the concept
of building, however since distutils/setuptools currently handles both
situations you cannot replace distutils/setuptools for one part of that
operation without also replacing it for the other, so it is the opinion of this
PEP that any method of replacing setuptools for building wheels *must* also
include a method for replacing setuptools for building sdists. To do otherwise
would incentivize people to either not use the new system, or more likely, to
not produce source distributions at all.

Similarly, PEP 517 mostly punts on the issue as well, however it does define
somewhat what a sdist is. It is the opinion of this PEP, that the sdist
definition provided by PEP 517 conflates the idea of a sdist with the idea of
a source tree directory (such as with a VCS) however those two items are
similar, but different concepts.

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

