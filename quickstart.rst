.. _clixon_quickstart:
.. sectnum::
   :start: 3
   :depth: 3

***********
Quick start
***********

This section describes how to run the *hello world* example available in source code at: `clixon hello example <https://github.com/clicon/clixon-examples/tree/master/hello/src>`_. 

Clixon is not a system in itself, it is a support system for an
application. In this case, the "application" is hello world. The hello
world application is very simple where the application semantics is
completely described by a YANG and CLI specification.

A more advanced application have backend and frontend plugins
that define application-specific semantics. No plugins are present in the
hello world application.

The hello world example can be run both natively on the host and in a docker container.

Host native
===========

Clixon
------
Go through the :ref:`install instructions <clixon_install>` to install
Clixon on your platform.  This includes installing CLIgen, Clixon,
creating users, groups, etc.

In short::
   
    git clone https://github.com/clicon/cligen.git
    cd cligen
    ./configure
    make && sudo make install
    git clone https://github.com/clicon/clixon.git
    cd clixon
    ./configure
    make && sudo make install         

Then proceed with host application install.

Files
-----
Files relevant to the hello world example are:

* `hello.xml <https://github.com/clicon/clixon-examples/tree/master/hello/src/hello.xml>`_: the XML configuration file
* `clixon-hello@2019-04-17.yang <https://github.com/clicon/clixon-examples/tree/master/hello/yang/clixon-hello@2019-04-17.yang>`_: the YANG spec
* `hello_cli.cli <https://github.com/clicon/clixon-examples/tree/master/hello/src/hello_cli.cli>`_: the CLIgen spec
* `startup_db <https://github.com/clicon/clixon-examples/tree/master/hello/src/startup_db>`_: The startup datastore containing restconf port configuration
* `Makefile <https://github.com/clicon/clixon-examples/tree/master/hello/src/Makefile.in>`_: install of specs, normally compile of plugins


Install and run
---------------

Checkout and configure the examples on the top-level::

    git clone https://github.com/clicon/clixon-examples.git
    cd clixon-examples
    ./configure

Compile and install::

    cd hello/src
    make && sudo make install

Start backend in the background:
::

    sudo clixon_backend

Start cli:
::

    clixon_cli

Using the CLI
=============
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
=======
Clixon also provides a Netconf interface. The following example starts a netconf client form the shell vi stdio, adds the hello world config, commits it, and shows it:
::

   olof@vandal> clixon_netconf -q
   <?xml version="1.0" encoding="UTF-8"?>
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability></capabilities></hello>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><edit-config><target><candidate/></target><config><hello xmlns="urn:example:hello"><world/></hello></config></edit-config></rpc>]]>]]>
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><ok/></rpc-reply>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><commit/></rpc>]]>]]>
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><ok/></rpc-reply>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get-config><source><running/></source></get-config></rpc>]]>]]>
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><data><hello xmlns="urn:example:hello"><world/></hello></data></rpc-reply>]]>]]>
   olof@vandal> 

Restconf
========
By default, Clixon uses `Native http`: supporting http/1 and http/2 (libnghttp2). The http server is integrated with the clixon restconf daemon and needs no extra installations, apart from ensuring you have server and client certs for https.

As an alternative, you can use the `FCGI` solution, where instead a reverse proxy such as `Nginx <https://nginx.org>`_  uses an internal FCGI socket communication to communicate with Clixon.  A reverse proxy, such as NGINX, needs to be configured. For more info about the fcgi solution, see :ref:`Restconf section<clixon_restconf>`.

  
Start and run
-------------
Regardless of which RESTCONF variant is used, start the restconf daemon as follows::

   sudo clixon_restconf

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


Docker container
================
You can run the hello example as a pre-built docker container, on a `x86_64` Linux. See instructions in the `clixon docker hello example <https://github.com/clicon/clixon-examples/tree/master/hello/docker>`_.

First, the container is started with the backend running:
::

 $ sudo docker run --rm -p 8080:80 --name hello -d clixon/hello

Then a CLI is started
::
   
 $ sudo docker exec -it hello clixon_cli
 cli> set ?
  hello                 
 cli> set hello world 
 cli> show configuration 
 hello world;

Or Netconf:
::

   $ sudo docker exec -it clixon/clixon clixon_netconf
   <?xml version="1.0" encoding="UTF-8"?>
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability></capabilities></hello>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get-config><source><candidate/></source></get-config></rpc>]]>]]>
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><data/></rpc-reply>]]>]]>

Or using restconf using curl on exposed port 8080:
::
   
  $ curl -X GET http://localhost:8080/restconf/data/hello:system
   
Next steps
==========
The hello world example only has a Yang spec and a template CLI
spec. For more advanced applications, customized backend, CLI, netconf
and restconf code callbacks becomes necessary.

Further, you may want to add upgrade, RPC:s, state data, notification
streams, authentication and authorization. The `main example <https://github.com/clicon/clixon/tree/master/example/main>`_ contains such capabilities.
