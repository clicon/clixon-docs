.. _clixon_install:

Installation
============
Clixon runs on Linux, `FreeBSD port <https://www.freshports.org/devel/clixon>`_. CPU architectures include x86_64, i686, ARM32.

You can also run Clixon in a docker container.

Ubuntu Linux
------------
This section can be used for many other Linux distributions.

Prerequisites
^^^^^^^^^^^^^
General prerequisites
"""""""""""""""""""""
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
"""""""""""""""""""""
Clixon's RESTCONF implementation currently supports two HTTP configurations:

  #. `FastCGI for reverse proxy`_
  #. `clixon native HTTP server`_

FastCGI for reverse proxy
"""""""""""""""""""""""""
To build clixon RESTCONF FastCGI support for nginx or similar reverse HTTP proxy::

  sudo apt-get install nginx libfcgi-dev

Then, when building clixon from source (see below), configure clixon with::

  configure --with-restconf=fcgi

Note that the libfcgi-dev package might not exist in Ubuntu 18 bionic or later, in which case need to `build fcgi from source`_),

Note that the 'fcgi' installation package might have a different name on other Linux distributions, such as "fcgi-dev" (alpine), "fcgi" (arch), "fcgi-devkit" (freebsd).

Clixon native HTTP server
"""""""""""""""""""""""""
To build the clixon native HTTP RESTCONF server::

  sudo apt-get install libevent-dev

It also requires openssl 1.1 API (not 1.0)

Then `build libevhtp from source`_.

Then, when building clixon from source (see below), configure clixon with(default)::

  configure --with-restconf=native

Build clixon from source
^^^^^^^^^^^^^^^^^^^^^^^^
Download clixon source code::

  git clone https://github.com/clicon/clixon.git
  
Configure Clixon using one of the following RESTCONF configurations::

  configure --with-restconf=native # clixon native HTTP server using libevhtp (default)
  configure --with-restconf=fcgi  # FastCGI support for reverse proxy, the default
                                  # when no '--with-restconf' option is specified
  configure --without-restconf

For more configure options see: `Configure options`_.

Build and install::
   
  make                      # Compile
  sudo make install         # Install libs, binaries, config-files and include-files
  sudo ldconfig

Building the example app and utils for running the tests
""""""""""""""""""""""""""""""""""""""""""""""""""""""""
To build and install the example app, from the top level clixon directory::

  make example
  cd example
  sudo make install

To build the utils for running the tests, from the top level clixon directory::

  sudo apt install libcurl4-openssl-dev
  cd util
  make
  sudo make install

FreeBSD
-------

FreeBSD has ports for both cligen and clixon available.
You can install them as binary packages, or you can build
them in a ports source tree locally.

If you install using binary packages or build from the
ports collection, the installation locations comply
with FreeBSD standards and you have some assurance
that the installed package is correct and functional.

The nginx setup for RESTCONF is altered - the system user
www is used, and the restconf daemon is placed in
/usr/local/sbin.

Binary package install
^^^^^^^^^^^^^^^^^^^^^^^^^
To install the pre-built binary package, use the FreeBSD pkg command:
::
   
  % pkg install clixon

This will install clixon and all the dependencies needed.

Build from source on FreeBSD
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
-------

Once installed, Clixon can be setup using systemd. The following shows an example with the backend and restconf daemons for the main example.
Install them as /etc/systemd/system/example.service and /etc/systemd/system/example_restconf.service, for example.

Systemd backend
^^^^^^^^^^^^^^^
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
^^^^^^^^^^^^^^^^
The Restconf service is installed at /etc/systemd/system/example_restconf.service, for example::
   
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

The restconf daemon can also be started using the clixon-lib process-control RPC. For more info, see :ref:`clixon_restconf`.

Docker
------
Clixon can run in a docker container.  As an example the `docker` directory has code for building and running the clixon test suite::

  cd docker/main
  make docker
  make test

The docker tests are run in the `Travis CI <https://travis-ci.org/github/clicon/clixon>`_
   
OpenWRT
-------

See [Clixon cross-compiler for openwrt](https://github.com/clicon/clixon-openwrt)

Vagrant
-------

Clixon uses vagrant in testing. For example to start a freebsd vagrant host, install Clixon and run the test suite, do  ::

  cd test/vagrant
  ./vagrant.sh freebsd/FreeBSD-12.1-STABLE

Other platforms include: ubuntu/bionic64 and generic/centos8

Build libevhtp from source
--------------------------
For RESTCONF using native http build evhtp from source as follows::

  sudo git clone https://github.com/clicon/clixon-libevhtp.git
  cd libevhtp
  ./configure --libdir=/usr/lib
  make

Note that evhtp requires openssl 1.1 API.

Note that you will likely need to add /usr/local/lib/libevhtp to your ld.so.conf configuration

Build fcgi from source
----------------------
For RESTCONF using fcgi build fcgi from source as follows::

  git clone https://github.com/FastCGI-Archives/fcgi2
  cd fcgi2
  ./autogen.sh
  ./configure --prefix=/usr
  make
  sudo make install


SSH subsystem
-------------

You can expose ``clixon_netconf`` as an SSH subsystem according to `RFC 6242`. Register the subsystem in ``/etc/sshd_config``::

	Subsystem netconf /usr/local/bin/clixon_netconf

and then invoke it from a client using::

	ssh -s <host> netconf


Configure options
-----------------

The Clixon `configure` script (generated by autoconf) includes several options apart from the standard ones.

These include (standard options are omitted)
  --enable-debug          Build with debug symbols, default: no
  --enable-optyangs       Install the yang files from clixon/yang/optional, required for running the example app and tests, default: no
  --enable-publish        Enable publish of notification streams using SSE and curl
  --with-cligen=dir       Use CLIGEN here
  --without-restconf      No RESTCONF
  --with-restconf=native  RESTCONF using native http with libevhtp. This is default
  --with-restconf=fcgi    RESTCONF using fcgi/ reverse proxy.
  --disable-nghttp2       Disable native http/2 using libnghttp2 (http/1 only)
  --disable-evhtp         Disable native http/1.1 using libevhtp (http/2 only)
  --with-configfile=FILE  set default path to config file
  --with-libxml2          use gnome/libxml2 regex engine
  --with-yang-installdir=DIR  Install Clixon yang files here (default: ${prefix}/share/clixon)
  --with-opt-yang-installdir=DIR  Install optional yang files here (default: ${prefix}/share/clixon)
  --without-sigaction     Disable sigaction logic (eg SA_RESTART mode)
  --enable-yang-patch     Enable RFC 8072 YANG patch (plain patch is always enabled)

There are also some variables that can be set, such as::

  ./configure LINKAGE=static ./configure         # Build static libraries
  ./configure CFLAGS="-O1 -Wall" INSTALLFLAGS="" # Use other CFLAGS

Note, you need to reconfigure and recompile from scratch if you want to build static libs
