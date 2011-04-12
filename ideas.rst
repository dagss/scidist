Scidist -- Nix for scientific computing
=======================================

Introduction
------------

Here's what I want for a scientific software distribution system (or
"Scidist", which I'll henceforth dub my utopian scientific
build/distribution system):

 - Non-root mode is completely necesarry (and may be the reason a lot
   of the current scientific distributions exist).

 - If I need to go back in time to compare with a run I did a year
   ago, it should be efficient. Jumping back and forth in time to
   compare effects (think benchmarks!) should be as quick and easy as
   jumping in git history.

 - Should be easy to switch between different versions of LAPACK/BLAS,
   or different Fortran compilers, or say "I want to be mostly on a
   stable branch but live on the trunk in cosmological
   software". Again, jumping between different configurations should
   be as easy as jumping between git branches.

 - Must encourage working on upstream projects. That is, very low
   treshold for patching upstream projects -- if I fix an issue by
   sending a patch of NumPy upstream, I want to start using the fix
   right away without having to "manually maintain my own package".

 - Should be easy to use to push *my own* software from my laptop to
   the cluster or to my peers. I guess this may be another instance of
   the point above.

 - Must support Python well, but not be overly focused on
   Python-specific solutions, and ideally useful outside the Python
   community as well. The software I use is mostly C/C++/Fortran,
   Python is just a nice shell around it.  Anything that encourages
   bundling LAPACK with the Python extension should put up a BIG
   warning flag.

And above all, must be *simple*. It is possible to solve a lot of
things by adding lots of complexity and features (think popular Linux
distros), but I don't think that path is viable for the scientific
community -- we just need better, simpler ideas.

Note that point 2-4 can be summarized as "must be more git-like". If
it is *trivial* and *efficient* to branch and merge an entire
scientific distribution (of course, you'd still track a stable and
tested upstream!), I feel 2-4 is almost solved, or at least it'll be a
very decent foundation to build a few utility scripts on.

Before I start rambling about Nix, here's some things I evaluated
before I fell for the Nix approach:

 - I was a fan of Gentoo Prefix for a while. Pro: Relies on the huge
   Gentoo community, Sage already ported, uses the "ports" approach of
   a single repo for all package definitions. Con: Simply too
   complicated, wants to force you to "live on the trunk". Also lacks
   several items on the above list.

 - Other solutions fail for much the same reasons our current approaches
   fall short, but here are some links:
   
   - http://0install.net/
   - http://lilypond.org/gub/
   - http://www.gnu.org/software/gsrc/


The core idea behind Nix
------------------------

Fortunately Eelco Dolstra comes, in my opinion, close to fixing this problem
in his PhD thesis:

http://www.st.ewi.tudelft.nl/~dolstra/pubs/phd-thesis.pdf

The result is now the Nix project,

http://nixos.org

It is fortunately very cleanly split into four components; the Nix
language and build system, nixpkgs, NixOS, and Hydra -- I'll focus on
Nix itself for now, touch upon nixpkgs eventually, and totally ignore
NixOS and Hydra (except mention right now that Hydra is a CI system
for Nix).

Read the docs for the details, but here's a quick tour:

Each package is described using a simple file (or small set of files)
containing a) tarball download link or pointer to VCS repo, b) some
metadata, d) commands to build it, d) any patches. By seperating the
main download from the packaging and keep many packages in the same
repo, one can work quite efficiently with revision control on a set of
packages. This part is similar to FreeBSD ports and Gentoo/Portage.

Each package is built and installed into its own directory, which
contains a hash from *all* the inputs to the package: The downloaded
tarball, the build script, the C compiler used, the Bash used, and so
on. So one would have::

    $NIXSTORE/as8f76234fasdgfas-myprogram-1.0/bin/myprog
    $NIXSTORE/as8f76234fasdgfas-myprogram-1.0/lib/libmyprog.so

