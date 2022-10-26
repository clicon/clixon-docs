.. _clixon_netconf:
.. sectnum::
   :start: 10
   :depth: 3

*******
NETCONF
*******

Overview
========
Netconf is an external client interface (cli and restconf are
other external interfaces). Netconf is also used in the internal IPC.

 .. image:: netconf1.jpg
   :width: 100%

Netconf is defined in RFC 6241 (see :ref:`Standards section <clixon_standards>`) and
implemented by the `clixon_netconf` client.

Any number of netconf clients can be created, each creating a new
session to the backend. The netconf client communicates to the outside
world via `stdio`. Usually one sets up an SSH sub-system to
communicate from external nodes.

Note that Netconf supports chunked framing defined in RFC 6241 from
Clixon 5.7, but examples may not be updated.

Command-line options
--------------------

The `clixon_netconf` client has the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default.
  -q              Quiet mode, server does not send hello message on startup
  -0              Set netconf base capability to 0, server does not expect hello, force EOM framing
  -1              Set netconf base capability to 1, server does not expect hello, force chunked framing
  -a <family>     Internal IPC backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal IPC socket domain path or IP addr (see -a)
  -d <dir>        Specify netconf plugin directory
  -p <dir>        Add Yang directory path (see CLICON_YANG_DIR)
  -y <file>       Load yang spec file (override yang main module)
  -U <user>       Over-ride unix user with a pseudo user for NACM.
  -t <sec>        Timeout in seconds. Quit after this time.
  -e              Do not ignore errors on packet input.
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Configure options
-----------------
The configuration file options related to NETCONF are the following:

CLICON_NETCONF_DIR
   Location of netconf .so plugins loaded alphabetically

CLICON_NETCONF_HELLO_OPTIONAL
   If true, an RPC can be processed directly with no preceeding hello message.
   This is not according to the standard RFC 6241 Sec 8.1.

CLICON_NETCONF_MESSAGE_ID_OPTIONAL
   If true, an RPC can be sent without a message-id.
   This is not according to the standard RFC 6241 Sec 4.1.


Starting
========
The Netconf client (``clixon_netconf``) can be started on the command line using stdin/stdout::

  > clixon_netconf -qf /usr/local/etc/clixon.conf < my.xml

It then reads and parses Netconf commands on stdin, eventually invokes
Netconf plugin callbacks, then establishes a connection to the backend
and usually sends the Netconf message over IPC to the backend. Some
commands (eg hello) are terminated in the client. The reply from the
backend is then displayed on stdout.

SSH subsystem
-------------
You can expose ``clixon_netconf`` as an SSH subsystem according to `RFC 6242`. Register the subsystem in ``/etc/sshd_config``::

	Subsystem netconf /usr/local/bin/clixon_netconf

and then invoke it from a client using::

	ssh -s <host> netconf

