.. _clixon_pyapi:
.. sectnum::
   :start: 21
   :depth: 3

**********
Python API
**********

This section documents the Clixon Python API. The Python API is used
to implement service logic for Clixon. The Python API is a wrapper
around the Clixon C library and is used to manipulate the
configuration tree.

Overview
========

The Clixon Python API consist of two parts:

- The server (clixon_server.py).
- Service modules.

The server listens for events from the Clixon backend and run the
modules when needed. All service logic are implemented in the modules
and are described in detail below.


Internals
=========

The Python API connect to Clixons internal socket and communicate over
NETCONF. When started it will enable notifications for service commits
and controller transactions.

Whenever a commit is done the Clixon Controller issue a notify on the
backend socket which is catched by the Python API.

When the Python API receives a service commit it runs all the service
modules which can manipulate the configuration three. Once the modules
have finished, the configuration are then sent back to Clixon over the
socket.

1. The user configures a service from the CLI.
2. User commits the service.
3. The server receives a '<services-commit>' message.
4. All service modules are executed and finishes. Each one of the
   modifies the configuration tree.
5. The new configuration generated from the modules in step are sent
   back to Clixon.
6. In the event of an error we will abort and send an error message
   back to the user.
7. The new configuration is pushed to the devices by the backend.


Installation
============

Prerequisites
-------------

Installation of Cligen and Clixon is not covered in this section. The
Clixon controller must be up and running before the Python API can be
used.

We also expect Python, Pip etc to be installed on the system.


Installation
------------

Python API is available on GitHub::

  $ git clone https://github.com/clicon/clixon-pyapi.git

Once we've cloned it all requirementes have to be installed::

  $ cd clixon-pyapi
  $ pip3 install -r requirements.txt

And then we can install the Clixon library, this make the Clixon
library available on the system::

  $ sudo python3 setup.py install

The server must be installed manually::

  $ sudo cp clixon_server.py /usr/local/bin/


Usage
=====

Command line options
--------------------

The Python API server have to following command line options::

   $ python3 clixon_server.py -h

   clixon_server.py -f<module1,module2> -s<path> -d -p<pidfile>
   -m       Modules path
   -f       Comma separate list of modules to exclude
   -d       Enable verbose debug logging
   -s       Clixon socket path
   -p       Pidfile for Python server
   -F       Run in foreground
   -P       Prettyprint XML
   -h       This!

Logging and debugging
---------------------
In case of debugging, the server can be run in the foreground and with debug flags:
::

   clixon_server.py -F -d -P

Startup
-------

When starting the server there is one thing we want to control, and
that is where the modules for the service logic are located. This can be modified with the '-m' flag.

Once we know where the modules are located we can start the server::

  python3 ./clixon_server.py -m /home/khn/Git/clixon-pyapi/modules/

The command line above let the server run in the background with
minimal logging.