and so on. The big point is that when upgrading to 1.1,
1.0 is still left in place, and 1.1 simply gets a new hash since
the downloaded tarball etc. hashes differently::

    $NIXSTORE/234asdfas1234-myprogram-1.1/bin/myprog
    $NIXSTORE/234asdfas1234-myprogram-1.1/lib/libmyprog.so

The idea is *not* to support using multiple versions concurrently in
the same program (which is a bad idea for any software). The big point
is that this facilitates very quickly switching between branches of
package sets, atomic upgrades, perfect rollbacks, and so on. Imagine
creating one branch of your software where everything is built with
ATLAS, another one where everything is built with GotoBLAS2, and
atomically and instantly switch between them. While not so
useful for production use, this can be *very* useful when debugging.

The "build artifact output directories" are called derivations in
Nix-speak. To describe a package (or, compute a derivation), one uses
a high-level functional programming language to describe every package
as a programmatic function. Fortunately, packages are still built
using Bash commands -- it is only processing package parameters (and
passing them on to Bash) that gets done in the functional language.
Of course, one composes a package using other functions, so one could
support installing an SPKG by writing a mkSagePkgDerivation function
that takes the spkg and does most of what you need, only leaving you
to connect configuration parameters with the corresponding Sage
environment variables.

Every package is purely a function of its inputs, and one is strict
about requiring all inputs that affect the build process to be passed
in. This is quite literal: If you want to build a C program, you need
to pass in which "stdenv" to use (meaning which bash, C compiler,
automake, etc.).  If you want to build a Fortran program, the Fortran
compiler must be passed in in addition. If you want to check out the
sources from Git instead of downloading a tarball, pass in the
"fetchGit" function that can be called to achieve this. Downloading a
tarball isn't magic, it is simply part of the functionality of "stdenv",
which you can trivially replace with your own.

So, again, if you want your setup to contain two versions of a
software, e.g., compiled with different Fortran compilers, you can
mostly call the same function twice while passing in different Fortran
compilers.  The results will hash differently and so be installed in
seperate locations.

You only specify build-time dependencies, which are simply other
packages passed as arguments to your package-building
functions. "Hard" run-time dependencies, where you link exactly one
derivation to a specific version of another, is automatically
registered simply by grepping through the built derivation for the
hashes of the built-time dependencies. So if your program links
with a shared library it will find the dependency. In some cases
one may need to emit a hash of a run-time dependency to a file
just to make sure it is picked up.

Finally, there's a garbage collector that removes unused derivations.

