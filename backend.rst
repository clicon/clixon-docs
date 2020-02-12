.. _clixon_backend:

Backend
=======

::

                           +---------------+
                           | configuration |
                           +---------------+
                                   |- XML
                                   v
   Frontends:   Netconf +----------+--------+
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
There are four different backend startup modes selected by the `-s` option. The difference is in how the running state is handled, ie what state the machine is when you start the daemon and how loading the configuration affects it:

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


Module-state
^^^^^^^^^^^^

Clixon has the ability to store Yang module-state information according to
RFC7895 in the datastores. Including yang module-state in the
datastores is enabled by the following entry in the Clixon
configuration:
::

   <CLICON_XMLDB_MODSTATE>true</CLICON_XMLDB_MODSTATE>

If the datastore does not contain module-state info, no
module-specific upgrades can be made, only the general-purpose upgrade
is available.

A backend does not perform detection of mismatching XML/Yang if:

1. The datastore was saved in a pre-3.10 system 
2. `CLICON_XMLDB_MODSTATE` was not enabled when saving the file
3. The backend configuration does not have `CLICON_XMLDB_MODSTATE` enabled.

Note that the module-state detection is independent of the other steps
of the startup operation: syntax errors, validation checks, failsafe mode, etc,
are still made, even though module-state detection does not occur.

Note also that a backend with `CLICON_XMLDB_MODSTATE` disabled
will silently ignore the module state.

Example of a (simplified) datastore with Yang module-state:
::
   
   <config>
     <a1 xmlns="urn:example:a">some text</a1>
     <modules-state xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
       <module-set-id>42</module-set-id>
       <module>
         <name>A</name>
         <revision>2019-01-01</revision>
         <namespace>urn:example:a</namespace>
       </module>
     </modules-state>
   </config>

Note that the module-state is not available to the user, the backend
datastore handler strips the module-state info. It is only shown in
the datastore itself.


   
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
events occur. 

Plugins are written in C as dynamically loaded modules (`.so` files). At startup, the backend daemon looks in the directory pointed to by the config option `CLICON_BACKEND_DIR`, and loads all files with `.so` suffixes from that dir in alphabetical order.

For example, to load all backend plugins from: `/usr/local/lib/example/backend`:
::

   <CLICON_BACKEND_DIR>/usr/local/lib/example/backend</CLICON_BACKEND_DIR>

You can filter which plugins to load by specifying a regular expression. For example, the following will only load backend plugins starting with "example":
::

   <CLICON_BACKEND_REGEXP>^example*.so$</CLICON_BACKEND_REGEXP>
   
A plugin must have a init function called `clixon_plugin_init`. If
this function does not exist, the backend will fail.

The backend calls `clixon_plugin_init` and expects it to return an API
struct defining all callbacks. The init function may return `NULL` in
which case the backend logs this and continues.

Once the plugin is loaded, it awaits callbacks from the backend.

Callbacks
^^^^^^^^^

The following callbacks are defined for backend plugins:

init
   Clixon plugin init function, called immediately after plugin is loaded into the backend. The name of the function must be called `clixon_plugin_init`. It returns a struct with the name of the plugin, and all other callback names.
start
   Called when application is "started", (almost) all initialization is complete and daemon is in the background. If daemon privileges are dropped (see `dropping privileges`_) this callback is called *before* privileges are dropped.
exit
   Called just before plugin is unloaded 
extension
  Called at parsing of yang modules containing an extension statement.  A plugin may identify the extension by its name, and perform actions on the yang statement, such as transforming the yang in-memory. A callback is made for every statement, which means that several calls per extension can be made.
reset
  Reset system status
upgrade
  General-purpose upgrade called once when loading the startup datastore
trans_begin, trans_validate, trans_complete, trans_commit, trans_revert, trans_end, trans_abort
  Transaction callbacks which are invoked for two reasons: validation requests or commits.  These callbacks are further described in `transactions`_ section.



Upgrade
-------

Clixon has several variant of update callbacks:

* General-purpose datastore upgrade.
* Module-specific manual upgrade
* Experimental automatic module upgrade

This section describes how a user can write upgrade callbacks for data
modeled by outdated Yang models. The scenario is a Clixon system with
a set of current yang models that loads a datastore with old or even
obsolete data.

General-purpose
^^^^^^^^^^^^^^^

A plugin registers a `ca_datastore_upgrade` function which gets called
once on startup. This upgrade should be seen as a generic wrapper
function to basic repair or upgrade of existing datastores. The
module-specific upgrade callbacks are more fine-grained.

The general-purpose upgrade callback is usable if module-state is not
available, or actions such as the following need to be done in the whole datastore:

 * Remove or rename nodes
 * Rename namespaces
 * Add nodes

A recommended method, as shown in the example, is to make a pattern
matching using XPath and then perform actions on the resulting nodes.

