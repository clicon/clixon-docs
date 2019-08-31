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

