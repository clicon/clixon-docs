.. _clixon_errors:
.. sectnum::
   :start: 19
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
  
Customized errors
-----------------
Netconf errors, such as returned by the backend on error, may be shown as a log message or CLI output.
A default Netconf to text translation is provided by the system, but
it is possible to customize the message by defining the ``ca_errmsg`` callback.

Customized errors applies to all clixon applications. In particular, logs for the backend and return output in the CLI.

For example, a Netconf message could be::

   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <rpc-error>
         <error-type>rpc</error-type>
         <error-tag>malformed-message</error-tag>
         <error-severity>error</error-severity>
         <error-message>List key m1 length mismatch</error-message>
      </rpc-error>
   </rpc-reply>

The default error message is constructed from the Netconf message as follows::

   rpc malformed-message List key m1 length mismatch

To replace this message with a customized variant, a callback is written as follows in plugin code::

   int
   example_cli_errmsg(clicon_handle h,
                      cxobj        *xerr,
                      cbuf         *cberr)
   {
       int    retval = -1;
       cxobj *x;

       cprintf(cberr, "My error message: ");
       if ((x=xpath_first(xerr, NULL, "//error-message"))!=NULL)
          cprintf(cberr, "%s ", xml_body(x));
       retval = 0;
    done:
       return retval;
   }

   static clixon_plugin_api api = {
     ...
    .ca_errmsg=example_cli_errmsg,
   };

The error message is now::

  My error message: List key m1 length mismatch

The example above is taken from the main example for CLI. Customizing error messages for backend or other applications is similar.
  
Note that the Netconf errors are only a part of all errors. The CLI in particular have error messages (or part of messages) that are not related to NETCONF and are therefore not affected by this translation.

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
Each clixon application has a ``-D <level>`` command-line option to enable debug flags when starting a program. The following flags are defined:

- ``CLIXON_DBG_DEFAULT`` (= 1) Default logs
- ``CLIXON_DBG_MSG``     (= 2) In/out messages and datastore reads
- ``CLIXON_DBG_DETAIL``  (= 4) Detailed logs
- ``CLIXON_DBG_EXTRA``   (= 8) Extra Detailed logs

You can combine flags, so that, for example ``-D 5`` means default + detailed, but no packet debugs.

You can direct the debug logs using the ``-l <option>`` as follows:

- s : syslog
- e : stderr
- o : stdout
- n : none
- f : file, followed by a filename, eg `-f/tmp/foo`

Example::

  clixon_backend -D 5 -f/tmp/log.txt

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