Example:
::

  static clixon_plugin_api api = {
     ...
     .ca_datastore_repair=example_upgrade
  };
  
  /*! General-purpose datastore upgrade callback called once on startup
   * @param[in] h    Clicon handle
   * @param[in] db   Name of datastore, eg "running", "startup" or "tmp"
   * @param[in] xt   XML tree. Upgrade this "in place"
   * @param[in] msd  Info on datastore module-state, if any
   */
  int
  example_upgrade(clicon_handle h,
                  char         *db,
		  cxobj        *xt,
		  modstate_diff_t *msd)
  {
      cxobj   **xvec = NULL;   /* vector of result nodes */
      size_t    xlen; 
      cvec     *nsc = NULL;    /* Canonical namespace */
      int       i;
      
      /* Skip other than startup datastore */
      if (strcmp(db, "startup") != 0) 
         return 0;
      /* Skip if there is proper module-state in datastore */
      if (msd->md_status) 
         return 0;
      /* Get canonical namespaces for using "normalized" prefixes */      
      if (xml_nsctx_yangspec(yspec, &nsc) < 0)
         err;
      /* Get all xml nodes matching path */
      if (xpath_vec(xt, nsc, "/a:x/a:y", &xvec, &xlen) < 0) 
         err;
      /* Iterate throughnodes and remove them */
      for (i=0; i<xlen; i++){
         if (xml_purge(xvec[i]) < 0)
	    err;
      }
  }

The example above first checks whether it is the `startup` datastore
and that it does not contain module-state. It then matches all nodes
matching the XPath `/a:x/a:y` using canonical prefixes, which are then
deleted.
  
Module-specific upgrade
^^^^^^^^^^^^^^^^^^^^^^^
Module-specific upgrade is only available if `module-state`_ is enabled.

If the module-state of the startup configuration does not match the
module-state of the backend daemon, a set of module-.specifc upgrade callbacks are
made. This allows a user to program upgrade funtions in the backend
plugins to automatically upgrade the XML to the current version.

A user registers upgrade callbacks based on module and revision
ranges. A user can register many callbacks, or choose wildcards.
When an upgrade occurs, the callbacks will be called if they match the
module and revision ranges registered.

Different strategies can be used for upgrade functions. One
coarse-grained method is to register a single callback to handle all
modules and all revisions. 

A fine-grained method is to register a separate _stepwise_ upgrade
callback per module and revision range that will be called in a series.

Registering a callback
^^^^^^^^^^^^^^^^^^^^^^
A user registers upgrade callbacks in the backend `clixon_plugin_init()` function. The signature of upgrade callback is as follows:
::
   
  upgrade_callback_register(h, cb, namespace, from, revision, arg);

where:

* `h` is the Clicon handle,
* `cb` is the name of the callback function,
* `namespace` defines a Yang module. NULL denotes all modules. Note that module `name` is not used (XML uses namespace, whereas JSON uses name, XML is more common).
* `from` is a revision date indicated an optional start date of the upgrade. This allows for defining a partial upgrade. It can also be `0` to denote any version.
* `revision` is the revision date "to" where the upgrade is made. It is either the same revision as the Clixon system module, or an older version. In the latter case, you can provide another upgrade callback to the most recent revision. 
* `arg` is a user defined argument which can be passed to the callback.

One example of registering a "catch-all" upgrade: 
::

   upgrade_callback_register(h, xml_changelog_upgrade, NULL, 0, 0, NULL);


