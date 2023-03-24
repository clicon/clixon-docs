.. _clixon_install:
.. sectnum::
   :start: 2
   :depth: 3

************
Installation
************

Ubuntu Linux
============

This section describes how to build Clixon from source on Ubuntu
Linux. You can use this as a base for other platforms as well since
many steps (such as prereqs) are similar.

Further, the `vagrant`_ scripts show how to build for some other Linux variants, such as Centos.

Prerequisites
-------------

General prerequisites
^^^^^^^^^^^^^^^^^^^^^
Install packages::

  sudo apt-get install flex bison

Install and build CLIgen::

  git clone https://github.com/clicon/cligen.git
  cd cligen;
  configure
  make;
  sudo make install

Add a clixon user and group, using useradd and usermod::
   
  sudo useradd -M -U clixon
  sudo usermod -a -G clixon <youruser>  # Remember to re-log in for this to take effect
  sudo usermod -a -G clixon www-data # Only if RESTCONF
  
If you do not require RESTCONF, then continue with `Build clixon from source`_.

RESTCONF HTTP Support
^^^^^^^^^^^^^^^^^^^^^
The RESTCONF implementation supports two HTTP configurations:

  #. `Clixon native HTTP server`_
  #. `FastCGI for reverse proxy`_

Clixon native HTTP server
^^^^^^^^^^^^^^^^^^^^^^^^^
Native http server requires openssl 1.1 or later.

Install TLS::

  sudo apt-get install libssl-dev

Thereafter configure using default options::

    configure

FastCGI for reverse proxy
^^^^^^^^^^^^^^^^^^^^^^^^^
FastCGI requires  support for Nginx or similar reverse HTTP proxy::

  sudo apt-get install nginx libfcgi-dev

Then, when building clixon from source (see below), configure clixon with::

  configure --with-restconf=fcgi

Note that the libfcgi-dev package might not exist in Ubuntu 18 bionic or later, in which case need to build `fcgi`_ from source.

Note also that the 'fcgi' installation package might have a different name on other Linux distributions, such as "fcgi-dev" (alpine), "fcgi" (arch), "fcgi-devkit" (freebsd).

Build Clixon from source
------------------------
Download clixon source code::

  git clone https://github.com/clicon/clixon.git
  
Configure Clixon using one of the following RESTCONF configurations::

  configure --with-restconf=native # clixon native HTTP server
  configure --with-restconf=fcgi   # FastCGI support for reverse proxy, the default
                                   # when no '--with-restconf' option is specified
  configure --without-restconf     # Do not build restconf

For more configure options see: `Configure options`_.

Build and install::
   
  make                      # Compile
  sudo make install         # Install libs, binaries, config-files and include-files
  sudo ldconfig             # To link new dynamic libraries

Building the example and utils
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
To build and install the example app, from the top level clixon directory::

  make example
  cd example
  sudo make install

To build the utils for running the tests, from the top level clixon directory::

  sudo apt install libcurl4-openssl-dev
  cd util
  make
  sudo make install

See also the :ref:`Quickstart section<clixon_quickstart>` section for building a complete *hello world* example.

  
FreeBSD
=======

FreeBSD has ports for both cligen and clixon available.
You can install them as binary packages, or you can build
them in a ports source tree locally.

If you install using binary packages or build from the
ports collection, the installation locations comply
with FreeBSD standards and you have some assurance
that the installed package is correct and functional.

The Nginx setup for RESTCONF is altered - the system user
www is used, and the restconf daemon is placed in
/usr/local/sbin.

Binary package install
----------------------
To install the pre-built binary package, use the FreeBSD pkg command:
::
   
  % pkg install clixon

This will install clixon and all the dependencies needed.

Build from source on FreeBSD
----------------------------
If you prefer you can also build clixon from the
`FreeBSD ports collection <https://www.freebsd.org/doc/handbook/ports-using.html>`_

Once you have the Ports Collection installed, you build clixon like this
::

   % cd /usr/ports/devel/clixon
   % make && make install

One issue with using the Ports Collection is that it may
not install the latest version from GitHub. The port is
generally updated soon after an official release, but there
is still a lag between it and the master branch. The maintainer
for the port tries to assure that the master branch will
compile always, but no FreeBSD specific functional testing
is done.

Systemd
=======
Once installed, Clixon can be setup using systemd. The following shows an example with the backend and restconf daemons from the main example.
Install them as /etc/systemd/system/example.service and /etc/systemd/system/example_restconf.service, for example.

Systemd backend
---------------
The backend service is installed at /etc/systemd/system/example.service, for example. Note that in this example, the backend installation requires the restconf service, which is not necessary.
::

   [Unit]
   Description=Starts and stops a clixon example service on this system
   Wants=example_restconf.service
   [Service]
   Type=forking
   User=root
   RestartSec=60
   Restart=on-failure
   ExecStart=/usr/local/sbin/clixon_backend -s running -f /usr/local/etc/example.xml
   [Install]
   WantedBy=multi-user.target


