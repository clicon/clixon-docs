.. _clixon_backend:

Backend
=======

::

                           +----------------+
                           | clixon options |
                           +----------------+
                                   |- XML
                                   v
   Frontends:    IPC    +----------+--------+
   CLI,           |     | backend  | plugin |   <-->  Underlying
   netconf      <--->   | daemon   |--------+         System
   restconf             |          | plugin |   <-->   
                        +----------+--------+
                                   ^
                                   |- XML
               Datastores:         v	                
               +-----------+  +-----------+  +-----------+
               | candidate |  |  running  |  |  startup  |
               +-----------+  +-----------+  +-----------+

The backend daemon is the central Clixon component. It consists of a main module and a number of dynamically loaded plugins. The backend has four APIs:

*configuration*
  An XML file read at startup, possibly amended with `-o` options.
*Internal interface*
  A NETCONF socket to frontend clients. This is by default a UNIX domain socket but can also be an IPv4 or IPv6 TCP socket.
*Datastores*
  XML files storing configuration. The three main datastores are `candidate`, `running` and `startup`. A user edits the candidate datastore, commits the changes to  running which triggers callbacks in the plugins.
*Application*
  Backend plugins configure the underlying system with application-specific APIs. These API:s depend on how the underlying system is configured, examples include configuration files or a socket, for example.

Note that a user typically does not access the datastores directly, it is possible to read, but write operations should not be done, since the backend daemon uses a datastore cache.
   
Command-line options
--------------------

The backend have the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       CLICON config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f: -lf<file>.
  -d <dir>        Specify backend plugin directory (default: none)
  -p <dir>        Yang directory path (see CLICON_YANG_DIR)
  -b <dir>        Specify XMLDB database directory
  -F              Run in foreground, do not run as daemon
  -z              Kill other config daemon and exit
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)(default: /usr/var/hello.sock)
  -P <file>       PID filename (default: /usr/local/var/hello.pidfile)
  -1              Run once and then quit (do not wait for events)
  -s <mode>       Specify backend startup mode: none|startup|running|init)
  -c <file>       Load extra xml configuration, but don't commit.
  -g <group>      Client membership required to this group (default: clicon)
  -U <user>       Run backend daemon as this user AND drop privileges permanently
  -y <file>       Load yang spec file (override yang main module)
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

  
Logging and debugging
^^^^^^^^^^^^^^^^^^^^^

At debug, the backend can be run in the foreground and with debug flags:
::

   clixon_backend -FD 1

Logging is done on syslog.  Alternatively, logging can be made on a file using the `-l` option:
::

   clixon_backend -lf<file>

When run in foreground, logging is by default done on both syslog and stderr.

In a debugging mode, it can be useful to run in `once-only` mode, where the backend quits directly after starting up, instead of waiting for events:
::

   clixon_backend -F1D 1

Startup
-------
There are four different backend startup modes selected by the `-s` option. The difference is in how the running state is handled, ie what state the machine is in when you start the daemon and how loading the configuration affects it:

`none`
   Do not touch running state. Typically after crash when running state and db are synched.
`init`
   Initialize running state. Start with a completely clean running state.
`running`
   Commit running db configuration into running state. Typically after reboot if a persistent running db exists.
`startup`
   Commit startup configuration into running state. After reboot when no persistent running db exists.

You use the `-s` to select startup mode:
::
   
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
  
Socket
------
::

   Frontends:   socket  +----------+
   CLI,           |     | backend  |
   netconf      <--->   | daemon   |
   restconf             |          |
                        +----------+

The Clixon backend creates a socket that the frontends can connect to.  Communication is made over this socket using internal Netconf.
The following config options are related to the internal socket:

CLICON_SOCK_FAMILY
   Address family for communicating with clixon_backend. One of: UNIX, IPv4, or IPv6. Can also be set with `-a` command-line option. Default is `UNIX` which denotes a UNIX socket.

CLICON_SOCK
  If family above is AF_UNIX: Unix socket for communicating with clixon_backend. If family is AF_INET: IPv4 address";

CLICON_SOCK_PORT
  Inet socket port for communicating with clixon_backend (only IPv4|IPv6). Default is port `4535`.

CLICON_SOCK_GROUP
  Group membership to access clixon_backend unix socket. Default is `clicon`.


Backend files
-------------

A couple of config options control files related to the backend, as follows:

CLICON_BACKEND_DIR
  Location of backend .so plugins. Load all `.so` plugins in this dir as backend plugins 

CLICON_BACKEND_REGEXP
  Regexp of matching backend plugins in CLICON_BACKEND_DIR. default: `*.so` 

CLICON_BACKEND_PIDFILE
  Process-id file of backend daemon

Plugins
-------

