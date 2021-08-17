.. _clixon_misc:

****
Misc
****

These are sections that do not fit into the rest of the document.

CLI
===


Differences to CLIgen
---------------------

Clixon adds some features and structure to CLIgen which include:

- A plugin framework for both textual CLI specifications(.cli) and object files (.so)
- Object files contains compiled C functions referenced by callbacks in the CLI specification. For example, in the cli spec command: `a,fn()`, `fn` must exist in the object file as a C function.
- The CLIgen `treename` syntax does not work.
- A CLI specification file is enhanced with the CLIgen variables `CLICON_MODE`, `CLICON_PROMPT`, `CLICON_PLUGIN`.
- Clixon generates a command syntax from the Yang specification that can be referenced as `@datamodel`. This is useful if you do not want to hand-craft CLI syntax for configuration syntax.

Example of `@datamodel` syntax:
::
   
  set    @datamodel, cli_set();
  merge  @datamodel, cli_merge();
  create @datamodel, cli_create();
  show   @datamodel, cli_show_auto("running", "xml");		   

The commands (eg `cli_set`) will be called with the first argument an api-path to the referenced object.


How to deal with large specs
----------------------------
CLIgen is designed to handle large specifications in runtime, but it may be
difficult to handle large specifications from a design perspective.

Here are some techniques and hints on how to reduce the complexity of large CLI specs:

Sub-modes
^^^^^^^^^
The `CLICON_MODE` is used to specify in which modes the syntax in a specific file should be added. For example, if you have major modes `configure` and `operation` you can have a file with commands for only that mode, or files with commands in both, (or in all).

First, lets add a basic set in each:
::
   
  CLICON_MODE="configure";
  show configure;

First, lets add a basic set in each:
::
   
  CLICON_MODE="configure";
  show configure;

Note that CLI command trees are *merged* so that show commands in other files are shown together. Thus, for example:
::

  CLICON_MODE="operation:files";
  show("Show") files("files");

will result in both commands in the operation mode:
::

  > clixon_cli -m operation 
  cli> show <TAB>
    routing      files

but 
::

  > clixon_cli -m operation 
  cli> show <TAB>
    routing      files
  
Sub-trees
^^^^^^^^^
You can also use sub-trees and the the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes:
::
   
  CLICON_MODE="subtree";
  subcommand{
    a, a();
    b, b();
  }

then access that subtree from other modes:
::
   
  CLICON_MODE="configure";
  main @subtree;
  other @subtree,c();

The configure mode will now use the same subtree in two different commands. Additionally, in the `other` command, the callbacks will be overwritten by `c`. That is, if `other a`, or `other b` is called, callback function `c` will be invoked.
  
C-preprocessor
^^^^^^^^^^^^^^

