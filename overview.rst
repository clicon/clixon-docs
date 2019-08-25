.. _clixon_overview:

Overview
========

Clixon is an open-source configuration tool that provides a
configuration database, CLI, NETCONF XML and RESTCONF interfaces all
defined by YANG.

Most of the projects using Clixon are for embedded networking
devices. But Clixon can be used for other YANG-based systems as well
due to a modular and pluggable architecture.

Clixon has a transaction mechanism that ensures configuration
operations are atomic. It also features a generated interactive
configuration CLI using `CLIgen <http://www.cligen.se>`_.

The goal of Clixon is to provide a useful, production-grade, scalable
and free YANG based configuration tool.

Clixon is open-source and dual licensed. Either Apache License, Version 2.0 or GNU
General Public License Version 2.


System Architecture
-------------------

::
   
                  +------------------------------------------+
                  |  +----------+                            |
                  |  |   cli    |--+                         |
                  |  +----------+   \ +----------+--------+  |
                  |  +----------+    \|          |        |  |
    User <-->     |  | restconf |---- | backend  | plugins|  |   <--> System
                  |  +----------+    /| daemon   |        |  |
                  |  +----------+   / +----------+--------+  |
	          |  | netconf  |--+  | datastore|           |
		  |  +----------+     +----------+           |
                  +------------------------------------------+
		 
The core of the  system architecture is a backend daemon with a configuration
datastore and a set of internal clients: cli, restconf and netconf.

The clients provide different frontend interfaces to external users,
including an interactive CLI, RESTCONF over HTTP, and XML NETCONF over
SSH.  Internally, the clients and backend communicate via NETCONF over
a UNIX socket.

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
generate the configuration part of the CLI,including set, delete, show
commands for a specific syntax.
   

Supported Platforms
-------------------

Clixon supports GNU/Linux, FreeBSD and Docker. MacOS may work. Linux
platforms include 32 and 64 bits Ubuntu, Alpine, Raspian, etc.

Standards Clixon supports (some partially):

* `RFC5277 <http://www.rfc-base.org/txt/rfc-5277.txt>`_ NETCONF Event Notifications.
* `RFC5789 <http://www.rfc-base.org/txt/rfc-5289.txt>`_ PATCH Method for HTTP.
* `RFC6020 <https://www.rfc-editor.org/rfc/rfc6020.txt>`_ YANG - A Data Modeling Language for the Network Configuration Protocol (NETCONF).
* `RFC6241 <http://www.rfc-base.org/txt/rfc-6241.txt>`_ NETCONF Configuration Protocol.
* `RFC6242 <http://www.rfc-base.org/txt/rfc-6242.txt>`_ Using the NETCONF Configuration Protocol over Secure Shell (SSH)
* `RFC7895 <http://www.rfc-base.org/txt/rfc-7895.txt>`_ YANG Module Library
* `RFC7950 <http://www.rfc-base.org/txt/rfc-7950.txt>`_ The YANG 1.1 Data Modeling Language
* `RFC7951 <http://www.rfc-base.org/txt/rfc-7951.txt>`_ JSON Encoding of Data Modeled with YANG
* `RFC8040 <https://tools.ietf.org/html/rfc8040>`_ RESTCONF Protocol
* `RFC8341 <http://www.rfc-base.org/txt/rfc-8341.txt>`_ Network Configuration Access Control Model
* `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_
* `Namespaces in XML 1.0 <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_
* `XPATH 1.0 <https://www.w3.org/TR/xpath-10>`_
* `W3C XML XSD <http://www.w3.org/TR/2004/REC-xmlschema-2-20041028>`_

How to get Clixon
-----------------

Get the Clixon source code from `Github <http://github.com/clicon/clixon>`_

Support
-------
For support issues use the `Clixon slack channel <https://clixondev.slack.com>`_

Bug reports
-----------

Report bugs via `Github issues <https://github.com/clicon/clixon/issues>`_