Backend plugins are the "glue" that binds the Clixon system to the
underlying system. The backend invokes *callbacks* in the plugins when
events.

Plugins are written in C as dynamically loaded modules (`.so` files). At startup, the backend daemon looks in the directory pointed to by the config option `CLICON_BACKEND_DIR`, and loads all files with `.so` suffixes from that dir in alphabetical order.

For example, to load all backend plugins from: `/usr/local/lib/example/backend`:
::

   <CLICON_BACKEND_DIR>/usr/local/lib/example/backend</CLICON_BACKEND_DIR>

You can filter which plugins to load by specifying a regular expression. For example, the following will only load backend plugins starting with "example":
::

   <CLICON_BACKEND_REGEXP>^example*.so$</CLICON_BACKEND_REGEXP>
   
Callbacks are registered in two ways:
  - *clixon_plugin_init* : returning an API struct containing a fixed set of callbacks
  - *register functions* : Some specific clixon features use registering functions instead. This allows for multiple callbacks.

Clixon_plugin_init
^^^^^^^^^^^^^^^^^^
A plugin must have a init function called `clixon_plugin_init`. If
this function does not exist, the backend will fail.

The backend calls `clixon_plugin_init` and expects it to return an API
struct (`clixon_plugin_api`) defining all callbacks. The init function may return `NULL` in which case the backend logs this and continues.

Once the plugin is loaded, it awaits callbacks from the backend.

The following callbacks are defined for backend plugins in *clixon_plugin_api*:

init
   Clixon plugin init function, called immediately after plugin is loaded into the backend. The name of the function must be called `clixon_plugin_init`. It returns a struct with the name of the plugin, and all other callback names.
start
   Called when application is started and initialization is complete, but before the application is placed in the background and drop privileges (see `dropping privileges`_), if those operations are requested.
pre_daemon
   Called just before server daemonizes(forks). Not called if in foreground.
daemon
   Called after the server has daemonized and before privileges are dropped. 
exit
   Called just before plugin is unloaded 
extension
  Called at parsing of yang modules containing an extension statement.  A plugin may identify the extension by its name, and perform actions on the yang statement, such as transforming the yang in-memory. A callback is made for every statement, which means that several calls per extension can be made.
reset
  Reset system status
upgrade
  General-purpose upgrade called once when loading the startup datastore
trans_{begin,validate,complete,commit,commit_done,revert,end,abort}
  Transaction callbacks which are invoked for two reasons: validation requests or commits.  These callbacks are further described in `transactions`_ section.

Registered callbacks
^^^^^^^^^^^^^^^^^^^^

A second group of callbacks use register functions. This is a more detailed mechanism than the fixed callbacks described previously, but are only defined to a limited sets of functions:

* ``rpc_callback_register()`` - for user-defined RPC callbacks.
* ``upgrade_callback_register()`` - for upgrading, see :ref:`clixon_upgrade`.

A user may register may register a callback for an incoming RPC, and
that function will be called. 

There may be several callbacks for the same RPC. The order the
callbacks are registered are as follows:

1. plugin_init
2. backend_rpc_init (where system callbacks are registered)
3. plugin_start

Which means if you register a copy-config callback in (1), it will be called *before* the system copy-config callback registered from (2) backend_rpc_init. If you register a copy-config in (3) plugin-start it will be called *after* the system copy-config.

Second, if there are more than one reply (eg ``<rpc-reply/><rpc-reply/>``) only the first reply will be parsed and used by the cli/netconf/restconf clients.

If you want to take the original and modify it, you should therefore register the callback in plugin_start (3) so that your callback will be called after the system RPC. Then you should modify the original reply (not add a new reply).


Transactions
------------
Clixon follows NETCONF in its validate and commit semantics.
Using the CLI or another frontend, you edit the `candidate` configuration, which is first
`validated` for consistency and then `committed` to the `running`
configuration.

A clixon developer writes commit functions to incrementally upgrade a
system state based on configuration changes. Writing commit callbacks
is the core functionality of a clixon system.

The netconf validation and commit operation is implemented in
Clixon by a transaction mechanism, which ensures that user-written
plugin callbacks are invoked atomically and revert on error.  If you
have two plugins, for example, a transaction sequence looks like the
following:
::
   
  Backend   Plugin1    Plugin2
  |          |          |
  +--------->+--------->+ begin
  |          |          |
  +--------->+--------->+ validate
  |          |          |
  +--------->+--------->+ commit
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


Privileges
----------

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
^^^^^^^^^^^^^^^^^^^

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

Note that the unprivileged user must exist on the system, see :ref:`clixon_install`.
 
Drop privileges temporary
^^^^^^^^^^^^^^^^^^^^^^^^^

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
