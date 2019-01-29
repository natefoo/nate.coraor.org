---
layout:     post
title:      "Python Packaging: What's the Big Wheel?"
date:       2015-08-28 16:03:10 -0400
tags:       python packaging
---

# Background

Python modules have numerous binary packaging formats, but the one that's
emerged as the winner has been the [Wheel
format](https://wheel.readthedocs.org/). Wheel comes with a few improvements
over Python's previous binary packaging format, Eggs.  Unfortunately, it does
not solve the largest problem that [we](https://galaxyproject.org/) with eggs.
[Armin Ronacher explains this quite
well](http://lucumr.pocoo.org/2014/1/27/python-on-wheels/):

> There are a few problems with wheels however. One of the problems is that
> wheels inherit some of the problems that egg already had. For instance Linux
> binary distributions are still not an option for most people because of two
> basic problems: Python itself being compiled in different forms on Linux and
> modules being linked against different system libraries. The first problem is
> caused by Python 2 coming in two flavours that are both incompatible to each
> other: UCS2 Pythons and UCS4 Pythons. Depending on which mode Python is
> compiled with the ABI looks different. Presently the wheel format (from what
> I can tell) does not annotate for which Python unicode mode a library is
> linked. A separate problem is that Linux distributions are less compatible to
> each other as you would wish and concerns have been brought up that wheels
> compiled on one distribution will not work on others.

> The end effect of this is that you presently cannot upload binary wheels to
> PyPI on concerns of incompatibility with different setups.

[Here's why that's such a big problem for
Galaxy](http://dev.list.galaxyproject.org/Question-Regarding-fetch-eggs-amp-c-vs-pip-td4666564.html#a4666567):

> Galaxy has a huge (and ever-growing) list of dependent python modules with C
> extensions. If we did not prebuild and distribute eggs for these, the initial
> setup to get Galaxy running would be long and problematic. Some people who
> download Galaxy to develop tools may not even have compilers installed, let
> alone the multitude of -dev or -devel packages that aren't part of a default
> Debian or RHEL installation that would be required to build all of these
> packages from source. One of the things that I feel makes Galaxy so
> accessible is that you can start using it immediately after you clone the
> source. So that ability to clone and start and have it work as reliably (and
> quickly) as possible is a high priority.

Mac OS X wheels are allowed in PyPI, and that's great as a lot of development
happens on OS X, but almost all production Galaxy servers run on Linux, and for
this, we still have no solution.

[For a long time in
Galaxy](https://mail.python.org/pipermail/distutils-sig/2010-January/015345.html),
we've monkeypatched setuptools' platform detection to add the necessary
UCS2/UCS4 detection and left it at that, hoping a better solution would come
along. When Wheels emerged as the new packaging format but did not solve these
problems, I decided to put in the work to solve them.

# Preliminary Work and Discussion

This work began as some discussions with
[PyPA](http://pypaio.readthedocs.org/en/latest/) people on IRC but after making
[a quick
implementation](https://bitbucket.org/pypa/wheel/pull-requests/54/soabi-2x-platform-os-distro-support-for/diff)
that turned out to be too naive, [I opened a
discussion](http://code.activestate.com/lists/python-distutils-sig/26009/) with
the
[Distutils-SIG](https://www.python.org/community/sigs/current/distutils-sig/)
to figure out where to go with it.

## A Good Tangent

This discussion quickly grew in scope to discuss another failing of wheel, in
that it does not provide a mechanism for dealing with *externally linked
dependencies*. My canonical example here is the
[psycopg2](http://initd.org/psycopg/) Python module, the most popular DB-API
implementation for PostgreSQL. It depends on the libpq (PostgreSQL's C API)
shared library being available at runtime. So even if we do build and host a
psycopg2 wheel, anyone wanting to use it has to install whatever package
provides (or compile from source) `libpq.so` on their system and ensure it's on
the runtime linker path, or else psycopg2 will fail to import. Wheels provide
no mechanism to inform what system libraries, packages, or anything else that's
external what they depend on. A variety of solutions were proposed, check out
the thread if you'd like to see some of the very good thinking on this subject.

Of course, this is a *general* wheel problem, not just wheels on Linux - so if
a psycopg2 wheel for OS X was in PyPI today, and you installed it but didn't
have libpq installed, you'd run into the same problem you'd have on Linux.

This problem interests me (for example, to solve it for Galaxy's eggs, we
compile a static version of libpq and link that into [the
egg](http://eggs.galaxyproject.org/psycopg2/)), but it isn't the problem I'm
trying to solve (and indeed, if you follow the discussion far enough, it
eventually finds [its own
thread](http://code.activestate.com/lists/python-distutils-sig/26056/)).

## The State of Things

Solving the external dependency problem is good and worth doing, but for the
moment I want to improve these problems in the current state:

1. Linux wheels are not allowed in PyPI.
2. Compiling a package from a source distribution requires installation of a
   variety of development tools and header packages.

Figuring out the right mix of packages may require multiple iterations of
attempted package installs, with errors that may be fairly confusing for people
not used to compiling software:

```
# pip install psycopg2
Collecting psycopg2
  Using cached psycopg2-2.6.1.tar.gz
Installing collected packages: psycopg2
  Running setup.py install for psycopg2
    Complete output from command /usr/bin/python -c "import setuptools, tokenize;__file__='/tmp/pip-build-C8JPB7/psycopg2/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /tmp/pip-UIFRep-record/install-record.txt --single-version-externally-managed --compile:
    running install
    running build
      ...
    running build_ext
    building 'psycopg2._psycopg' extension
    creating build/temp.linux-x86_64-2.7
    creating build/temp.linux-x86_64-2.7/psycopg
    x86_64-linux-gnu-gcc -pthread -DNDEBUG -g -fwrapv -O2 -Wall -Wstrict-prototypes -fno-strict-aliasing -D_FORTIFY_SOURCE=2 -g -fstack-protector-strong -Wformat -Werror=format-security -fPIC -DPSYCOPG_DEFAULT_PYDATETIME=1 -DPSYCOPG_VERSION="2.6.1 (dt dec pq3 ext lo64)" -DPG_VERSION_HEX=0x090403 -DHAVE_LO64=1 -I/usr/include/python2.7 -I. -I/usr/include/postgresql -I/usr/include/postgresql/9.4/server -c psycopg/psycopgmodule.c -o build/temp.linux-x86_64-2.7/psycopg/psycopgmodule.o -Wdeclaration-after-statement
    In file included from psycopg/psycopgmodule.c:27:0:
    ./psycopg/psycopg.h:30:20: fatal error: Python.h: No such file or directory
     #include <Python.h>
                        ^
    compilation terminated.
    error: command 'x86_64-linux-gnu-gcc' failed with exit status 1
    
    ----------------------------------------
Command "/usr/bin/python -c "import setuptools, tokenize;__file__='/tmp/pip-build-C8JPB7/psycopg2/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /tmp/pip-UIFRep-record/install-record.txt --single-version-externally-managed --compile" failed with error code 1 in /tmp/pip-build-C8JPB7/psycopg2
```

Granted, the error you will receive when you install a wheel without the
external dependencies installed could also be confusing for people new to *NIX:

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/local/lib/python2.7/dist-packages/psycopg2/__init__.py", line 50, in <module>
    from psycopg2._psycopg import BINARY, NUMBER, STRING, DATETIME, ROWID
ImportError: libpq.so.5: cannot open shared object file: No such file or directory
```

But I argue that that's at least better than the previous situation - plus,
package authors can always catch that `ImportError` and provide an error
message with a bit more guidance. But the big point is that you only need to
have one package installed, no compilers, and no -dev packages. If you are
installing psycopg2 there's a decent likelihood you have the `libpq` package
(on Debian) installed, but not as much of a likelihood you have `libpq-dev`
installed.

And one minor point, all of this is potentially on a production box. These
days, especially with Docker, I don't even like to install compilers on
production boxes if I don't have to.

## Direction

I ultimately declined my own [pull request on
wheel](https://bitbucket.org/pypa/wheel/pull-requests/54/soabi-2x-platform-os-distro-support-for/diff)
because the Linux platform detection was flawed. As a result, there's a new
library in my versions of pip and wheel that attempts to detect the current
linux distribution in a variety of ways, but preferably via `/etc/os-release`.

As it turns out, the use of `/etc/os-release` for platform detection had
[already been hashed out on Distutils-SIG a while
ago](http://code.activestate.com/lists/python-distutils-sig/24584/), but I
didn't discover that thread until today. Thankfully, it looks like my
implementation should go along with the proposal discussed there (my
implementation was based on Nick Coghlan's suggestion in my later thread, so
it's no surprise).

Eventually I'd like to have it do illumos distribution detection as well, but
that's a much lower priority.

# Changes

## ABI Tags

The first problem was that of Python ABI compatibility with differing Unicode
widths. Thankfully, [PEP 425](https://www.python.org/dev/peps/pep-0425/)
provides the specification for how UCS and other ABI incompatibilities should
be tagged in built distributions such as wheel. Unfortunately, the [ABI
tag](https://www.python.org/dev/peps/pep-0425/#abi-tag) was not implemented for
Python 2 (see related issues,
[pypi-metadata-format#25](https://bitbucket.org/pypa/pypi-metadata-formats/issues/25),
[wheel#101](https://bitbucket.org/pypa/wheel/issues/101/python-wheels-need-ucs-tag-on-2x),
and
[wheel#63](https://bitbucket.org/pypa/wheel/issues/63/backport-soabi-to-python-2)).
However, implementing this tag in the pip and wheel packages was relatively
easy.

## Platform Tags

If the platform tag doesn't start with Linux, there's no changes here. On
Linux, I append the Linux distribution and distribution version number to the
platform portion of the tag. So where the old tag would've been `linux_x86_64`
on a 64-bit x86 CPU, the new tag will read like `linux_x86_64_ubuntu_14_04` or
`linux_x86_64_debian_8`.

Thankfully, the PEP 425 reference implementation was written with tag
specificity in mind, so in fact when comparing tags, a list of tags, from most
to least specific, is checked for compatibility. In the case of the platform
tag:

- When **installing** wheels, the wheel with the most-specific matching tag
  will be installed first. A list of platform tags might look like:
  - `linux_x86_64_rhel_6_5`
  - `linux_x86_64_rhel_6`
  - `linux_x86_64`
  - `any`
- When **building** wheels, the tag for the distribution's *major* version will
  be preferred, for example, `linux_x86_64_rhel_6`. It is possible to build a
  wheel for the specific major/minor version using the `--plat-name` argument
  to `bdist_wheel` if the wheel builder has reason to believe it would not be
  compatible with older minor releases.

See related issue [pypi-metadata-formats#15](https://bitbucket.org/pypa/pypi-metadata-formats/issues/15).

## binary-compatibility.cfg

As you may have noticed, it's now possible to build wheels that are targeted
for RHEL, but what if you don't have a RHEL system on which to build, but you
do have a CentOS system. Or what if someone built RHEL wheels and you want to
install them on your CentOS system. Of course, because they are ABI compatible,
it shouldn't be a problem to install one's wheels on the other. As a solution
for this, Nick Coghlan proposed a file, `binary-compatibility.cfg`, in JSON
format, to be found in either `/etc/python` or the root of a currently active
virtualenv, which defines what platforms the current platform is compatible
with. So for example, to build and install RHEL 7 wheels on my CentOS 7 host,
I'd write a `binary-compatibility.cfg` like so:

```
{
  "linux_x86_64_centos_7": {
    "build": "linux_x86_64_rhel_7",
    "install": ["linux_x86_64_rhel_7"]
  }
}
```

Ideally, `/etc/python/binary-compatibility.cfg` would be maintained by
distribution vendors.

## "Generic" wheels

As mentioned above, it is possible to override the platform tag using the
`--plat-name` argument to `bdist_wheel`. Because the `linux_x86_64` platform
remains in the list of platforms for which a wheel will be searched, a wheel
builder can choose to build for that platform if it has no non-standard
external dependencies.

Anyone execising this option should take care that:

1. No non-standard libraries (anything other than libc, libm, etc.) are linked
   by *any* dynamic object in the built distribution.
2. The wheel is built on a suitably old platform that it will run on any Linux
   release within the last ~5 years. Due to symbol versioning in the GNU C
   Library (glibc), modules built on a newer glibc cannot be used with an older
   glibc.

Ideally, both of these checks could be automated, and perhaps the decision
could be made to utilize the "generic" tag automatically if both 1 and 2 are
true, except that "suitably old" glibc versions would need to be a policy
decision and updated on a semi-regular basis.

## ABI stability

The platform detection code currently makes a rudimentary effort to determine
whether a given Linux distribution and version can be considered to have a
stable ABI or not. This information is not used, but may be useful for PyPI to
decide whether or not to accept wheels for a given platform tag.

# Supporting work

## docker-build

I don't have time to write much about it here, but this work was tested using
[Galaxy docker-build](https://github.com/galaxyproject/docker-build/), which
received quite a bit of work over the process itself. docker-build is a set of
scripts and YAML that we use to build Galaxy tool dependencies (typically
scientific software) in Docker, for packaging in the Galaxy Tool Shed.

I extended docker-build to build wheels for as many distributions as had
official images in Docker Hub, plus a few more, for both the UCS-2, and UCS-4
variants of Python 2.6 and 2.7. Note to self, I need to push updated *-wheel
images to Docker Hub.

# Conclusion

I hope this is accepted upstream, although I haven't seemed to attract much
attention with my latest posts to Distutils-SIG. I may try opening PRs shortly
and see if that spawns any discussion.

The modifications to wheel are in [my wheel repository on
Bitbucket](https://bitbucket.org/natefoo/wheel). The modifications to pip are
in [the linux-wheels branch of my pip repository on
GitHub](https://github.com/natefoo/pip/tree/linux-wheels).

You can see the wheels that I've built using docker-build [on our wheel
index](https://wheels.galaxyproject.org/).
