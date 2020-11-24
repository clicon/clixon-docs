.. _clixon_overview:

Overview
========

Clixon is a management framework that can be used by networking
devices and other computer systems.  Clixon provides a datastore, CLI,
NETCONF and RESTCONF interfaces all defined by YANG.

Clixon links:

  - `Source code at github <http://www.github.com/clicon/clixon>`_.
  - `Project <http://www.clicon.org>`_.
  - `Docs <https://clixon-docs.readthedocs.io/en/latest/>`_.

Most of the projects using Clixon are for networking devices. But Clixon
can be used for other YANG-based systems as well due to a modular and
pluggable architecture.

Clixon has a transaction mechanism that ensures configuration
operations are atomic. It also features a generated interactive
command-line interface using `CLIgen <http://www.cligen.se>`_.

The goal of Clixon is to provide a useful, production-grade, scalable
and free YANG based configuration tool.

Clixon is open-source and dual licensed. Either Apache License, Version 2.0 or GNU
General Public License Version 2.


System Architecture
-------------------

::
   
                  +------------------------------------------+
                  |  Frontends:          +------------+      |
                  |  +----------+        | configfile |      |
                  |  |   cli    |--+     +------------+      |
                  |  +----------+   \ +----------+---------+ |
                  |  +----------+    \| backend  | plugins | |
      User  <-->  |  | restconf |---- | daemon   |         | |  <--> Underlying
                  |  +----------+    /+----------+---------+ |       System
                  |  +----------+   /    +------------+      |
	          |  | netconf  |--+     | datastores |      |
		  |  +----------+        +------------+      |
                  +------------------------------------------+
		 
The core of the Clixon architecture is a backend daemon with
configuration datastores and a set of internal clients: cli, restconf
and netconf.

The clients provide frontend interfaces to users, including an
interactive CLI, RESTCONF over HTTP, and XML NETCONF over SSH.
Internally, the clients and backend communicate via NETCONF over a
UNIX socket.

The backend manages a configuration datastore and implements a
transaction mechanism for configuration operations (eg, create, read,
update, delete) . The datastore supports candidate, running and
startup configurations.

When you adapt Clixon to your system, you typically start with a set
of YANG specifications that you want implemented. You then write
backend plugins that interact with the underlying system. The plugins
are written in C using the Clixon API and a set of plugin
callbacks. The main callback is a transaction callback, where you
specify how configuration changes are made to your system.

You can also design an interactive CLI using `CLIgen
<http://www.cligen.se>`_, where you specify the CLI commands and write
CLI plugins.  You will have to write CLI rules, but Clixon can
generate the configuration part of the CLI, including set, delete, show
commands for a specific syntax.
   
Notifications are supported both for CLI, netconf and restconf clients, sometimes referred to as "streams".

Platforms
---------

Clixon supports GNU/Linux, FreeBSD and Docker. MacOS may work. Linux
platforms include 32 and 64 bits Ubuntu, Alpine, Raspian, etc.

Standards
---------
Clixon supports standards including YANG, NETCONF, RESTCONF, XML and XPath. See :ref:`clixon_standards` for more details.

How to get Clixon
-----------------
Get the Clixon source code from `Github <http://github.com/clicon/clixon>`_::

   git clone git@github.com:clicon/clixon.git

Support
-------
For support issues use the `Clixon slack channel <https://clixondev.slack.com>`_. Please ask for access.

Bug reports
-----------
Report bugs via `Github issues <https://github.com/clicon/clixon/issues>`_

Reference docs
--------------
The user-manual is this document.
For reference documentation of the C-code, Doxygen is used. To build the reference doumentation you need to check out the source code, and type ``make doc``, eg::

  git clone git@github.com:clicon/clixon.git
  cd clixon
  ./configure
  make doc

direct your browser to::

  file:///<your home path>/clixon/doc/html/index.html
  


