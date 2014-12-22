---
layout: post
title:  "pkg-config, pkgconf, OmniOS, and uulm.mawi"
date:   2014-12-21 13:31:02
categories: omnios development
tags: pkgconf pkg-config
---
(Thank you for reading this, my introductory post)

I have been running OmniOS for just under a year as my home media and backup
server in order to benefit from the reliability and advanced features of ZFS.
Specifically, I wanted ZFS's robustness to serve as bedrock for Apple's Time
Machine and disk images, whose fragility had earlier almost resulted in my
losing hundreds of gigabytes of backup and had certainly used up days in trying
to recover. The Time Machine fragility still exists, but ZFS allows me to
recover in seconds by rolling back to a good snapshot.

OmniOS and ZFS have been essentially perfect.

Now I wanted to develop on OmniOS, so I downloaded my first source tarball
package and set to trying to build it. Here is where I ran into my first
blocker.

Running the source code autogen.sh resulted in an error message about missing
pkg-config, which I recognized from Mac OS X development. Searching package
sources on OmniOS resulted in one hit, though for a package named "pkgconf" not
"pkg-config":

{% highlight bash %}
$ pkg search pkg-config 
INDEX        ACTION  VALUE                                          PACKAGE
basename     link    usr/local/bin/pkg-config                       pkg:/library/pkgconf@0.9.3-0.151012
pkg.summary  set     pkgconf is a modern replacement for pkg-config pkg:/library/pkgconf@0.9.3-0.151012
{% endhighlight %}

The summary describes it as a "modern replacement for pkg-config", and from the
project site, I learned that pkgconf is "an API-driven pkg-config replacement"
which the authors assert is "better for distros"
([pkgconf on Github](https://github.com/pkgconf/pkgconf)). I suppose that's why
OmniTI chose it.

Now one more bit of background. My OmniOS system is configured through accident
with two package publishers:

{% highlight bash %}
$ pkg publisher
PUBLISHER   TYPE     STATUS   URI
omnios      origin   online   http://pkg.omniti.com/omnios/r151012/
uulm.mawi   origin   online   http://scott.mathematik.uni-ulm.de/release/
{% endhighlight %}

The accident occurred in trying to run napp-it.org's instructions for installing
netatalk (one of the steps on [nappit-org's
omnios_en instructions](http://napp-it.org/downloads/omnios_en.html)) but
finding the uulm.mawi repository down for service. The napp-it script normally
adds uulm.mawi transiently and removes it when finished; but when the repository
was down, the script died and left the repository active.

I was aware of this problem, but I also realized that in order to take advantage
of uulm.mawi package updates, I would need an active repository. So I was happy
to leave it as is. Indeed uulm.mawi is the source of the pkgconf package above,
not OmniTI.

Re-running my source code's autogen.sh, however, did not show any difference
from before having installed pkgconf:

{% highlight bash %}
configure.ac:292: error: possibly undefined macro: AC_MSG_ERROR
configure.ac:310: error: possibly undefined macro: AC_DEFINE
configure.ac:1052: error: possibly undefined macro: PKG_CHECK_MODULES
configure:57875: error: possibly undefined macro: PKG_CHECK_EXISTS
{% endhighlight %}

In seeing the first error related to AC_MSG_ERROR (with "AC" meaning
"autoconf"), I went on a bit of a long, wild goose chase in thinking my autoconf
setup was broken. I certainly learned a lot about autoconf in trying to debug
its operation, but I could find no problem (and indeed there wasn't).

Re-examing the errors above, it is actually the last lines that are most
significant. They reveal that pkgconf is not operating entirely correctly. I
needed some clue about the pkgconf package, and I found this useful command:

{% highlight bash %}
$ pkg contents pkgconf
PATH
usr/local
usr/local/bin
usr/local/bin/amd64
usr/local/bin/amd64/pkgconf
usr/local/bin/i386
usr/local/bin/i386/pkgconf
usr/local/bin/pkg-config
usr/local/bin/pkgconf
usr/local/share
usr/local/share/aclocal
usr/local/share/aclocal/pkg.m4
{% endhighlight %}

Examining pkg.m4 revealed the missing macros, PKG_CHECK_MODULES and
PKG_CHECK_EXISTS. Moreover, their location in "aclocal" steered me to the right
culprit, the command-line tool also named "aclocal".

aclocal's man page has this useful switch:

{% highlight bash %}
--print-ac-dir
      print name of directory holding system-wide third-party m4
      files, then exit
{% endhighlight %}

So I ran that:

{% highlight bash %}
$ aclocal --print-ac-dir
/usr/share/aclocal
{% endhighlight %}

Aha! aclocal is looking in /usr/share/aclocal and not /usr/local/share/aclocal
where pkg.m4 lives. My problem was due to a prefix incompatibility between the
OmniTI repository, which provided the automake package (with a /usr prefix); and
uulm.mawi, which provided pkgconf (and which states emphatically that "all
packages use /usr/local as prefix").

For this single file, my chosen action was to link pkg.m4 into
/usr/share/aclocal:

{% highlight bash %}
sudo ln -s /usr/local/share/aclocal/pkg.m4 /usr/share/aclocal/pkg.m4
{% endhighlight %}

This fixed my build, and my development project can continue.
