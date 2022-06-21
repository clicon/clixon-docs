.. _clixon_backend:
.. sectnum::
   :start: 7
   :depth: 3

*******
Backend
*******

 .. image:: backend1.jpg
   :width: 100%
	       
The backend daemon is the central component in the Clixon architecture. It consists of a main module and a number of dynamically loaded plugins. The backend has four APIs:

*configuration*
  An XML file read at startup, possibly amended with `-o` options. 
*Internal interface / IPC*
  A NETCONF socket to frontend clients. This is by default a UNIX domain socket but can also be an IPv4 or IPv6 TCP socket but with limited functionality.
*Datastores*
  XML (or JSON) files storing configuration. The three main datastores are `candidate`, `running` and `startup`. A user edits the candidate datastore, commits the changes to  running which triggers callbacks in plugins.
*Application*
  Backend plugins configure the base system with application-specific APIs. These API:s depend on how the underlying system is configured, examples include configuration files or a socket.

Note that a user typically does not access the datastores directly, it is possible to read, but write operations should not be done, since the backend daemon in most cases uses a datastore cache.
   
Command-line options
====================
The backend have the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut, (n)one or (f)ile. Syslog is default. If foreground, then syslog and stderr is default.
  -d <dir>        Specify backend plugin directory
  -p <dir>        Add Yang directory path (see CLICON_YANG_DIR)
  -b <dir>        Specify datastore directory
  -F              Run in foreground, do not run as daemon
  -z              Kill other config daemon and exit
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)
  -P <file>       Process ID filename
  -1              Run once and then quit (do not wait for events)
  -s <mode>       Specify backend startup mode: none|startup|running|init)
  -c <file>       Load extra XML configuration file, but do not commit.
  -q              Quit startup directly after upgrading and print result on stdout
  -U <user>       Run backend daemon as this user AND drop privileges permanently
  -g <group>      Client membership required to this group (default: clicon)
  -y <file>       Load yang spec file (override yang main module)
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

  
Logging and debugging
---------------------
In case of debugging, the backend can be run in the foreground and with debug flags:
::

   clixon_backend -FD 1

Logging is by default on syslog.  Alternatively, logging can be made on a file using the `-l` option:
::

   clixon_backend -lf<file>

When run in foreground, logging is by default done on both syslog and stderr.

In a debugging mode, it can be useful to run in `once-only` mode, where the backend quits directly after starting up, instead of waiting for events:
::

   clixon_backend -F1D 1

Startup
=======
The backend can perform startup in four different modes. The difference is how the running state is handled, i.e., what state the system is in when you start the daemon and how loading the configuration affects it:

`none`
   Do not touch running state. Typically after crash when running state and db are synched.
`init`
   Initialize running state. Start with a completely clean running state.
`running`
   Commit running db configuration into running state. Typically after reboot if a persistent running db exists.
`startup`
   Commit startup configuration into running state. After reboot when no persistent running db exists.

Use the ``-s`` option to select startup mode, example::
   
   clixon_backend -s running

You may also add a default method in the configuration file:
::

   <clixon-config xmlns="http://clicon.org/config">
     ...
     <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE
   </clixon-config>

When loading the startup/tmp configuration, the following actions are performed by the system:

* Check syntax errors,
* Upgrade callbacks.
* Validation of the XML against the current Yang models
* If errors are detected, enter `failsafe` mode.

The following config option is related to startup:

CLICON_BACKEND_RESTCONF_PROCESS
   Enable process-control of restconf daemon, ie start/stop restconf daemon internally using fork/exec. Disable if you start the restconf daemon by other means.
  
IPC Socket
==========
::

   Frontends:           +----------+
   CLI,          IPC    | backend  |
   netconf      <--->   | daemon   |
   restconf             |          |
                        +----------+

The Clixon backend creates a socket that the frontend clients can connect to.  Communication is made over this IPC socket using internal Netconf.
The following config options are related to the internal socket:

CLICON_SOCK_FAMILY
   Address family for communicating with clixon_backend. One of: UNIX, IPv4, or IPv6. Can also be set with `-a` command-line option. Default is ``UNIX`` which denotes a UNIX domain socket.

CLICON_SOCK
  If the address family of the socket is ``AF_UNIX``: Unix socket for communicating with clixon_backend. If the family is ``AF_INET`` it denottes the IPv4 address;