NACM
====
Clixon implements the `Network Configuration Access Control Model
<http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM / RFC8341).
NACM rpc and datanode access validation is supported, not outgoing notifications.

NACM rules apply to all datastores.

Restrictions
------------
Access notification authorization (Sec 3.4.6) is NOT implemented.

Data-node paths, eg ``<rule>...<path>ex:table/ex:parameter</path></rule>`` instance-identifiers are restricted to canonical namespace identifiers for both XML and JSON encoding. That is, if a symbol (such as ``table`` above) is a symbol in a module with prefix ``ex``, another prefix cannot be used, even though defined with a ``xmlns`` rule.

Config options
--------------
The following configuration options are related to NACM:

CLICON_NACM_MODE
  NACM mode is either: `disabled`, `internal`, or `external`. Default: `disabled`.

CLICON_NACM_FILE
  If NACM mode is external, this file contains the NACM config.

CLICON_NACM_CREDENTIALS
  Verify NACM user credentials with unix socket peer credentials.  This means that a NACM user must match a UNIX user accessing ``CLIXON_SOCK``. Credentials are either: `none`, `exact` or `except`. Default: `except`.

CLICON_NACM_RECOVERY_USER
   RFC8341 defines a 'recovery session' as outside its scope. Clixon defines this user as having special admin rights to exempt from all access control enforcements.

CLICON_NACM_DISABLED_ON_EMPTY
   RFC 8341 defines enable-nacm as true by default. Since also write-default is deny by default it leads to that empty configs can not be edited. Default: `false`.
  
Mode
----
NACM rules are either internal or external. If external, rules are loaded from a separate file, specified by the option ``CLICON_NACM_FILE``.

If the NACM mode is internal, the NACM configuration is a part of the
regular candidate/running datastore. NACM rules are read from the
`running` datastore, ie they need to be committed. 

Since NACM rules are part of the config itself it means that there may
be bootstrapping issues. In particular, NACM default is `enabled` with
read/exec permit, and write `deny`. Loading an empty config therefore
leads to a "deadlock" where no user can edit the datastore.

Work-arounds include restarting the backend with a NACM config in the startup db, or using a `recovery user`_.

Access control
--------------
NACM is implemented in the Clixon backend at:

* Incoming RPC (module-name/protocol-operation)
* Before modifying the data store (data create/delete/update)
* After retrieving data (data read)

User credentials
^^^^^^^^^^^^^^^^
Access control relies on a user and groups. When an internal Clixon
client communicates with the backend, it piggybacks the name of the
user in the request, See :ref:`Internal netconf username
<clixon_misc>`::

  <rpc username="myuser"><get-config><source><running/></source></get-config></rpc>

The authentication of the username needs to be done in the client by either SSL certs (such as in :ref:`RESTCONF auth callback <clixon_restconf>`) or by SSH (as in NETCONF/CLI over SSH).

The Clixon backend can check credentials of the client if it uses a
UNIX socket (not IP socket) for internal communication between clients
and backend. In this way, a username claimed by a client can be verified against the UNIX user credentials.

The allowed values of `CLICON_NACM_CREDENTIALS` is:

* `none`: Do not match NACM user to any user credentials. Any user can pose as any other user. Set this for IP sockets, or do not use NACM.
* `exact`: Exact match between NACM user and unix socket peer user. 
* `except`: Exact match between NACM user and unix socket peer user, `except` for root and `wwwuser`. This is default.


Recovery user
-------------
RFC 8341 defines a NACM emergency recovery session mechanism.  Clixon
implements a recovery user set by option
``CLICON_NACM_RECOVERY_USER``. If a client accesses the backend as
that user, all NACM rules will be bypassed. By default there is no such
user.

Moreover, this mechanism is controlled by `user credentials`_ which means
you can control who can act as the recovery user.

For example, by setting ``CLICON_NACM_CREDENTIALS`` to `except` the
RESTCONF daemon can make backend calls posing as the recovery user,
even though it runs as `wwwuser`.

Alternatively, ``CLICON_NACM_CREDENTIALS`` can be set to `exact` and
the recovery user as `root`, in which case only a netconf or cli
session running as root can make recovery operations.


Confirm-commit
==============
Confirm as defined in RFC 6241 Sec 8.4 is enabled by::

    <CLICON_FEATURE>ietf-netconf:confirmed-commit</CLICON_FEATURE>

Confirmed-commit adds the ``<cancel-commit>`` operation and more parameters to ``<commit>``. 

The "rollback" datastore is added and is used as a temporary revert datastore.

Callhome
========
With Clixon, you can make a solution following `RFC 8071: NETCONF Call Home and RESTCONF Call Home <http://www.rfc-editor.org/rfc/rfc8071.txt>`_ over SSH as a utility using openssh.

The solution is built "around" Clixon meaning that Clixon itself is
used as-is.  This may be referred to as "external" callhome since it
is done using external tools, not clixon itself. In contrast, rstconf
call-home is "internal", see callhome section in :ref:`Restconf section <clixon_restconf>`.

Other solutions are possible as well, especially on the
client side, and a full system integration requires a callhome
framework to determine when and how callhomes are made as well as
addressing the security implications addressed by RFC 8071.

Overview of a callhome architecture with a device (where clixon resides) and a client::

     device/server                             client
  +-----------------+  2b) tcp connect   +---------------------+
  | 2a) callhome    | ---------------->  | 1c) callhome-client |
  +-----------------+                    +---------------------+
          | 3)                                   ^  |
          v                                   1b)|  v 
  +-----------------+   4) ssh session   +---------------------+   5) stdio
  |     sshd -i     | <----------------> | 1a)   ssh           |  <------  <rpc>...</rpc>"
  +-----------------+                    |---------------------+   
          | stdio                      
  +-----------------+
  | clixon_netconf  |
  +-----------------+
          | 
  +-----------------+
  | clixon_backend  |
  +-----------------+


The steps to make a Netconf callhome is as follows:

