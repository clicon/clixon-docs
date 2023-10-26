.. _clixon_misc:
.. sectnum::
   :start: 20
   :depth: 3

****
Misc
****

These are sections that do not fit into the rest of the document.

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