CLICON_SOCK_PORT
  Inet socket port for communicating with clixon_backend (only IPv4|IPv6). Default is port `4535`.

CLICON_SOCK_GROUP
  Group membership to access clixon_backend UNIX socket. Default is `clicon`. This is not available for IP sockets.


Backend files
=============
The following config options control files related to the backend:

CLICON_BACKEND_DIR
  Location of backend ``.so`` plugins. Load all ``.so`` plugins in this dir as backend plugins in alphabetical order

CLICON_BACKEND_REGEXP
  Regexp of matching backend plugins in CLICON_BACKEND_DIR. default: ``*.so``

CLICON_BACKEND_PIDFILE
  Process-id file of backend daemon

Backend plugins
===============

This section describes backend-specific plugins, see :ref:`Plugin section<clixon_plugins>` for a general description of plugins.

Clixon_plugin_init
------------------
Apart from the generic plugin callbacks (init, start, etc), the following callbacks are specific to the backend:

pre_daemon
   Called just before server daemonizes(forks). Not called if in foreground.
daemon
   Called after the server has daemonized and before privileges are dropped. 
statedata
  Provide state data XML from a plugin
reset
  Reset system status
upgrade
  General-purpose upgrade called once when loading the startup datastore
trans_{begin,validate,complete,commit,commit_done,revert,end,abort}
  Transaction callbacks are invoked for two reasons: validation requests or commits.  These callbacks are further described in `transactions`_ section.

Registered callbacks
--------------------
The callback also supports three forms of registered callbacks:

* ``rpc_callback_register()`` - for user-defined RPC callbacks, see :ref:`Plugin section <clixon_plugins>`.
* ``action_callback_register()`` - for user-defined Action callbacks.
* ``upgrade_callback_register()`` - for upgrading, see :ref:`Upgrade section <clixon_upgrade>`.
* ``clixon_pagination_cb_register()`` - for pagination, as described in :ref:`Pagination section<clixon_pagination>`.
  
Transactions
============
Clixon follows NETCONF in its validate and commit semantics.
Using the CLI or another frontend, you edit the `candidate` configuration, which is first
`validated` for consistency and then `committed` to the `running`
configuration.

A clixon developer writes commit functions to incrementally upgrade a
system state based on configuration changes. Writing commit callbacks
is the core functionality of a clixon system.

The netconf validation and commit operation is implemented in
Clixon by a transaction mechanism, which ensures that user-written
plugin callbacks are invoked atomically and revert on error. The transaction callbacks are the following:

begin
   Transaction start
validate
   Transaction validation
complete
   Transaction validation complete
commit
   Transaction commit
commit_done
   Transaction when commit done. After this, the source db is copied to target and default values removed.
end
   Transaction completed. 
revert
   Transaction revert
abort
   Transaction aborted

In a system with two plugins, for example, a transaction sequence looks like the following::
   
  Backend   Plugin1    Plugin2
  |          |          |
  +--------->+--------->+ begin
  |          |          |
  +--------->+--------->+ validate
  |          |          |
  +--------->+--------->+ complete
  |          |          |
  +--------->+--------->+ commit
  |          |          |
  +--------->+--------->+ commit_done
  |          |          |
  +--------->+--------->+ end


If an error occurs in the commit call of plugin2, for example,
the transaction is aborted and the commit reverted:
::

  Backend   Plugin1    Plugin2
  |          |          |
  +--------->+--------->+ begin
  |          |          |
  +--------->+--------->+ validate
  |          |          |
  +--------->+---->X    + commit error
  |          |          |
  +--------->+          + revert
  |          |          |
  +--------->+--------->+ abort

Callbacks
---------
In the the context of a callback, there are two XML trees:

src
  The original XML tree, such as "running"
target
  The new XML tree, such as "candidate"
  
There are three vectors pointing into the XML trees:

delete
  The delete vector consists of XML nodes in "src" that are removed in "target"
add
  The add vector consists of nodes that exists in "target" and do not exist in "src"
change
  The change vector consists of nodes that exists in both "src" and "target" but are different
  
All transaction callbacks are called with a transaction-data argument
(td). The transaction data describes a system transition from a src to
target state.  The struct contains source and target XML tree
(e.g. candidate/running) in the form of XML tree vectors (dvec, avec,
cvec) and associated lengths. These contain the difference between src
and target XML, ie "what has changed".  It is up to the validate
callbacks to ensure that these changes are OK. It is up to the commit
callbacks to enforce these changes in the configuration of the system.

