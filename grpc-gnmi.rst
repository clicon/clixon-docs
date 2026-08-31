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
   Prototype gRPC/gNMI support is introduced in Clixon version 7.8

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
``$prefix/share/clixon/proto/``, including the required Google well-known
protobuf definitions (``any.proto``, ``descriptor.proto``, ``duration.proto``)
under ``$prefix/share/clixon/proto/google/protobuf/``.

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

Local client-side
=================
The recommended gNMI client is `gnmic <https://gnmic.openconfig.net>`_, a purpose-built
high-level gNMI CLI. It has the gNMI protobuf schema built in, so no proto files are
needed on the client side.

Install gnmic
-------------
Follow the instructions on `<https://gnmic.openconfig.net/install/>`_, for example::

   bash -c "$(curl -sL https://get-gnmic.openconfig.net)"

Since the Clixon gRPC daemon currently has no TLS, all examples use ``--insecure``.

Low-level debugging with grpcurl
--------------------------------
For low-level protocol debugging, a generic gRPC client such as `grpcurl
<https://github.com/fullstorydev/grpcurl>`_ can also be used. Unlike gnmic it needs
access to the ``gnmi.proto`` schema file and its dependencies.

On a development system these are available in the build tree. On a **runtime
system** (where only Clixon binaries are installed) they are installed by
``make install`` to ``$prefix/share/clixon/proto/``, where ``$prefix`` is typically
``/usr`` or ``/usr/local``::

   gnmi.proto
   gnmi_ext.proto
   google/protobuf/any.proto
   google/protobuf/descriptor.proto
   google/protobuf/duration.proto

grpcurl calls need an ``-import-path`` pointing at the installed proto directory::

   grpcurl -plaintext \
     -import-path $prefix/share/clixon/proto \
     -proto gnmi.proto \
     -d '{}' localhost:9339 gnmi.gNMI/Capabilities

Supported RPCs
==============

Capabilities
------------
Returns the set of YANG modules loaded by the Clixon backend together with supported
encodings (``JSON_IETF``, ``JSON``, ``ASCII``).

Example::

   gnmic -a 127.0.0.1:9339 --insecure capabilities

Get
---
Retrieves configuration and/or state data. The ``type`` field controls what is returned:

* ``ALL`` — configuration and state data
* ``CONFIG`` — configuration data only
* ``STATE`` — state (operational) data only

The ``path`` elements are translated to XPath for the backend query. Multiple paths in
a single request are each queried independently.

Example — get a subtree::

   gnmic -a 127.0.0.1:9339 --insecure \
     get --path "/interfaces/interface[name=eth0]" --type ALL

Set
---
Modifies configuration data. Supports three operations in a single request:

* ``update`` — merge the given value into the datastore
* ``replace`` — replace the subtree with the given value
* ``delete`` — remove the specified path

Each operation is applied in order: deletes, then replaces, then updates.

Example — set and delete a leaf::

   gnmic -a 127.0.0.1:9339 --insecure \
     set --update-path "/example:val" --update-value "hello"

   gnmic -a 127.0.0.1:9339 --insecure \
     set --delete "/example:val"

Subscribe
---------
The ``ONCE``, ``STREAM`` and ``POLL`` subscription list modes are supported.

ONCE
^^^^
A ``SubscribeRequest`` with ``mode=ONCE`` returns current data for each subscribed path
as a series of ``SubscribeResponse`` update messages, followed by a final
``sync_response=true`` message, after which the RPC completes.

Example — subscribe ONCE::

  gnmic -a 127.0.0.1:9339 --insecure \
    subscribe --path "/example:val" --mode once

STREAM
^^^^^^
A ``SubscribeRequest`` with ``mode=STREAM`` first returns the current data for each
subscribed path followed by ``sync_response=true`` (unless ``updates_only`` is set, in
which case only the sync_response is sent), then keeps the RPC open and sends periodic
updates.

The ``SAMPLE`` and ``TARGET_DEFINED`` per-subscription modes are supported; both sample
the subscribed path periodically:

* ``sample_interval`` — sampling period in nanoseconds; 0 means target-defined (10s default). Intervals below 100 ms are clamped.
* ``suppress_redundant`` — if set, updates are only sent when the value has changed since the last update
* ``heartbeat_interval`` — with ``suppress_redundant``, forces an update after this period even if the value is unchanged

The ``ON_CHANGE`` per-subscription mode is not yet implemented.

The subscription is terminated when the client closes or cancels the RPC.

Example — stream with 2s sampling::

  gnmic -a 127.0.0.1:9339 --insecure \
    subscribe --path "/example:val" \
    --mode stream --stream-mode sample --sample-interval 2s

POLL
^^^^
A ``SubscribeRequest`` with ``mode=POLL`` first returns initial data and
``sync_response=true`` as for STREAM, then keeps the RPC open. Each subsequent
``SubscribeRequest`` containing a ``poll`` message triggers a fresh set of updates for
all subscribed paths, followed by a ``sync_response``.

Per the gNMI specification, any message other than ``poll`` sent after the initial
request terminates the RPC with an ``INVALID_ARGUMENT`` error.

Example — poll mode (gnmic prompts interactively for each poll)::

  gnmic -a 127.0.0.1:9339 --insecure \
    subscribe --path "/example:val" --mode poll

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
* **Subscribe ON_CHANGE** — the ``ON_CHANGE`` per-subscription mode is not implemented; ``STREAM`` subscriptions are sampled periodically
* **Subscribe encoding** — Subscribe responses always use ASCII encoding regardless of the requested encoding
* **Prefix field** — the ``prefix`` field in ``GetRequest`` and ``SetRequest`` is silently ignored; all paths must be absolute
* **Path wildcards** — ``*`` and ``...`` path wildcards are not supported
* **union_replace** — the Set ``union_replace`` operation is not implemented
* **use_models** — the ``use_models`` field in ``GetRequest`` is ignored