``nixpkgs`` builds on top of ``nix`` to actually build a distribution.
It does so by:

 - Having a tree of package files (e.g.,
   ``nixpkgs/pkgs/development/interpreters/python/2.6/default.nix``

 - Have some top-level files that ``import``-s the packages and strings
   them together. From ``nixpkgs/pkgs/top-level/all-packages.nix``::

       rsnapshot = callPackage ../tools/backup/rsnapshot {
           logger = inetutils;
       };

       inetutils = [...]

Finally, the ``nix-env`` command is the front-end to Nix. The
essential purpose is to create the "top-level" derivation that asks
for every other derivation that is needed (reference them of
``all-packages.nix``, which gives you closures that also references
build dependencies); this derivation builds a ``/local``-style tree of
symlinks into the other derivations, so that only one path needs to be
added to ``$PATH``.  ``nix-env`` swaps your current set of symlinks
when you switch "profile"; potentially to an earlier "generation" (a
rollback).


Needed work
-----------

My current hunch is that we should not use the nixpkgs distribution,
but build something new on top of core Nix. Reasons:

 - nixpkgs is probably more complicated than what we need

 - easier to get up and running by starting from scratch

 - we want to focus on a very small subset of packages and have
   individual release schedules

 - Nix is too difficult to use for non-experts, front-end should look
   a bit different

 - ``nix-env`` in some ways duplicates what a VCS can do for us

 - I must admit I really loathe the deep directory nesting of packages
   in ``nixpkgs`` and would like a flatter namespace, although this is
   not a good reason...

Of course, after having built something and got our direction
straight, one can decide that a merger with ``nixpkgs`` is in order.
This *will* however require lots of patches to ``nixpkgs`` scripts, it
is not possible to use ``nixpkgs`` out of the box.

As a general idea, I hope that as much of the "state" as possible
can be put into git repositories that are created for the user.
So the ``$SCIDIST`` distribution root directory may look like::

    $SCIDIST/conf # local git repository with configuration, 
                  # such as text file listing wanted packages
    $SCIDIST/scidist # git repository containing .nix expressions,
                     # upgrading a package involves pull-ing this
                     # one to a new version and then rebuilding
    $SCIDIST/local # symlink to a nix derivation, put this in $PATH
    $SCIDIST/store # Nix state, managed entirely by basic nix system
    $SCIDIST/bin # commands for package management -- may also link to a derivation

In release 0.1, the system works entirely by:

 - Modify configuration files in ``/conf`` in order to describe the system
   one wants (say, there's a text file of wanted packages)

 - Run ``bin/scidist build [confdir] [localsymlinktarget]`` to make
   sure that ``/local`` matches ``/conf``

 - A rollback to a previous configuration is done manually like this::

       (cd conf; git reset --hard HEAD^)
       bin/scidist build

 - Upgrading to the next release of scidist looks like this::

       (cd scidist; git branch prevrelease; git pull)
       bin/scidist build

   And if that broke things::

       (cd scidist; git checkout prevrelease)
       bin/scidist build

 - To use a Nix distribution there'll be a convenient command::

       $ bin/scidist env
       export PATH=/home/dagss/nix/local:$PATH
       # PYTHONPATH etc. as needed

   So, we have a canonical way of getting environment variables set up
   for a shell::

       $ source <(/path/to/my/scidist/bin/scidist env)

Building on this, we can start to add polish in the ``scidist`` command.


Ticket #1: Built distributions cannot be moved
''''''''''''''''''''''''''''''''''''''''''''''

This is the old question of -Wl,-rpath vs. LD_LIBRARY_PATH. The
current approach in ``nixpkgs`` is to hard-code RPATH for loading dynamic
libraries; meaning a set of Nix packages cannot easily be relocated
(and they are not made for it). Sage OTOH uses LD_LIBRARY_PATH,
however this is broken for other reasons (interfers with how dynamic
libraries are loaded *globally*, so that, e.g., it's impossible for me
to launch the local convenience program for sending something to the
printer in my institute from within a Sage shell).

Fortunately, in modern ``ld.so`` there's support for relative RPATHs
using ``$ORIGIN`` (type ``man ld.so``), which solves this problem if
one only builds packages diligently. As a quicker hack, the Nix team
has created the ``patchelf`` utility for rewriting RPATHs after the
fact; this could be used on older systems. Not sure about Mac or
Cygwin...

Ticket #2: Nix patches its gcc etc.
'''''''''''''''''''''''''''''''''''

This one initially made me put Nix aside for a year, but I was really
wrong: The patches are AFAIK (I didn't actually read them) only about
making sure ``/usr/include`` isn't looked up by default; patching the
toolchain is not necesarry for the Nix concept to work. There's many
"lightweight sandboxes" exploiting LD_PRELOAD available that can be
used instead -- not as secure, but a lot more reasonable for our
purposes. See #4.

Ticket #3: nix-env is too complicated
'''''''''''''''''''''''''''''''''''''

My current hunch is that we must make our own front-end tool instead
of ``nix-env``. Simply put, since our final goal is a bit different
from NixOS, we need a different UI.  See #4.

An idea is to utilize git instead of some of the features ``nix-env``
has. We don't want profiles; instead one can have a file of the
packages one wants to have installed under (a local) git repository,
and switching profiles then means having many branches and/or clones
of that repository.

Then, of course, a nice frontend to install and remove packages, but
always just as furnish above something very simple and as stateless as
possible.

Yes, this was vague, more investigation needed. And by all means,
let's just wrap ``nix-env`` if possible.

Ticket #4: nixpkgs wants its own toolchain
''''''''''''''''''''''''''''''''''''''''''

At least on the Linux platform, ``nixpkgs`` takes things to the extreme: In
order to build its own GCC, it even downloads a binary bootstrap
tarball (with Busybox!), to really ensure that things are the same
everywhere.

Of course, ``libc.so`` in the binary bootstrap tarball
segfaults on my Uni's computers...

For our purposes we *really* don't want to be this pedantic.  If
nothing else, having to build the toolchain is a major marketing
problem in getting Scidist accepted. Also there's a real problem: For
GUI components, it is likely a lot more reliable to link with the
system ``libX`` than to build our own.

However, it would be nice to not loose the integrity features;
if I build the same Scidist on two Ubuntu 10.04 computers, it'd
be nice if the hashes ended up the same, but they should end up
different on Ubuntu 10.10. At the same time, we must make sure
that users can upgrade their system without rebuilding everything.

Here's a way to do it:

 - We have a script that probes the system for the presence of a set
   of predeclared "base system" packages that we want Scidist to use
   from the host system. We'll use stdenv/gcc as the example.
   When run, the script finds the current path to gcc, and runs it with
   ``--version`` and records the output. The product of the script are
   NIX packages::
 
       $SCIDIST/host/gen1/stdenv.nix
       $SCIDIST/host/gen1/...

   Inside ``stdenv.nix``, there's code to "build" the package, which
   means simply wrapping the binaries present one on the system, while
   making sure (at run-time, somehow) that ``--version`` still produces
   the same output. Also, LD_PRELOAD tricks (see #2) can be played in
   the wrappers to make sandboxed builds, which helps with writing
   packages to make sure all dependencies are explicit.

   When run again, then if anything has changed (say, upgraded gcc through
   ``apt-get install``), a new "host system generation" is created::

       $SCIDIST/host/gen1/stdenv.nix
       $SCIDIST/host/gen2/stdenv.nix
       $SCIDIST/host/gen3/stdenv.nix

   Each containing a different string for their expected ``--version``
   output.

 - The top-level derivation (that is managed by the package manager
   front-end) must end up looking something like this (though probably
   generated from a domain-specific language)::

       [
           (cfitsio {stdenv=gen1.stdenv}),
           (python2.7 {stdenv=gen1.stdenv}),
           (healpix {stdenv=gen2.stdenv}),
       ]

   Here, ``cfitsio`` and ``python2.7`` were installed first, then the
   host ``gcc`` was upgraded (resulting in a new host-generation being
   created), and then  finally ``healpix`` was installed.

This also solves the problem with distributing binary packages. It
is OK to ship the "host-expressions" from one computer to another
as long as nothing triggers them to build. So, you could
have::

    [...]
    (cfitsio {stdenv=sageBuildFarm34.stdenv}),
    [...]

And if you want to trigger a local build of that package instead of
using an available binary, you change it and rebuild::

    [...]
    (cfitsio {stdenv=gen3.stdenv}),
    [...]



Ticket #5: Downloadable package/bootstrap scripts
'''''''''''''''''''''''''''''''''''''''''''''''''

Title says it all.

Ticket #6: Soft run-time dependencies
'''''''''''''''''''''''''''''''''''''

With Python packages, it is often the case that you depend on another
package at run-time only. Adding the package as a hard run-time
dependency is overkill; it would trigger a rebuild whenever the
dependency changes, but we know the result will be the same.

So perhaps we need something to say "if you install ``joblib``, you
want ``argparse`` as well, even if there is no explicit dependency in
the Nix expressions". This really classifies more as meta-information
about packages than a part of Nix expressions.

Perhaps Nix has this somewhere as well and I just didn't find it yet.

