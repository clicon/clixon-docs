.. _clixon_misc:
.. sectnum::
   :start: 22
   :depth: 3

****
Misc
****

These are sections that do not fit into the rest of the document.

High availability
=================
This is a brief note on a potential future feature.

Clixon is mainly a stand-alone app tightly coupled to the application/device with "shared fate", that is, if clixon fails, so does the application.

That said, the primary state is the *backend* holding the *configuration datastore* that can be shared in several ways. This is not implemented in Clixon, but potential implementation strategies include:
  * *Active/standby*: With a standard failure/liveness detection of a master backend, a standby could be started when the master fails using "-s running" (just picking up the state from the failed master). The cache write-through mechanism supports this. Would suffer from outage during standby boot. One way already possible today is using existing detection/restart mechanisms in SystemD for example.
  * *Active/active*: Would need an active cache coherence algorithm. Would also need a failure/liveness detection.

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

Formats
=======

Other formats
-------------
While only XML and JSON are currently supported as datastore formats, Clixon also supports `CLI` and `TEXT` formats for printing, and saving and loading files.

The main example contains example code showing how to load and save a config using other formats.

Example of showing a config as XML, JSON, TEXT and CLI::

   cli> show configuration xml
   <table xmlns="urn:example:clixon">
      <parameter>
         <name>a</name>
         <value>17</value>
      </parameter>
      <parameter>
         <name>b</name>
         <value>99</value>
      </parameter>
   </table>
   cli> show configuration json
   {
     "clixon-example:table": {
       "parameter": [
         {
           "name": "a",
           "value": "17"
         },
         {
           "name": "b",
           "value": "99"
         }
       ]
     }
   }
   cli> show configuration text
   clixon-example:table {
       parameter a {
           value 17;
       }
       parameter b {
           value 99;
       }
   }
   cli> show configuration cli
   set table parameter a
   set table parameter a value 17
   set table parameter b
   set table parameter b value 99

Save and load a file using TEXT::

   cli> save foo.txt text
   cli> load foo.txt replace text

Internal C API
^^^^^^^^^^^^^^
CLI show and save commands uses an internal API for print, save and load of the formats. Such CLI functions include: `cli_show_config`, `cli_pagination`, `load_config_file`, `save_config_file`.

The following internal C API is available for output:

* XML: ``clixon_xml2file()`` and ``clixon_xml2cbuf()`` to file and memory respectively.
* JSON: ``clixon_json2file()`` and ``clixon_json2cbuf()``
* CLI: ``clixon_cli2file()``
* TEXT: ``clixon_txt2file()``

The arguments of these functions are similar with some local variance. For example::

   int
   clixon_xml2file(FILE             *f,
                   cxobj            *xn,
		   int               level,
		   int               pretty,
		   clicon_output_cb *fn,
		   int               skiptop,
		   int               autocliext)

where:

* `f` is the output stream (such as `stdout`)
* `xn` is the top-level XML node
* `level` is indentation level to start with, normally `0`
* `pretty` makes the output indented and use newlines
* `fn` is the output function to use. `NULL` means `fprintf`, `cligen_output` is used for scrolling in CLI
* `skiproot` only prints the children by skipping the top-level XML node `xn`
* `autocliext` Set if you want to activate autocli extensions (eg `hide` extensions)
