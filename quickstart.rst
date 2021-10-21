.. _clixon_quickstart:

Quick start
===========

.. This is a comment
   
This section describes how to run the Hello world example available in source code at: `clixon hello example <https://github.com/clicon/clixon-examples/tree/master/hello/src>`_.

Clixon is not a system in itself, it is a support system for an
application. In this case, the "application" is hello world. The hello
world application is very simple where the application semantics is
completely described by a YANG specification and a CLI specification.

A more advanced application will have backend (and frontend) plugins
to add application-specific semantics. This is not necessary in the
hello world application.

Content
-------
Files relevant to the hello example are:

* `hello.xml <https://github.com/clicon/clixon-examples/tree/master/hello/src/hello.xml>`_: the XML configuration file
* `clixon-hello@2019-04-17.yang <https://github.com/clicon/clixon-examples/tree/master/hello/yang/clixon-hello@2019-04-17.yang>`_: the YANG spec
* `hello_cli.cli <https://github.com/clicon/clixon-examples/tree/master/hello/src/hello_cli.cli>`_: the CLIgen spec
* `Makefile <https://github.com/clicon/clixon-examples/tree/master/hello/src/Makefile.in>`_: where plugins are built and installed

Compile and run
---------------

Before you start, go through the :ref:`install instructions <clixon_install>` for your platform.
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
--------

Clixon can use two RESTCONF compile-time variants:
  
*  `FCGI` : Reverse proxy such as `Nginx <https://nginx.org>`_  using an internal FCGI socket communication.  A reverse proxy needs to be configured.
*  `Native http`: web-server using `libevhtp <https://github.com/clicon/clixon-libevhtp.git>`_ (http/1) and nghttp2 (http/2). The web server is integrated with the clixon restconf daemon and needs no extra installations, apart from ensuring you have server and client certs for https.

FCGI
^^^^
In the case of the FCGI solution, a reverse proxy such as nginx needs to be installed, and edit config file `/etc/nginx/sites-available/default`::

   server {
      ...
      location / {
         fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
         include fastcgi_params;
      }
      # Enable this for restconf notification streams
      location /streams {
         fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
	 include fastcgi_params;
 	 proxy_http_version 1.1;
	 proxy_set_header Connection "";
      }
   }

Note that the second part is necessary only for notification streams.

Start nginx daemon::
   
   sudo /etc/init.d/nginx start

or using systemd::
   
  sudo systemctl start nginx.service
  
Start and run
^^^^^^^^^^^^^
Regardless of which RESTCONF variant is used, next step is to start the restconf daemon:
::

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


Run a container
---------------
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
----------
The hello world example only has a Yang spec and a template CLI
spec. For more advanced applications, customized backend, CLI, netconf
and restconf code callbacks becomes necessary.

Further, you may want to add upgrade, RPC:s, state data, notification
streams, authentication and authorization. The `main example <https://github.com/clicon/clixon/tree/master/example/main>`_ contains such capabilities.
