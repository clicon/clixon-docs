.. _clixon_netconf:

NETCONF
=======

Overview
--------

Netconf is an external client interface (cli and restconf are
others). Netconf is also used in the internal IPC.

::

                   +---------------------------------------+
                   |  +------------+  IPC  +------------+  |
                   |  |  netconf   | ----- |  backend   |  |
      User <-----> |  |------------|       |------------|  |
                   |  | nc plugins |       | be plugins |  |
                   |  +------------+       +------------+  |
                   +---------------------------------------+

External Netconf
----------------

Netconf is defined in RFC 6241 (see :ref:`clixon_standards`) and
implemented by the `clixon_netconf` client.

Command-line options
^^^^^^^^^^^^^^^^^^^^

The `clixon_netconf` client has the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f: -lf<file>.
  -q              Quiet mode, do not run hello protocol
  -a <family>     Internal IPC backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal IPC socket domain path or IP addr (see -a)(default: /usr/var/hello.sock)
  -d <dir>        Specify netconf plugin directory
  -p <dir>        Yang directory path (see CLICON_YANG_DIR)
  -y <file>       Load yang spec file (override yang main module)
  -U <user>       Over-ride unix user with a pseudo user for NACM.
  -t <sec>        Timeout in seconds. Quit after this time.
  -e              Dont ignore errors on packet input.
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Options
^^^^^^^
The configuration file options related to RESTCONF common to both fcgi and evhtp are the following:

CLICON_RESTCONF_DIR
   Location of restconf .so plugins. Load all .so plugins in this dir as restconf code plugins.

CLICON_RESTCONF_PATH
   FCGI unix socket. Should be specified in webserver (only fcgi)

CLICON_RESTCONF_PRETTY
   RESTCONF return value is pretty-printed or not


Starting
--------
The netconf client (``clixon_netconf``) can be started on the command line using stdin/stdout::

  > clixon_netconf -qf /usr/local/etc/clixon.conf < my.xml

It then reads and parses netconf commands on stdin, eventually invokes
netconf plugin callbacks, then establishes a connection to the backend
and usually sends the netconf message over IPC to the backend. Some
commands (eg hello) are terminated in the client. The reply from the
backend is then displayed on stdout.

SSH subsystem
^^^^^^^^^^^^^

You can expose ``clixon_netconf`` as an SSH subsystem according to `RFC 6242`. Register the subsystem in ``/etc/sshd_config``::

	Subsystem netconf /usr/local/bin/clixon_netconf

and then invoke it from a client using::

	ssh -s <host> netconf

IPC
---

Clixon uses NETCONF in the IPC protocol between its clients
(cli/netconf/restconf) and the backend. This *internal* Netconf (IPC)
is slightly different from regular Netconf:

- A different framing
- Netconf extentions

.. note::
        The IPC is an internal interface, do not use externally
  
Framing
^^^^^^^
A fixed header using session id and message length before the netconf message::

  struct clicon_msg {
     uint32_t    op_len;     /* length of message. network byte order. */
     uint32_t    op_id;      /* session-id. network byte order. */
     char        op_body[0]; /* rest of message, actual data */
  };


Extensions
^^^^^^^^^^

The protocol have a couple of extensions to the Netconf protocol as follows:

* *content* - for ``get`` command with values "config", "nonconfig" or "all", to indicate which parts of state and config are requested. This option is taken from RESTCONF. Example::

    <rpc><get content="nonconfig"/></rpc>
    
* *depth* - for ``get`` and ``get-config`` how deep a tree is requested. Also from RESTCONF. Example::

    <rpc><get depth="2"/></rpc>
    
* *username* - for top-level ``rpc`` command. Indicates which user the client represents ("pseudo-user"). This is either the actual user logged in as the client (eg "peer-user") or can represent another user. The credentials mode determines the trust-level of the pseudo-username. Example::

    <rpc username="root"><close-session/></rpc>
    
* *autocommit* - for ``edit-config``. If true, perform a ``commit`` operation immediately after an edit. If this fails, make a ``discard`` operation. Example::

    <rpc><edit-config autocommit="true"><target><candidate/></target><config>...</config></edit-config></rpc>
    
* *copystartup* - for ``edit-config`` combined with autocommit. If true, copy the running db to the startup db after a commit. The combination with autocommit is the default for RESTCONF operations. Example::

     <rpc><edit-config autocommit="true" copystartup="true"><target><candidate/></target><config>...</config></edit-config></rpc>

* *objectcreate* and *objectexisted* - in the data field of ``edit-config`` XML data tree. In the request set objectcreate to false/true whether an object should be created if it does not exist or not. If such a request exists, then the ok reply should contain "objectexists" to indicate whether the object existed or not (eg prior to the operation). The reason for this protocol is to implement some RESTCONF PATCH and PUT functionalities. Example::

      <rpc><edit-config objectcreate="false"><target><candidate/></target>
         <config>
            <protocol objectcreate="true">tcp</protocol>
         </config>
      </edit-config></rpc>]]>]]>
      <rpc-reply><ok objectexisted="true"/></rpc-reply>]]>]]>

The reason for introducing the objectcreate/objectexisted attributes are as follows:
      * RFC 8040 4.5 PUT: if the PUT request creates a new resource, a "201 Created" status-line is returned.  If an existing resource is modified, a "204 No Content" status-line is returned.
      * RFC 8040 4.6 PATCH: If the target resource instance does not exist, the server MUST NOT create it.