1) Start the ssh client using ``-o ProxyUseFdpass=yes -o ProxyCommand="callhome-client"``. Callhome-client listens on port 4334 for incoming TCP connections.
2) Start the callhome program on the server making tcp connect to client on port 4334 establishing a tcp stream with the client
3) The callhome program starts ``sshd -i`` using the established stream socket 
4) The callhome-client returns with an open stream socket to the ssh client establishing an SSH stream to the server
5) Netconf messages are sent on stdin to the ssh client in turn using the established SSH stream and the Netconf subsystem to clixon, which returns a reply.

The callhome and callhome-client referred to above are implemented by the utility functions: ``util/clixon_netconf_ssh_callhome`` and ``util/clixon_netconf_ssh_callhome_client``.

Example::

  # Start ssh on client: bind to 1.2.3.4:4334 sending an rpc on stdin
  client> echo '<?xml version="1.0" encoding="UTF-8"?><hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability></capabilities></hello>]]>]]><rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get-config><source><candidate/></source></get-config></rpc>]]>]]>' > msg
  client> ssh -s -v -o ProxyUseFdpass=yes -o ProxyCommand="clixon_netconf_ssh_callhome_client -a 1.2.3.4" . netconf < msg

  # Start callhome on server: connect to 1.2.3.4:4334
  server> sudo clixon_netconf_ssh_callhome -a 1.2.3.4

  # Reply on client stdout: (skipping hello):
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><data/></rpc-reply>]]>]]>
  
The example is implemented as a regression test in ``test/test_netconf_ssh_callhome.sh``

The RFC lists several security issues that need to be addressed in a solution, including "pinning" of host keys etc.

.. note::
        Warning: there are security implications of using this example as noted in `RFC 8071: NETCONF Call Home and RESTCONF Call Home <http://www.rfc-editor.org/rfc/rfc8071.txt>`_

IPC
===
Clixon uses NETCONF in the IPC protocol between its clients
(cli/netconf/restconf) and the backend. This *internal* Netconf (IPC)
is slightly different from regular Netconf:

- A different framing
- Netconf extentions

.. note::
        The IPC is an internal interface, do not use externally
  
Framing
-------
A fixed header using session id and message length before the netconf message::

  struct clicon_msg {
     uint32_t    op_len;     /* length of message. network byte order. */
     uint32_t    op_id;      /* session-id. network byte order. */
     char        op_body[0]; /* rest of message, actual data */
  };


Extensions
----------
The internal IPC protocol have a couple of extensions to the Netconf protocol as follows:

* *content* - for ``get`` command with values "config", "nonconfig" or "all", to indicate which parts of state and config are requested. This option is taken from RESTCONF. Example::

    <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get content="nonconfig"/></rpc>
    
* *depth* - for ``get`` and ``get-config`` how deep a tree is requested. Also from RESTCONF. Example::

    <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><get depth="2"/></rpc>
    
* *username* - for top-level ``rpc`` command. Indicates which user the client represents ("pseudo-user"). This is either the actual user logged in as the client (eg "peer-user") or can represent another user. The credentials mode determines the trust-level of the pseudo-username. Example::

    <rpc username="root"><close-session/></rpc>
    
* *autocommit* - for ``edit-config``. If true, perform a ``commit`` operation immediately after an edit. If this fails, make a ``discard`` operation. Example::

    <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><edit-config autocommit="true"><target><candidate/></target><config>...</config></edit-config></rpc>
    
* *copystartup* - for ``edit-config`` combined with autocommit. If true, copy the running db to the startup db after a commit. The combination with autocommit is the default for RESTCONF operations. Example::

     <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><edit-config autocommit="true" copystartup="true"><target><candidate/></target><config>...</config></edit-config></rpc>

* *objectcreate* and *objectexisted* - in the data field of ``edit-config`` XML data tree. In the request set objectcreate to false/true whether an object should be created if it does not exist or not. If such a request exists, then the ok reply should contain "objectexists" to indicate whether the object existed or not (eg prior to the operation). The reason for this protocol is to implement some RESTCONF PATCH and PUT functionalities. Example::

      <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
         <edit-config objectcreate="false"><target><candidate/></target>
            <config>
               <protocol objectcreate="true">tcp</protocol>
             </config>
         </edit-config>
      </rpc>]]>]]>
      <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
         <ok objectexisted="true"/>
      </rpc-reply>]]>]]>

The reason for introducing the objectcreate/objectexisted attributes are as follows:
      * RFC 8040 4.5 PUT: if the PUT request creates a new resource, a "201 Created" status-line is returned.  If an existing resource is modified, a "204 No Content" status-line is returned.
      * RFC 8040 4.6 PATCH: If the target resource instance does not exist, the server MUST NOT create it.


