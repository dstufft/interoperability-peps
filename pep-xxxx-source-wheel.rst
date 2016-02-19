PEP: XXX
Title: Source Wheel Format
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

This PEP defines a "Source Wheel" distribution. A "Source Wheel" is a ZIP
format archive with a specially formatted file name and the .swhl extension.
This is a successor to the sdist format produced by distutils and setuptools.


Rationale
=========

The current source distribution format in use is extremely underspecified. It
is essentially specified as "whatever distutils" produces. Over time a number
of projects (most famously, setuptools) have extended this format, further
complicating the format's already ambigious definition. This level of ambiguity
extends out to the point that the only file you can *really* depend on existing
is ``setup.py`` at the root. This means that to get any any other information
about the source distribution, this file must be executed.

By defining a new standard implementation of what a source distribution is,
this PEP hopes to help break the cycle of dependency on distutils/setuptools
and to remove the ambiguity.


Details
=======

.. note::

    I need to actually define what this looks like. However in very short terms
    it will be something like::

        DIST-INFO/
            METADATA
            ...
        SRC/
            ...
        pypa.toml

    Where the DIST-INFO directory is very similar to what is already in use by
    Wheel files, except a restricted set of metadata is acceptable due to the
    nature of a source distribution. The SRC dir is just a dumping grounds for
    all of the files for the actual distribution itself. They are moved down a
    level to avoid any collisions with any other files. The pypa.toml file is
    from the other PEP, build pipelines.

    It's important to note here that the metadata in DIST-INFO is a *subset* of
    what is available to a wheel. We need to sort out exactly what subset is
    acceptable, but once we figure that out we should encode that here.

    It's possible that ``pypa.toml`` could instead be something like
    ``pypa.json``. The desire to use ``TOML`` when this file is human editable
    is no longer a concern here, and it is potentionally nice to allow projects
    to consume source wheels without needing to have a ``TOML`` implementation
    available. Of course this would make it somewhat more complicated for
    tooling because they would not be able to rely on their being a single
    represenatation of the data on disk, but it's not really much harder to
    have an if statement for ``toml.loads`` vs ``json.loads``.


Frequently Asked Questions
==========================

Why a New Extension?
--------------------

One of the nice side benefits of the way that Wheel was rolled out, was that
since it used an entirely new extension (instead of say, changing .egg) was
that tools which didn't understand Wheel files just completely ignored them.
This made it easy to make new decisions in that format without having to worry
about how every tool that currently exists is going to react to such a file.

Likewise, making a new extension for a replacement to sdist means that we can
ignore how older tools are going to handle these new formats. In particular
this means that there will be no need for a ``setup.py`` shim inside of a
source wheel. Individual projects will be able to decide how much they care
about legacy support for versions of installers that don't support source
wheels and publish a "legacy sdist" which includes a ``setup.py`` shim or they
can decide to only support tooling that understands it and forgo needing to
include that ``setup.py`` shim.


Why not add XYZ Feature?
------------------------

The goal here with this version of Source Wheels is to get the minimal amount
of features to match what we need to replace the legacy sdists for the 95% use
case. Thus we're not going to look at *adding* new features or drastically
altering the formats that are already in wide use, which leaves out things
present in other PEPs like switching to a JSON encoding of the metadata, or
building extensible metadata. These items are considered to be explicitly out
of scope for this PEP and can be addressed in a future PEP.



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

