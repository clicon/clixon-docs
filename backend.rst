.. _clixon_backend:

Backend
=======

::

                            +-------------+
                            | config file |
                            +-------------+
                                   |
                                   v
   Frontends:   netconf +----------+--------+
   CLI,           |     | backend  | plugin |   <--> 
   netconf      <--->   | daemon   |--------+         "System"
   restconf             |          | plugin |   <--> 
                        +----------+--------+
                                   ^
                                   |- XML
               Datastores:         v	                
               +-----------+  +-----------+  +-----------+
               | candidate |  |  running  |  |  startup  |
               +-----------+  +-----------+  +-----------+

The backend daemon is the central Clixon component. The backend has four APIs:

- The *configuration file* read at startup, possibly amended with `-o` options.
- A *NETCONF internal interface* to the frontend clients. This is by default a UNIX domain socket but can also be an IPv4 or IPv6 TCP socket.
- A file interface to the *XML datastores*. The three main datastores are `candidate`, `running` and `startup`. Datastores may use JSON format (experimental).
- Backend *plugins* configure the underlying system with *application-dependent APIs*. This can be a config file or message-passing, for example.

A typical usage is as follows:

1. A user accesses the system via one of the frontend clients: CLI, NETCONF or RESTCONF.
2. The frontend client uses the internal NETCONF interface to access the backend. Usually this is an `edit_config` for modifying values, or a `get_config` to get values. A RESTCONF or CLI command is usually a combination of many NETCONF commands, being more primitive.
3. The backend receives the NETCONF message and reads or modifies one of the XML datastores. Reads are typically done from the `running` datastore, while writes are made on the `candidate` datastore. 
4. After editing the `candidate` datastore, the user may `commit` the changes which triggers a validation and commit callback (see `transactions`_) to the plugins, which in turn may result in an action on the application-specific API:s, such as editing a config file and restaring a process. 

Note that a user typically does not access the datastores directly, it is possible to read, but write operations should not be done, since the backend daemon uses a datastore cache.


Command-line options
--------------------

The backend have the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       CLICON config file
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f: -lf<file>.
  -d <dir>        Specify backend plugin directory (default: none)
  -p <dir>        Yang directory path (see CLICON_YANG_DIR)
  -b <dir>        Specify XMLDB database directory
  -F              Run in foreground, do not run as daemon
  -z              Kill other config daemon and exit
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)(default: /usl/var/hello.sock)
  -P <file>       PID filename (default: /usr/local/var/hello.pidfile)
  -1              Run once and then quit (dont wait for events)
  -s <mode>       Specify backend startup mode: none|startup|running|init)
  -c <file>       Load extra xml configuration, but don't commit.
  -g <group>      Client membership required to this group (default: clicon)
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

Startup modes
^^^^^^^^^^^^^
There are four different backend startup modes selected by the `-s` option. The difference is in how the running state is handled, ie what state the machine is when you start the daemon and how loading the configuration affects it:

- `none` - Do not touch running state. Typically after crash when running state and db are synched.
- `init` - Initialize running state. Start with a completely clean running state.
- `running` - Commit running db configuration into running state. Typically after reboot if a persistent running db exists.
- `startup` - Commit startup configuration into running state. After reboot when no persistent running db exists.

You use the `-s` to select startup mode:
::
   
   clixon_backend -s running

You may also add a default method in the configuration file:
::

   <clixon-config xmlns="http://clicon.org/config">
     ...
     <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE
   </clixon-config>


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

Transactions
------------
Clixon follows NETCONF in its validate and commit semantics.
Using the CLI or another frontend, you edit the `candidate` configuration, which is first
`validated` for consistency and then `committed` to the `running`
configuration.

A clixon developer writes commit functions to incrementaly upgrade a
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

An alternative is to drop privileges temporary and the be able to raise privileges when needed:
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
