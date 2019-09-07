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

- The configuration file read at startup, possibly amended with `-o` options.
- A NETCONF interface to the frontend clients. This is by default a UNIX domain socket but can also be an IPv4 or IPv6 TCP socket.
- A file interface to the XML datastores. The three main datastores are `candidate`, `running` and `startup`. Datastores may use JSON format (experimental).
- Backend plugins configure the underlying system with application-dependent APIs. This can be a config file or message-passing, for example.


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


Transactions
------------
Clixon follows NETCONF in its validate and commit semantics.
You edit a `candidate` configuration, which is first
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

 
