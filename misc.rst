.. _clixon_misc:
.. sectnum::
   :start: 19
   :depth: 3

****
Misc
****

These are sections that do not fit into the rest of the document.

Debugging
=========

Debug flags
-----------
Each clixon application has a ``-D <level>`` command-line option to enable debug flags when starting a program. The following flags are defined:
- ``CLIXON_DBG_DEFAULT`` (= 1) Default logs
- ``CLIXON_DBG_MSG``     (= 2) In/out messages and datastore reads
- ``CLIXON_DBG_DETAIL``  (= 4) Detailed logs
- ``CLIXON_DBG_EXTRA``   (= 8) Extra Detailed logs

You can combine flags, so that, for example ``-D 5`` means default + detailed, but no packet debugs.

You can direct the debug logs using the ``-l <option>`` as follows:
- (s)yslog,
- std(e)rr,
- std(o)ut
- (n)one
- (f)ile, followed by a filename

Example::

  clixon_backend -D 1 -f/tmp/log.txt

Change debug
------------

You can also change debug level in run-time in different ways.
For example, using netconf to change debug level in backend::

   echo "<rpc username=\"root\" xmlns=\"urn:ietf:params:xml:ns:netconf:base:1.0\"><debug xmlns=\"http://clicon.org/lib\"><level>1</level></debug></rpc>]]>]]>" | clixon_netconf -q0

In this example, netconf is run using EOM encoding and does not require hello:s.   

Using curl to change debug in backend via the restconf daemon::

   curl -Ssik -X POST -H "Content-Type: application/yang-data+json" http://localhost/restconf/operations/clixon-lib:debug -d '{"clixon-lib:input":{"level":1}}'

Debugger
--------

Enable debugging when configuring (compile-time)::

   ./configure --enable-debug

which includes symbol table info so that you can make breakpoints on functions(output is omitted)::

   > sudo gdb clixon_backend 
   (gdb) run -FD 1 -l e
   Starting program: /usr/local/sbin/clixon_backend -FD 1 -l e
   (gdb) b main
   Breakpoint 1 at 0x55555555bcea: file backend_main.c, line 492.
   (gdb) where
   #0  main (argc=5, argv=0x7fffffffe4e8) at backend_main.c:492

In the example, the backend runs in the foreground(`-F`), runs with debug level `1` and directs the debug messages to stderr.

Valgrind and callgrind
----------------------

Examples of using valgrind for memeory checks::
  
  valgrind --leak-check=full --show-leak-kinds=all clixon_netconf -qf /tmp/myconf.xml -y /tmp/my.yang

Example of using callgrind for profiling::  

  LD_BIND_NOW=y valgrind --tool=callgrind clixon_netconf -qf /tmp/myconf.xml -y /tmp/my.yang
  sudo kcachegrind

Or massif for memory usage::
  
  valgrind --tool=massif clixon_netconf -qf /tmp/myconf.xml -y /tmp/my.yang
  massif-visualizer

Error reporting
===============

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

CLI Errors
----------
There are several types of error messages in the CLI. The first class is "syntax" errors with things like::

  cli> command
  CLI syntax error: "foo": Unknown command
  cli>

These are errors immediately detected by the CLIgen parser and are
internally generated in CLIgen. Errors include command, syntax and type
checking. They are shown on stderr, the CLI continues, without
logging.

A second type of errors are "semantic" errors detected when processing
CLI callbacks. These errors are more heavyweight than syntax errors
and are declared in code using standard clixon `Error call`_.  They
are logged and can be directed to syslog, and are by default printed
on stderr.  The CLI continues after the error message is printed.
Typical places are user callbacks, backend rpc errors, validation,
etc, either system-defined or user-defined callbacks.
They are on the form::

  cli> command
  Nov 15 15:42:56: acl_get_list: 334: Yang error: no ACLs defined
  CLI command error
  cli>

A third class of CLI errors are similar to the previous class but quits the CLI::

  cli> command
  Nov 15 15:42:56: acl_get_list: 334: Yang error: no ACLs defined  
  sh#

These errors are typically due to system functions failing in a fatal way.
  
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

