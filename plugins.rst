.. _clixon_plugins:

Plugins
=======

Plugins are the "glue" that binds the underlying system to Clixon and where
application semantics is added by an application developer.

The backend, CLI, Restconf and Netconf applications may use plugins,
although the main use of plugins are the :ref:`clixon_backend` plugins.

Plugins are written in C as dynamically loaded modules (``.so``
files). At startup, the application looks in the assigned directory and loads all files with ``.so`` suffixes from that dir in *alphabetical* order.

Plugins are located as follows:

* Backend plugins are are in ``CLICON_BACKEND_DIR``
* CLI plugins are are in ``CLICON_CLI_DIR``
* NETCONF plugins are in ``CLICON_NETCONF_DIR``
* RESTCONF plugins are in ``CLICON_CLI_DIR``;
  
For example, to load all backend plugins from: ``/usr/local/lib/example/backend``:
::

   <CLICON_BACKEND_DIR>/usr/local/lib/example/backend</CLICON_BACKEND_DIR>

Callbacks are registered in two ways:
  - *clixon_plugin_init* : Return an API struct containing a fixed set of callbacks
  - *register functions* : Register callback functions for more complex interaction

Further, CLI callbacks as described in :ref:`clixon_cli` are special
in the way that they are invoked from CLIgen specifications, not from
the application itself.
    
Clixon_plugin_init
------------------
On startup, the application loads each plugin and calls
``clixon_plugin_init`` in that plugin.   The function is expected to return an API
struct ``clixon_plugin_api`` defining a set of static callbacks. The init
function may return ``NULL`` in which case it is logged and ignored.

Once the plugin is loaded, it awaits callbacks from the application.

An example of a minimal plugin is as follows::

  static clixon_plugin_api api = {
    "example",                              /* Plugin name */
    clixon_plugin_init,                     /* init - must be called clixon_plugin_init */
    example_start,                          /* start */
    example_exit,                           /* exit */
    example_extension,                      /* yang extensions */
    .ca_daemon=example_daemon,              /* Plugin daemonized () (backend only) */
  };
  
  clixon_plugin_api *
  clixon_plugin_init(clicon_handle h)
  {
     return &api;
  }

First the ``clixon_plugin_api`` struct defines the callbacks. There are two blockx of callbacks:

* Common callbacks for backend, cli, netconf, and restconf
* Application-specific callbacks. In the example, only ``ca_daemon`` is specific to backends. By far, the backend have most specific callbacks.

Second, the ``clixon_plugin_init()`` function is defined and (at a minimum) returns the struct.
  
The following callbacks are common plugins for all Clixon applications:

init
   Clixon plugin init function, called immediately after plugin is loaded into the application. The name of the function must be called ``clixon_plugin_init``. It returns a struct with the name of the plugin, and all other callback names.
start
   Called when application is started and initialization is complete, but before the application is placed in the background and privileges are dropped (if applicable)
exit
   Called just before plugin is unloaded at exit
extension
  Called at parsing of yang modules containing an extension statement.  A plugin may identify the extension by its name, and perform actions on the yang statement, such as transforming the yang in-memory. A callback is made for every statement, which means that several calls per extension can be made. See :ref:`clixon_misc` and the main example on how to use extensions.

See :ref:`clixon_backend` for backend specific callbacks, as well as corresponding manual sections for netconf/restconf/cli callbacks.

Registering callbacks
---------------------

A second group of callbacks use register functions. This is a more detailed mechanism than the fixed callbacks described previously, but are only defined to a limited sets of functions:

* ``rpc_callback_register()`` - for user-defined RPC callbacks. Applicable for NETCONF, RESTCONF and backend.
* ``upgrade_callback_register()`` - for upgrading, see :ref:`clixon_upgrade`. Applicable only for backend.
* ``clixon_pagination_cb_register()`` - for pagination, as described in :ref:`clixon_pagination`. Applicable only for backend.

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

Example: RPC callback
^^^^^^^^^^^^^^^^^^^^^

This example shows how to define a new RPC in YANG for the backend, register a callback function in C, read and write a parameter.
It is revised slightly from the main example.

YANG::

    module clixon-example {
      namespace "urn:example:clixon";
      ...
      rpc example {
          input {
	      leaf x {
		type string;
		...
 	  output {
	      leaf y {
		type string;
                ...
		
Register RPC in clixon_plugin_init()::		

    clixon_plugin_api *clixon_plugin_init(clicon_handle h)
    {
       ...
       rpc_callback_register(h, example_rpc, NULL, "urn:example:clixon", "example");

Callback function reading value input x, modifying value and writing it as output value y::

   static int 
   example_rpc(clicon_handle h,            /* Clicon handle */
               cxobj        *xe,           /* Request: <rpc><xn></rpc> */
	       cbuf         *cbret,        /* Reply eg <rpc-reply>... */
	       void         *arg,          /* client_entry */
	       void         *regarg)       /* Argument given at register */
   {
       char *val;
       val = xml_find_body(xe, "x");       /* Read x value of incoming rpc */
       cprintf(cbret, "<rpc-reply xmlns=\"%s\">", NETCONF_BASE_NAMESPACE);
       val[0]++;                           /* Increment first char */
       /* Construct reply */
       cprintf(cbret, "<y xmlns=\"urn:example:clixon\">%s</y>", val);
       cprintf(cbret, "</rpc-reply>");

Result netconf session::

  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="42">
     <example xmlns="urn:example:clixon">
        <x>42</x>
     </example>
  </rpc>]]>]]>
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="42">
     <y xmlns="urn:example:clixon">52</y>
  </rpc-reply>]]>]]>

Plugin callback guidelines
--------------------------

.. note::
        This information is important to understand for the stability of clixon

The Clixon programs run as non-blocking `single-threaded`
applications. It calls functions from within dynamically loaded modules.
The callback code must be written with this programming model in mind.
The behavior of the callback directly impacts the behavior of the caller
and the whole system.

The most serious effect is when crash within a callback happens. This
will cause the whole program to crash.

A more subtle problem is the environment of the program. Clixon will
configure the environment, and it expects that the callback will return
with the exact same environment intact. If you change a signal handler,
a terminal configuration, etc. `you must restore the state as it was
on entry`. Failure to do this can cause problems that are difficult to
isolate and fix.

A list of things to watch out for (but not complete!):

  * a crash in the plugin
  * change of signal behaviour, such as blocking or assigning signal handlers
  * change of terminal settings (for CLI callbacks)
  * change of process privileges
  * asynchronous calls
  * If you fork or create threads, ensure the main program continues uninterrupted

The following config option is related to checking callbacks:

CLICON_PLUGIN_CALLBACK_CHECK
   Enable check of resources before and after each callback. Checks are currently limited to signal and terminal settings

  