You can also add the C preprocessor as a first step. You can then define macros, include files, etc. Here is an example of a Makefile using cpp:
::
   
   C_CPP    = clispec_example1.cpp clispec_example2.cpp
   C_CLI    = $(C_CPP:.cpp=.cli
   CLIS     = $(C_CLI)
   all:     $(CLIS)
   %.cli : %.cpp
        $(CPP) -P -x assembler-with-cpp $(INCLUDES) -o $@ $<


Error and logs
==============

Initialization
--------------

Clixon core applications typically have a command-line option controlling the logs as follows:

  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f as follows: ``-lf<file>``.

An example of a clixon error as it may appear in a syslog::

  Mar 24 10:30:48 Alarik clixon_restconf[3993]: clixon_restconf openssl: 3993 Started

In C-code, clixon error and logging is initialized by ``clicon_log_init``::

  clicon_log_init(prefix, upto, flags); 

where:

* `prefix`: appears first in the error string
* `upto`: log priority as defined by syslog(3), eg: LOG_DEBUG, LOG_INFO,..
* `flags`: a bitmask of where logs appear, values are: ``CLICON_LOG_STDERR``, ``_STDOUT``, ``_SYSLOG``, ``_FILE``.


  
Error call
----------
An error is typically called by ``clicon_err()`` and a return value of ``-1`` as follows::

  clicon_err(category, errno, format, ...)
  return -1;

where:

* `category` is an error "category" including for example "yang", "xml" See `enum clicon_err` for more examples.
* `errno`  if given, usually errors as given by ``errno.h``
* `format` A variable arg string describing the error.

Specialized error handling
--------------------------
An application can specialize error handling for a specific category by using `clixon_err_cat_reg()` and a log callback. Example::


   /* Clixon error category log callback 
    * @param[in]    handle  Application-specific handle
    * @param[out]   cb      Read log/error string into this buffer
    */
   static int
   my_log_cb(void  *handle,
             cbuf  *cb)
   {
       cprintf(cb, "Myerror");
       return 0;
   }

   main(){
     ...
     /* Register error callback for category */
     clixon_err_cat_reg(OE_SSL, h, openssl_cat_log_cb);

In this example, "Myerror" will appear in the log string.
  

Extensions
==========

Clixon implements YANG extensions.  There are several uses, but one is
to "annotate" a YANG specification with application-specific data that can be used
in plugin code for some reason.

An extension with an argument is introduced in YANG as follows::

   module example-lib {
      namespace "urn:example:lib";
      extension mymode {
         argument annotation;
      }

Such an extension can then be used in YANG declarations in two ways, either
*inline* or *augmented*.

An inlined extension is useful in a YANG module that the designer has
control over and can add extension reference directly in the YANG
specification.

Assume for example that an interface declaration is extended with the extension declared above, as follows::

   module my-interface {
     import example-lib{
       prefix exl;
     }
     container "interfaces" {
       list "interface" {
         exl:mymode "my-interface";
         ...

If you instead use an external YANG, where you cannot edit the YANG
itself, you can use augmentation instead, as follows::

  module my-augments {
   import example-lib{
      prefix exl;
   }
   import ietf-interfaces{
      prefix if;
   }
   augment "/if:interfaces/if:interface"{
      exl:mymode "my-interface";
   }
   ...

When this is done, it is possible to access the extension value in
plugin code and use that value to perform application-specific
actions. For example, assume an XML interface object ``x`` retrieve
the annotation argument::

     char      *value = NULL;
     yang_stmt *y = xml_spec(x);

     if (yang_extension_value(y, "mymode", "urn:example:lib", &value) < 0)
        err;
     if (value != NULL){
        // use extension value
        if (strcmp(value, "my-interface") == 0)
	   ...
	 
A more advanced usage is possible via an extension callback
(``ca_callback``) which is defined for backend, cli, netconf and
restconf plugins. This allows for advanced YANG transformations. Please
consult the main example to see how this could be done.

High availability
=================

This is a brief note on a potential future feature.

Clixon is mainly a stand-alone app tightly coupled to the application/device with "shared fate", that is, if clixon fails, so does the application.

That said, the primary state is the *backend* holding the *configuration database* that can be shared in several ways. This is not implemented in Clixon, but potential implementation strategies include:
  * *Active/standby*: With a standard failure/liveness detection of a master backend, a standby could be started when the master fails using "-s running" (just picking up the state from the failed master). The default cache write-through can be used (``CLICON_DATASTORE_CACHE = cache``). Would suffer from outage during standby boot.
  * *Active/active*: The config-db cache is turned off (``CLICON_DATASTORE_CACHE = nocache``) and two backend process started with a load-balancing in front. Turning the cache off would suffer from performance degradation (and its not currently tested in regression tests). Would also need a failure/liveness detection.

In both cases the *config-db* would be a single-point-of-failure but could be mitigated by a replicated file system, for example.

Regarding clients:
  * the *CLI* and *NETCONF* clients are stateless and spun up on demand.
  * the *RESTCONF* daemon is stateless and can run as multiple instances (with a load-balancer)

Process handling
================

Clixon has a simple internal process handling currently used for :ref:`internal restconf <clixon_restconf>` but can also be used for user applications.
The process data structure has a unique name with pids created for "active" processes. There are three states:

STOPPED
   pid=0,   No process running
RUNNING
   pid is set, Process started and assumed running
EXITING
   pid set, Process is killed by parent but not waited for (eg not 
   
There are three operations that a client can perform on processes:

start
   Start a process
restart
   Restart a process
stop
   Stop a process

State machine
-------------

The state machine for a process is as follows::   

   --> STOPPED --(re)start-->    RUNNING(pid)
          ^   <--1.wait(kill)---   |  ^
	  |                   stop/|  | 
          |                 restart|  | restart
          |                        v  |
          wait(stop) ------- EXITING(dying pid)
	  
The Process struct is created by calling clixon_process_register() with static info such as name, description, namespace, start arguments, etc. Starts in STOPPED state::

       --> STOPPED

On operation "start" or "restart" it gets a pid and goes into RUNNING state::

           STOPPED -- (re)start --> RUNNING(pid)

When running, several things may happen:

     1. It is killed externally: the process gets a SIGCHLD triggers a wait and it goes to STOPPED::
	  
           RUNNING  --sigchld/wait-->  STOPPED

     2. It is stopped due to a rpc or configuration remove: The parent kills the process and enters EXITING waiting for a SIGCHLD that triggers a wait,	therafter it goes to STOPPED::

           RUNNING --stop--> EXITING  --sigchld/wait--> STOPPED
     
     3. It is restarted due to rpc or config change (eg a server is added, a key modified, etc). The parent kills the process and enters EXITING waiting for a SIGCHLD that triggers a wait, therafter a new process is started and it goes to RUNNING with a new pid::

           RUNNING --restart--> EXITING  --sigchld/wait + restart --> RUNNING(pid)


Event notifications
===================

Clixon implements RFC 5277 NETCONF Event Notifications

The main example illustrates an EXAMPLE stream notification that triggers every 5s. First, declare a notification in YANG::

    notification event {
         description "Example notification event.";
         leaf event-class {
           type string;
           description "Event class identifier.";
         }
	 ...

To start a notification stream via netconf::

   <rpc><create-subscription xmlns="urn:ietf:params:xml:ns:netmod:notification"><stream>EXAMPLE</stream></create-subscription></rpc>]]>]]>
   <rpc-reply><ok/></rpc-reply>]]>]]>
   <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0"><eventTime>2019-01-02T10:20:05.929272</eventTime><event><event-class>fault</event-class><reportingEntity><card>Ethernet0</card></reportingEntity><severity>major</severity></event></notification>]]>]]>

This can also be triggered via the CLI::

  clixon_cli -f /usr/local/etc/example.xml
  cli> notify
  cli> event-class fault;
  reportingEntity {
    card Ethernet0;
  }
  severity major;

  cli> no notify
  cli>

Restconf notifications (FCGI only) is also supported, 

Leafrefs
========

Some notes on `YANG leafref <https://www.rfc-editor.org/rfc/rfc7950.html#section-9.9.3>`_ implementation in Clixon, especially as used in `openconfig modules <https://datatracker.ietf.org/doc/html/draft-openconfig-netmod-opstate-01>`_,  which rely heavily on leafrefs.

Typically, a YANG leafref declaration looks something like this::

  container c {  
    leaf x {
      type leafref {
        path "../config/name";   /* "deferring node" */
        require-instance <bool>; /* determines existing deferred node */
      }
    }
    container config {
      leaf name {
        type unit32;             /* "deferred node" */
      }
    }
  }

Types
-----

Consider the YANG example above, the type of ``x`` is the deferred node:s, in this example ``uint32``.
The validation/commit process, as well as the autocli type system and completion handles accordingly.

For example, if the deferred node is a more complex type such as identityref with options "a, b", the completion of "x" will show the options "a,b".

Require-instance
----------------
Assume the yang above have the following two XML trees::

  <c>
    <x>foo</x>
  </c>

and::

  <c>
    <x>foo</x>
    <config>
      <name>foo</name>
    </config>
  </c>
  
The validity of the trees is controlled by the `require-instance property <https://www.rfc-editor.org/rfc/rfc7950.html#section-9.9.3>`_ . According to this semantics:

 - if require-instance is false (or not present) both trees above are valid,
 - if require-instance is true, theupper tree is invalid and the lower is valid

Openconfig lists
----------------

Openconfig has a construct for lists that provides some challenges for clixon. The way many lists are defined is described in the `openconfig draft <https://datatracker.ietf.org/doc/html/draft-openconfig-netmod-opstate-01#section-8.1.2>`_.
where a leaf key is deferring node to a deferred node not yet defined. Since require-instance is false in these constructs, the defer can be unresolved and still valid.
