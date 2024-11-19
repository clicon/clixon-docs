.. _clixon_errors:
.. sectnum::
   :start: 20
   :depth: 3

****************
Errors and debug
****************

Error reporting
===============

Initialization
--------------
Clixon core applications typically have a command-line option controlling the logs as follows:

  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f as follows: ``-lf<file>``.

An example of a clixon error as it may appear in a syslog::

  Mar 24 10:30:48 Alarik clixon_restconf[3993]: clixon_restconf openssl: 3993 Started

Configure options
-----------------
The following options control the Clixon logging:

``CLICON_LOG_DESTINATION``
   Log destination, a bitmask of syslog, stderr, stdout, and file. Example: ``syslog stderr``

``CLICON_LOG_FILE``
   If log destination includes ``file``, this is the log-file

Note that the configure options are overriden by the command-line argument ``-l``.

C-code
------
In C-code, clixon error and logging is initialized by ``clixon_log_init``::

  clixon_log_init(h, prefix, upto, flags); 

where:

* `prefix`: appears first in the error string
* `upto`: log priority as defined by syslog(3), eg: LOG_DEBUG, LOG_INFO,..
* `flags`: a bitmask of where logs appear, values are: ``CLIXON_LOG_STDERR``, ``_STDOUT``, ``_SYSLOG``, ``_FILE``.


Error call
----------
An error is typically called by ``clixon_err()`` and a return value of ``-1`` as follows::

  clixon_err(category, errno, format, ...)
  return -1;

where:

* `category` is an error "category" including for example "yang", "xml" See `enum clixon_err` for more examples.
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

Error categories
----------------
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

Debugging
=========

Debug flags
-----------
Each clixon application has a ``-D <level>`` command-line option to enable debug flags when starting a program. Levels can be combined and use either symbolic or numerical values. Example:

   clixon_cli -D default -D detail

Levels are separated into `subject-area` and `detail`.

The subject-area levels are the following:

- ``default``   Default logs
- ``msg``       In/out messages and datastore reads
- ``init``      Initialization
- ``xml``       XML logs
- ``xpath``     XPath processing logs
- ``yang``      YANG processing logs
- ``backend``   Backend-specific processing
- ``cli``       CLI-frontend
- ``netconf``   Netconf-frontend
- ``restconf``  Restconf-frontend
- ``snmp``      SNMP-frontend
- ``nacm``      Netconf access control model
- ``proc``      Process handling
- ``datastore`` Datastore handling
- ``event``     Event handling
- ``rpc``       RPC handling
- ``stream``    Notification streams
- ``parse``     XML, YANG, XPath, etc parsing
- ``app``       Application-specific handling, ie any application using clixon can use this
- ``app2``      Application-specific 2
- ``app3``      Application-specific 3
- ``all``       All subject-area flags

The detail area levels are the following:

- ``detail``    Detail logging
- ``detail2``   Extra details
- ``detail3``   Probably more detail than you want

You can combine flags, so that, for example ``-D 5`` means default + detailed, but no packet debugs.  Similarly, some messages require multiple flags, like XML + DETAIL would be ``-D 20``.

You can direct the debug logs using the ``-l <option>`` as follows:

- ``s`` : syslog
- ``e`` : stderr
- ``o`` : stdout
- ``n`` : none
- ``f`` : file, followed by a filename, eg ``-f/tmp/foo``

Example::

  clixon_backend -D 5 -lf/tmp/log.txt

Configure options
-----------------
The following options control the Clixon debugging:

``CLICON_DEBUG``
   Debug flags, a bitmask of debug values. Example: ``msg app2``

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

Customization
=============
Errors, logs and denugs can be customized by plugins via the `ca_errmsg` API.

Customized errors applies to all clixon applications. For example, logs for the backend and return output in the CLI.

The API provides a single function callback which can be used in a various ways. The example shows one simple way as described here.

First define an error message callback as part of the plugin initialization::

   static clixon_plugin_api api = {
     ...
    .ca_errmsg=example_cli_errmsg,
   };

The errmsg callback has many parameters. Some are not always applicable:

  * h : Clixon handle
  * fn : name of source file (only err)
  * line: line of source file (only err)
  * type: log, err or debug (actual types called ``LOG_TYPE_LOG`` etc)
  * category: Error category (see Section `Error categories`_) (only err)
  * suberr: Error number, eg ``errno`` (only err)
  * xerr: XML structure, either NETCONF (for err) or just generic XML (debug, log)
  * format: Format string similar to `printf`
  * ap: Variable argument list assciated with format. Similar to `vprintf`
  * cbmsg: Customized error message as output of the function. If NULL, use regular message.
   
A simple way to replace all error messages would be::

   int
   example_cli_errmsg(clixon_handle        h,
                      const char          *fn,
                      const int            line,
                      enum clixon_log_type type,
                      int                 *category,
                      int                 *suberr,
                      cxobj               *xerr,
                      const char          *format,
                      va_list              ap,
                      cbuf               **cbmsg)
   {
       if (type != LOG_TYPE_ERR)
          return 0;
       if ((*cberr = cbuf_new()) == NULL){
          fprintf(stderr, "cbuf_new: %s\n", strerror(errno));
          return -1;
       }
       cprintf(*cberr, "My error message");
       *category = 0;
       suerr = 0;
       retval = 0;
    done:
       return retval;
   }

All error message are now::

  My error message

Which may not be useful.

More logic needs to be added, for example a more advanced
classification and translation/changing of error messages. Any field
can be used to classify. The `format` string and the `ap` objects may
be translated/converted which is out-of-scope of this document.

Indirection
-----------

The customized callback may also be changed dynamically. The example
shows an extra indirection layer, where a new function is registered
before a call, and deregistered after.

Please see the main example, where `example_cli_errmsg` just
dispatches the call to a dynamic `myerrmsg`.
