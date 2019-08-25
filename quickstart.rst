.. _clixon_quickstart:

Quick start
===========

.. This is a comment
   
This section describes how to run the Hello world example available in the soure code at: `hello example <https://github.com/clicon/clixon/tree/master/example/hello>`_.

Clixon is not a system in itself, it is a support system for an
application. In this case, the "application" is hello world. The hello
world application is very simple where the application semantics is
completely described by a YANG specification and a CLI specification.

A more advanced application will have backend (and frontend) plugins
to add application-specific semantics. This is not necessary in the
hello world application.

Content
-------
The application directory contains a Clixon example which includes a simple example. It contains the following files:

* `hello.xml`       The configuration file. See `clixon-config.yang <https://github.com/clicon/clixon/tree/master/yang/clixon/clixon-config@2019-06-05.yang>`_ for the uang spec of hello.xml.
* `clixon-hello@2019-04-17.yang` The yang spec of the example.
* `hello_cli.cli`                CLIgen specification.
* `Makefile.in`                  Example makefile where plugins are built and installed
* `README.md`                    This file


Compile and run
---------------

Before you start, go through the install instructions.
::

    make && sudo make install

Start backend in the background:
::

    sudo clixon_backend

Start cli:
::

    clixon_cli


Using the CLI
-------------

The example CLI allows you to modify and view the data model using `set`, `delete` and `show` via generated code.

The following example shows how to add a very simple configuration `hello world` using the generated CLI. The config is added to the candidate database, shown, committed to running, and then deleted.

::

   olof@vandal> clixon_cli
   cli> set <?>
     hello
   cli> set hello world
   cli> show configuration
   hello world;
   cli> commit
   cli> delete <?>
     all                   Delete whole candidate configuration
     hello
   cli> delete hello 
   cli> show configuration 
   cli> commit 
   cli> quit
   olof@vandal> 

Netconf
-------

Clixon also provides a Netconf interface. The following example starts a netconf client form the shell, adds the hello world config, commits it, and shows it:
::

   olof@vandal> clixon_netconf -q
   <rpc><edit-config><target><candidate/></target><config><hello xmlns="urn:example:hello"><world/></hello></config></edit-config></rpc>]]>]]>
   <rpc-reply><ok/></rpc-reply>]]>]]>
   <rpc><commit/></rpc>]]>]]>
   <rpc-reply><ok/></rpc-reply>]]>]]>
   <rpc><get-config><source><running/></source></get-config></rpc>]]>]]>
   <rpc-reply><data><hello xmlns="urn:example:hello"><world/></hello></data></rpc-reply>]]>]]>
   olof@vandal> 


Restconf
--------

Clixon also provides a Restconf interface. A reverse proxy needs to be configured. There are nginxsetup_ for Clixon.

Start restconf daemon
::

   sudo su -c "/www-data/clixon_restconf" -s /bin/sh www-data &

Start sending restconf commands (using Curl):
::

   olof@vandal> curl -X POST http://localhost/restconf/data -d '{"clixon-hello:hello":{"world":null}}'
   olof@vandal> curl -X GET http://localhost/restconf/data 
   {
      "data": {
        "clixon-hello:hello": {
          "world": null
        }
      }
   }

Next steps
----------
The hello world example only has a Yang spec and a template CLI
spec. For more advanced applications, customized backend, CLI, netconf
and restconf code callbacks becomes necessary.

Further, you may want to add upgrade, RPC:s, state data, notification
streams, authentication and authorization. The main example contains
such capabilities.





