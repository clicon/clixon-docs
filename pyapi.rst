.. _clixon_pyapi:
.. sectnum::
   :start: 14
   :depth: 3

**********
Python API
**********

This section documents the Clixon Python API.

Overview
========

The Clixon Python API consist of two parts:

- The server (clixon_server.py).
- Service modules.

The server listens for events from the Clixon backend will run the
modules when needed. All service logic are implemented in the modules
and will be described in detail below.


Internals
=========

The Python API will connect to Clixons internal socket and communicate
over NETCONF. When started it will enable notifications for service
commits and controller transactions.

Whenever a commit is done the Clixon Controller will issue a notify on
the backend socket which will be catched by the Python API.

When the Python API receives a service commit it will run all the
service modules which can manipulate the configuration three. Once the
modules have finished, the configuration will be sent back to Clixon
over the socket.

1. The user configures a service from the CLI.
2. User commits the service.
3. The server receives a '<services-commit>' message.
4. All service modules are executed and finishes. Each one of the
   modifies the configuration tree.
5. The new configuration generated from the modules in step will be
   sent back to Clixon.
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

And then we can install the Clixon library, this will make the Clixon
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

The command line above will let the server run in the background with minimal logging.


Service modules
===============

Service modules contains the actual code and logic which is used when
modifying the configuration three for services. When a module is
launched by the Python server the server will call the setup method.

A minimal service module may look like this:

.. code:: python

  from clixon.clixon import rpc


  @rpc()
  def setup(root, log):
      log.info("I am a module")


Modules basics
--------------

The setup method takes two parameter, root and log. Log is used for
logging and is a reference to a Python logging object. The log
parameter can be used to print log messages. If the server is running
in the foreground the log messages will be seen in the terminal,
otherwise they will be written to syslog.

.. code:: python

  from clixon.clixon import rpc


  @rpc()
  def setup(root, log):
      log.info("Informative log")
      log.error("Error log")
      log.debug("Debug log")

The root parameter is the configuration three received from the Clixon
backend.

Contents of the root parameter can be written in XML format by using the dumps() method:

.. code:: python

  from clixon.clixon import rpc


  @rpc()
  def setup(root, log):
      log.debug(root.dumps())


Python object tree
------------------

Manipulating the configuration three is the central part of the
service modules. For example we might have a service which only
purpose is to change the hostname on Juniper devices.

In the Juniper CLI one would do something similar to this to configure
the hostname::

  admin@junos> configure
  Entering configuration mode

  [edit]
  admin@junos# set system host-name foo-bar-baz

  [edit]
  admin@junos# commit
  commit complete

However, in the Clixon CLI we can model this behaviour to look like
whatever we want using service YANG models. For example altering the
hostname for a lot of devices could look like this::

  test@test> configure
  test@test[/]# set services hostname test hostname foo-bar-baz
  test@test[/]# commit

Clixon itself will not modify the configuration when the commit is
issued, but this must be implemented using a service module.

Let's start with an example:

.. code:: python

  from clixon.clixon import rpc

  @rpc()
  def setup(root, log):
      hostname = root.services.hostname.hostname

      for device in root.devices:
	  device.config.configuration.system.host_name


When the service module above is executed Clixon will automatically
call the setup method. The wrapper rpc will take care of fetching the
configuration tree from Clixon and write the modified configuration
back when the setup function returns.

As seen above we're modifying the object root which is passed as a
parameter to setup. root is the configuration we have in Clixon parsed
by the Python API and converted to a tree of Python objects.

We can also create new configuration. Let's use the same example again
and create a new node named test:

.. code:: python

  from clixon.clixon import rpc

  @rpc()
  def setup(root, log):
      device.config.configuration.system.create("test", cdata="foo")

The code above would translate to an NETCONF/XML string which looks like this:

.. code:: xml

  <device>
    <config>
      <configuration>
	<system>
	  <test>
	    foo
	  </test>
	</system>
      </configuration>
    </config>
  </device>

Object tree API
---------------

Clixon Python API contains a few methods to work with the
configuration three.

Parsing
^^^^^^^

The most fundamental method is parse_string from parse.py, this method
will take any XML string and convert it to a tree of Python objects:

.. code:: python

  >>> from clixon.parser import parse_string
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>> root.xml.tags.tag
  foo
  >>>

As seen in the example above we get an object (root) back from
parse_string, root is a representation of the XML string xmlstr.

Something worth noting is that XML tags with '-' in them will be
renamed. A tag named "foo-bar" will have the name "foo_bar" after
being parsed since Python don't allow '-' in object names.

The original name will be saved and when the object tree is converted
back to XML the original name will be present:

.. code:: python

  >>> xmlstr = "<xml><tags><foo-bar>foo</foo-bar></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>> root.xml.tags.foo_bar
  foo
  >>> root.dumps()
  '<xml><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

Creation
^^^^^^^^

It is also possible to create the tree manually:

.. code:: python

  >>> from clixon.element import Element
  >>>
  >>> root = Element("root")
  >>> root.create("xml")
  >>> root.xml.create("tags")
  >>> root.xml.tags.create("foo-bar", cdata="foo")
  >>> root.dumps()
  '<xml><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

Attributes
^^^^^^^^^^

For any object it is possible to add attributes:

.. code:: python

  >>> root.xml.attributes = {"foo": "bar"}
  >>> root.dumps()
  '<xml foo="bar"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>> root.xml.attributes["baz"] = "baz"
  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

The Python API is not aware of namespaces etc but the user must handle
that.

Adding tags
^^^^^^^^^^^

We can now add a new tag to root and look at the generated XML using
the method dumps():

.. code:: python

  >>> root.xml.create("foo", cdata="bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags><foo>bar</foo></xml>'
  >>>

Renaming tags
^^^^^^^^^^^^^

If needed we can also rename a tag:

.. code:: python

  >>> root.xml.foo.rename("bar", "bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags><bar>bar</bar></xml>'
  >>>

Removing tags
^^^^^^^^^^^^^

And remove the tag:

.. code:: python

  >>> root.xml.delete("bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags></xml>'
  >>>

Altering CDATA
^^^^^^^^^^^^^^

CDATA can be altered:

.. code:: python

  >>> root.xml.tags.tag
  foo
  >>> root.xml.tags.tag.cdata = "baz"
  >>> root.xml.tags.tag
  baz
  >>> root.dumps()
  '<xml><tags><tag>baz</tag></tags></xml>'
  >>>

Iterate objects
^^^^^^^^^^^^^^^

We can also iterate over objects if we have a tag with tags below
it. Let's start over with a new example:

.. code:: python

  >>> from clixon.parser import parse_string
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag><tag>bar</tag><tag>baz</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>>
  >>> for tag in root.xml.tags.tag:
  ...     print(tag)
  ...
  foo
  bar
  baz
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>>
  >>> for tag in root.xml.tags.tag:
  ...     print(tag)
  ...
  foo

As seen above we have an XML string with a list of tags which we
iterate over.

Adding objects
^^^^^^^^^^^^^^

We can also add objects to the tree:

.. code:: python

  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>> new_tag = Element("new-tag")
  >>> new_tag.create("new-tag")
  >>> root.xml.tags.add(new_tag)
  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar><new-tag><new-tag/></new-tag></tags></xml>'
  >>>

The method add() will add the object to the tree and. The object must
be an Element object.