Another example are fine-grained stepwise upgrades of a single module [upgrade example](#example-upgrade):
::
   
   upgrade_callback_register(h, upgrade_2016, "urn:example:interfaces",
                             20140508, 20160101, NULL);
   upgrade_callback_register(h, upgrade_2018, "urn:example:interfaces",
                             20160101, 20180220, NULL);

      20140508       20160101       20180220
   ------+--------------+--------------+-------->
         upgrade_2016   upgrade_2018

In the latter case, the first callback upgrades
from revision 2014-05-08 to 2016-01-01; while the second makes upgrades from
2016-01-01 to 2018-02-20. These are run in series.

Upgrade callback
^^^^^^^^^^^^^^^^
When Clixon loads a startup datastore with outdated modules, the matching
upgrade callbacks will be called.

Note the following:

* Upgrade callbacks _will_ _not_ be called for data that is up-to-date with the current system
* Upgrade callbacks _will_ _not_ be called if there is no module-state in the datastore, or if module-state support is disabled.
* Upgrade callbacks _will_ be called if the datastore contains a version of a module that is older than the module loaded in Clixon.
* Upgrade callbacks _will_ also be called if the datastore contains a version of a module that is not present in Clixon - an obsolete module.

Re-using the previous stepwise example, if a datastore is loaded based on revision 20140508 by a system supporting revision 2018-02-20, the following two callbacks are made:
::

  upgrade_2016(h, <xml>, "urn:example:interfaces", 20140508, 20180220, NULL, cbret);
  upgrade_2018(h, <xml>, "urn:example:interfaces", 20140508, 20180220, NULL, cbret);

Note that the example shown is a template for an upgrade function. It
gets the nodes of an yang module given by `namespace` and the
(outdated) `from` revision, and iterates through them. 

If no action is made by the upgrade calback, and thus the XML is not
upgraded, the next step is XML/Yang validation.

An out-dated XML may still pass validation and the system will go up
in normal state.

However, if the validation fails, the backend will try to enter the
failsafe mode so that the user may perform manual upgarding of the
configuration.

Example upgrade
^^^^^^^^^^^^^^^

The example and  shows the code for upgrading of an interface module. The example is inspired by the ietf-interfaces module that made a subset of the upgrades shown in the examples.

The code is split in two steps. The `upgrade_2016` callback does the following transforms:

  * Move /if:interfaces-state/if:interface/if:admin-status to /if:interfaces/if:interface/
  * Move /if:interfaces-state/if:interface/if:statistics to if:interfaces/if:interface/
  * Rename /interfaces/interface/description to /interfaces/interface/descr

The `upgrade_2018` callback does the following transforms:
  * Delete /if:interfaces-state
  * Wrap /interfaces/interface/descr to /interfaces/interface/docs/descr
  * Change type /interfaces/interface/statistics/in-octets to decimal64 and divide all values with 1000

Please consult the `upgrade_2016` and `upgrade_2018` functions in [the
example](../example/example_backend.c) and
[test](../test/test_upgrade_interfaces.sh) for more details.

Extra XML
^^^^^^^^^
If the Yang validation succeeds and the startup configuration has been committed to the running database, a user may add "extra" XML.

There are two ways to add extra XML to running database after start. Note that this XML is "merged" into running, not "committed".

The first way is via a file. Assume you want to add this xml:
::

  <config>
    <x xmlns="urn:example:clixon">extra</x>
  </config>

You add this via the -c option:
::
   
   clixon_backend ... -c extra.xml

The second way is by programming the plugin_reset() in the backend
plugin. The example code contains an example on how to do this (see
plugin_reset() in example_backend.c).

The extra-xml feature is not available if startup mode is `none`. It
will also not occur in failsafe mode.


Failsafe mode
^^^^^^^^^^^^^
If the startup fails, the backend looks for a `failsafe` configuration
in `CLICON_XMLDB_DIR/failsafe_db`. If such a config is not found, the
backend terminates. In this mode, running and startup mode should be
unchanged.

If the failsafe is found, the failsafe config is loaded and
committed into the running db.

If the startup mode was `startup`, the `startup` database will
contain syntax errors or invalidated XML.

If the startup mode was `running`, the the `tmp` database will contain
syntax errors or invalidated XML.


Repair
^^^^^^
If the system is in failsafe mode (or fails to start), a user can
repair a broken configuration and then restart the backend. This can
be done out-of-band by editing the startup db and then restarting
clixon.

In some circumstances, it is also possible to repair the startup
configuration on-line without restarting the backend. This section
shows how to repair a startup datastore on-line.

However, on-line repair _cannot_ be made in the following circumstances:

* The broken configuration contains syntactic errors - the system cannot parse the XML.
* The startup mode is `running`. In this case, the broken config is in the `tmp` datastore that is not a recognized Netconf datastore, and has to be accessed out-of-band.
* Netconf must be used. Restconf cannot separately access the different datastores.

First, copy the (broken) startup config to candidate. This is necessary since you cannot make `edit-config` calls to the startup db:
::
   
  <rpc>
    <copy-config>
      <source><startup/></source>
      <target><candidate/></target>
    </copy-config>
  </rpc>

You can now edit the XML in candidate. However, there are some restrictions on the edit commands. For example, you cannot access invalid XML (eg that does not have a corresponding module) via the edit-config operation.
For example, assume `x` is obsolete syntax, then this is _not_ accepted:
::
   
  <rpc>
    <edit-config>
      <target><candidate/></target>
      <config>
        <x xmlns="example" operation='delete'/>
      </config>
    </edit-config>
  </rpc>

Instead, assuming `y` is a valid syntax, the following operation is allowed since `x` is not explicitly accessed:
::
   
  <rpc>
    <edit-config>
      <target><candidate/></target>
      <config operation='replace'>
        <y xmlns="example"/>
      </config>
    </edit-config>
  </rpc>

Finally, the candidate is validate and committed:
::
   
  <rpc>
    <commit/>
  </rpc>

The example shown in this Section is also available as a regression [test script](../test/test_upgrade_repair.sh).


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