Systemd restconf
----------------
The Restconf service can be installed at, for example, /etc/systemd/system/example_restconf.service::
   
   [Unit]
   Description=Starts and stops an example clixon restconf service on this system
   Wants=example.service
   After=example.service
   [Service]
   Type=simple
   User=root
   Restart=on-failure
   ExecStart=/usr/local/sbin/clixon_restconf -f /usr/local/etc/example.xml
   [Install]
   WantedBy=multi-user.target

The restconf daemon can also be started internally using the clixon-lib process-control RPC. For more info, see :ref:`Restconf section<clixon_restconf>`.

Docker
======
Clixon can run in a docker container.  As an example the `docker` directory has boilerplate code for building Clixon in a container::

  cd docker/base
  make docker

For complete examples see:

* `Hello world <https://github.com/clicon/clixon-examples/tree/master/hello/docker>`_
* `Clixon CI test container <https://github.com/clicon/clixon/tree/master/docker/main>`_
* `Openconfig <https://github.com/clicon/clixon-examples/tree/master/openconfig/docker>`_

   
Vagrant
=======
Clixon uses vagrant in testing. For example to start a Freebsd vagrant host, install Clixon and run the test suite, do  ::

  cd test/vagrant
  ./vagrant.sh generic/freebsd12

Other platforms include: ubuntu/bionic64 and generic/centos8. To look at how Clixon is installed natively on those platforms please look in the build scripts under test/vagrant/.

OpenWRT
=======
See `Clixon cross-compiler for Openwrt <https://github.com/clicon/clixon-openwrt>`_


Prereqs from source
===================

FCGI
----
For RESTCONF using fcgi build fcgi from source as follows::

  git clone https://github.com/FastCGI-Archives/fcgi2
  cd fcgi2
  ./autogen.sh
  ./configure --prefix=/usr
  make
  sudo make install

SSH subsystem
=============
You can expose ``clixon_netconf`` as an SSH subsystem according to `RFC 6242`. Register the subsystem in ``/etc/sshd_config``::

	Subsystem netconf /usr/local/bin/clixon_netconf

and then invoke it from a client using::

	ssh -s <host> netconf

Configure options
=================
The Clixon `configure` script (generated by autoconf) includes several options apart from the standard ones.

These include (standard options are omitted)
  --enable-debug             Build with debug symbols, default: no
  --enable-yang-patch         Enable RFC 8072 YANG patch (plain patch is always enabled)
  --enable-publish           Enable publish of notification streams using SSE and curl
  --disable-http1            Disable native http/1.1 (ie http/2 only)
  --disable-nghttp2          Disable native http/2 using libnghttp2 (ie http/1 only)
  --with-cligen=dir          Use CLIGEN here
  --with-restconf=native     RESTCONF using native http. (DEFAULT)
  --with-restconf=fcgi       RESTCONF using fcgi/ reverse proxy.
  --without-restconf         No RESTCONF
  --with-configfile=FILE     Set default path to config file
  --with-libxml2             Use gnome/libxml2 regex engine
  --without-sigaction        Disable sigaction logic (some platforms do not support SA_RESTART mode)
  --with-yang-installdir=DIR  Install Clixon yang files here (default: ${prefix}/share/clixon)
  --with-yang-standard-dir=DIR  Location of standard IETF/IEEE YANG specs for tests and example
                                (default: $prefix/share/yang/standard). You can retrieve the standard files at https://github.com/YangModels/yang

There are also some variables that can be set, such as::

  ./configure LINKAGE=static                     # Build static libraries
  ./configure CFLAGS="-O1 -Wall" INSTALLFLAGS="" # Use other CFLAGS

Note, you need to reconfigure and recompile from scratch if you want to build static libs

macOS
=====

Clixon can be built on macOS, however not all tests will pass at this
moment and there might be pieces which will not run properly.

A few packages must be installed using for example HomeBrew::

  brew install openssl nghttp2


Since we install a few libraries from HomeBrew we might want to set C
and library paths::

    $ export LIBRARY_PATH=$LIBRARY_PATH:/opt/homebrew/opt/openssl/lib
    $ export C_INCLUDE_PATH=/opt/homebrew/opt/openssl/include/


Then Cligen and Clixon can be built as normal. Since Clixon will
install things in "/usr/local/sbin/" you might want to add this to
PATH. Either temporarily using::

  export PATH=$PATH:/usr/local/sbin/


Or permanently by adding the above to .bash_profile or similar.

Since macOS don't use systemd or similar you'll have to start and stop
clixon_backend etc manually.
