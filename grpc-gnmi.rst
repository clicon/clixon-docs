.. _grpc_gnmi:
.. sectnum::
   :start: 13
   :depth: 3

**********
gNMI/gRPC
**********

Clixon supports `gNMI <https://github.com/openconfig/gnmi>`_ (gRPC Network Management Interface)
via a standalone daemon ``clixon_grpc``. gNMI is a gRPC-based protocol for retrieving and
modifying device configuration and operational state using YANG-defined data models.

.. note::
   Preliminary gRPC/gNMI support is introduced in Clixon version 7.8

Architecture
============

The gRPC daemon acts as a northbound interface translating gNMI RPCs to Clixon internal IPC
calls to the Clixon backend. It runs as a separate process alongside the backend and other
frontends (CLI, NETCONF, RESTCONF).

::

   gNMI client
       |  gRPC/HTTP2 (TCP port 9339)
       v
   clixon_grpc
       |  Internal IPC
       v
   clixon_backend

The transport layer uses HTTP/2 with Length-Prefixed-Message (LPM) gRPC framing.
Protobuf encoding uses the `gnmi.proto` from `OpenConfig/gNMI
<https://github.com/openconfig/gnmi/blob/master/proto/gnmi/gnmi.proto>`_.

Build and Install
=================

Dependencies
------------

* ``libnghttp2`` — HTTP/2 library (already required for RESTCONF)
* ``libprotobuf-c`` — C protobuf runtime library
* ``protoc-gen-c`` — protobuf-c compiler plugin (build time only)

Enable the gRPC daemon at configure time::

   ./configure --enable-grpc

Then build and install::

   make
   sudo make install

This builds the ``clixon_grpc`` executable and installs proto files to
``$prefix/share/clixon/proto/``.

Starting clixon_grpc
====================

Command-line options
--------------------
::

   $ clixon_grpc -h
   usage:clixon_grpc [options]
   where options are
	-h 		Help
	-D <level>	Debug level
	-f <file>	Clixon config file
	-l <s|e|o|n|f<file>> 	Log on (s)yslog, std(e)rr, std(o)ut, (n)one or (f)ile
	-p <port>	gRPC listen port (default: 9339)
	-d 		Daemonize
	-1 		Oneshot: connect to backend and exit

The ``-f`` flag points to the same Clixon configuration file used by other daemons.
The default gNMI port (9339) can be overridden with ``-p``.

Configuration
-------------

No additional Clixon configuration options are needed beyond the standard configuration
file. The backend must be running before ``clixon_grpc`` is started.

Example startup sequence::

   clixon_backend -f /etc/clixon/example.xml -s init -d
   clixon_grpc -f /etc/clixon/example.xml -d

Supported RPCs
==============

Capabilities
------------
Returns the set of YANG modules loaded by the Clixon backend together with supported
encodings (``JSON_IETF``, ``JSON``, ``ASCII``).

Example::

   grpcurl -plaintext \
     -import-path /usr/local/share/clixon/proto \
     -import-path /usr/include \
     -proto gnmi.proto \
     -d '{}' localhost:9339 gnmi.gNMI/Capabilities

Get
---
Retrieves configuration and/or state data. The ``type`` field controls what is returned:

* ``ALL`` — configuration and state data
* ``CONFIG`` — configuration data only
* ``STATE`` — state (operational) data only

The ``path`` elements are translated to XPath for the backend query. Multiple paths in
a single request are each queried independently.

Example — get a leaf::

   grpcurl -plaintext \
     -import-path /usr/local/share/clixon/proto \
     -import-path /usr/include \
     -proto gnmi.proto \
     -d '{"path":[{"elem":[{"name":"interfaces"},{"name":"interface","key":{"name":"eth0"}}]}],"type":"ALL","encoding":"ASCII"}' \
     localhost:9339 gnmi.gNMI/Get

Set
---
Modifies configuration data. Supports three operations in a single request:

* ``update`` — merge the given value into the datastore
* ``replace`` — replace the subtree with the given value
* ``delete`` — remove the specified path

Each operation is applied in order: deletes, then replaces, then updates.

Example — set a leaf::

   grpcurl -plaintext \
     -import-path /usr/local/share/clixon/proto \
     -import-path /usr/include \
     -proto gnmi.proto \
     -d '{"update":[{"path":{"elem":[{"name":"val"}]},"val":{"string_val":"hello"}}]}' \
     localhost:9339 gnmi.gNMI/Set

Subscribe
---------
The ``ONCE`` mode is supported. A ``SubscribeRequest`` with ``mode=ONCE`` returns
current data for each subscribed path as a series of ``SubscribeResponse`` update
messages, followed by a final ``sync_response=true`` message.

Example — subscribe ONCE::

   grpcurl -plaintext \
     -import-path /usr/local/share/clixon/proto \
     -import-path /usr/include \
     -proto gnmi.proto \
     -d '{"subscribe":{"mode":"ONCE","encoding":"ASCII","subscription":[{"path":{"elem":[{"name":"val"}]}}]}}' \
     localhost:9339 gnmi.gNMI/Subscribe

Encodings
=========

The following gNMI encodings are supported:

* ``JSON_IETF`` — RFC 7951 JSON encoding (default for Get responses)
* ``JSON`` — JSON encoding
* ``ASCII`` — plain text (used for Subscribe responses)

``PROTO`` and ``BYTES`` encodings are not supported.

Limitations
===========

The following features are not yet implemented:

* **TLS** — plain TCP only; no SSL/TLS transport
* **Authentication** — no credentials or certificate validation
* **NACM** — no access control applied
* **Subscribe STREAM and POLL** — only ``ONCE`` mode is implemented
* **Subscribe encoding** — Subscribe responses always use ASCII encoding regardless of the requested encoding
* **Prefix field** — the ``prefix`` field in ``GetRequest`` and ``SetRequest`` is silently ignored; all paths must be absolute
* **Path wildcards** — ``*`` and ``...`` path wildcards are not supported
* **union_replace** — the Set ``union_replace`` operation is not implemented
* **use_models** — the ``use_models`` field in ``GetRequest`` is ignored

