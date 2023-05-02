.. _clixon_pyapi:
.. sectnum::
   :start: 21
   :depth: 3

**********
Python API
**********

This section documents the Clixon Python API. 
The Python API is a stand-alone client which uses the internal Netconf protocol to the Clixon backend.
The primary application is the clixon-server and its network services.


Overview
========
The Clixon Python API consist of two parts:

- The server (clixon_server.py).
- Service modules.

The server listens for events from the Clixon backend and run the
modules when needed. All service logic are implemented in the modules
and are described in detail below.


Overview
========
The Python API connect to Clixons internal socket and communicate over
NETCONF. When started it registers for notifications of service commits
and controller transactions.

Whenever a commit occurs, the Clixon Controller issues a notification the Python API listens to.

When the Python API receives a service commit it runs all the service
modules which manipulates the configuration tree.  When finished, the
configuration is sent back to the Clixon backend.

In summary:
1. The user configures a service from the CLI.
2. The user commits the service.
3. Pyapi receives a `<services-commit>` notification.
4. Pyapi executes all service modules that may modify the configuration (device) tree.
5. For each modification, an `<edit-config>` message is sent to the backend.
6. When completed, pyapi sends `<transaction-actions-done>` to the backend.
7. If pyapi encounters an error, it aborts by sending `<transaction-error>` to the backend instead.
8. The new configuration is pushed to the devices by the backend.

Installation
============

Prerequisites
-------------
Installation of Cligen and Clixon is not covered in this section. The
Clixon controller must be up and running before the Python API can be
used.

It is expected that Python, Pip etc are installed on the system.


Installation
------------
Python API is available on GitHub::

  $ git clone https://github.com/clicon/clixon-pyapi.git

Once cloned, the requirementes are installed::

  $ cd clixon-pyapi
  $ pip3 install -r requirements.txt

And then the Clixon pyapi library is installed::

  $ sudo python3 setup.py install

The server is installed manually, for example::

  $ sudo cp clixon_server.py /usr/local/bin/

The install script `install.sh` performs the two steps above.

Usage
=====

Command line options
--------------------
The Python API server has the following command line options::

   $ python3 clixon_server.py -h

   clixon_server.py -f<module1,module2> -s<path> -d -p<pidfile>
   -m       Location of python code for service modules
   -f       Comma separate list of modules to exclude
   -d       Enable verbose debug logging
   -s       Clixon socket path
   -p       Pidfile for Python server
   -F       Run in foreground
   -P       Prettyprint XML
   -h       This!

Logging and debugging
---------------------
The server can be run in the foreground with debug flags::

   clixon_server.py -F -d -P

Startup
-------
Pyapi needs to know where the python code for the service model is located.
This can be modified with the '-m' flag::

  python3 ./clixon_server.py -m /usr/local/share/clixon-pyapi/

which makes the server run in the background with minimal logging.