The "td" parameter can be accessed with the following functions:

``uint64_t transaction_id(transaction_data td)``
  Get the unique transaction-id
``void   *transaction_arg(transaction_data td)``
  Get plugin/application specific callback argument
``int     transaction_arg_set(transaction_data td, void *arg)``
  Set callback argument
``cxobj  *transaction_src(transaction_data td)``
  Get source database xml tree
``cxobj  *transaction_target(transaction_data td)``
  Get target database xml tree
``cxobj **transaction_dvec(transaction_data td)``
  Get vector of xml nodes that are deleted src->target
``size_t  transaction_dlen(transaction_data td)``
  Get length of delete xml vector
``cxobj **transaction_avec(transaction_data td)``
  Get vector of xml nodes that are added src->target
``size_t  transaction_alen(transaction_data td)``
  Get length of add xml vector
``cxobj **transaction_scvec(transaction_data td)``
  Get vector of xml nodes that changed
``cxobj **transaction_tcvec(transaction_data td)``
  Get vector of xml nodes that changed
``size_t  transaction_clen(transaction_data td)``
  Get length of changed xml vector

A programmer can also use XML flags that are set in "src" and "target" XML trees to identify what has changed. The following flags are used to makr the trees:

XML_FLAG_DEL
  All deleted XML nodes in "src" and all its descendants
XML_FLAG_ADD
  All added XML nodes in "target" and all its descendants
XML_FLAG_CHANGE
  All changed XML nodes in both "src" and "target" and all its descendants.
  Also all ancestors of all added, deleted and changed nodes.

For example, assume the tree (A B) is replaced with (B C), then the two trees are marked with the following flags::

             src            target
              o CHANGE        o CHANGE
              |               |
              o CHANGE        o CHANGE
             /|               |\
   DELETE   A B               B C ADD

  
Privileges
==========
The backend process itself does not really require any specific
access, but it may be an important topic for an application using
clixon when the plugins are designed. A plugin may need to access
privileged system resources (such as configure files).

The backend itself is usually started as root: `sudo clixon_backend -s init`, which means that the plugins also run as root (being part of the same process).

The backend can also be started as a non-root user. However, you may
need to set some config options to allow user write access, for
example as follows(there may be others):
::
   
    <CLICON_SOCK>/tmp/example.sock</CLICON_SOCK>
    <CLICON_BACKEND_PIDFILE>/tmp/mytest/example.pid</CLICON_BACKEND_PIDFILE>
    <CLICON_XMLDB_DIR>/tmp/mytest</CLICON_XMLDB_DIR>

Dropping privileges
-------------------
You may want to start the backend as root and then drop privileges
to a non-root user which is a common technique to limit exposure of exploits.

This can be done either by command line-options: `sudo clicon_backend -s init -U clicon` or (more generally) using configure options:
::

    <CLICON_BACKEND_USER>clicon</CLICON_BACKEND_USER>
    <CLICON_BACKEND_PRIVILEGES>drop_perm</CLICON_BACKEND_PRIVILEGES>

This will initialize resources as root and then *permanently* drop uid:s to the
unprivileged user (`clicon` in the example abobe). It will also change
ownership of several files to the user, including datastores and the
clicon socket (if the socket is unix domain).

Note that the unprivileged user must exist on the system, see :ref:`Install section<clixon_install>`.
 
Drop privileges temporary
-------------------------
If you drop privileges permanently, you need to access all privileged
resources initially before the drop. For a plugin designer, this means
that you need to access privileges system resources in the
`plugin_init` or `plugin_start` callbacks. The transaction callbacks, for example, will be run in unprivileged mode.

An alternative is to drop privileges temporary and then be able to raise privileges when needed:
::

    <CLICON_BACKEND_USER>clicon</CLICON_BACKEND_USER>
    <CLICON_BACKEND_PRIVILEGES>drop_temp</CLICON_BACKEND_PRIVILEGES>

In this mode, a plugin callback (eg commit), can temporarily raise the
privileges when accessing system resources, and the lower them when done.

An example C-code for raising privileges in a plugin is as follows:
::

   uid_t euid = geteuid();
   restore_priv();
   ... make high privilege stuff...
   drop_priv_temp(euid);
