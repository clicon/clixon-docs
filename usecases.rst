.. _clixon_usecases:

Usecases
========

This section contains usecases which illustrate the flow of data from
a user via Clixon frontends, backend to the underlying system and back.

CLI read
--------

::

                        +----------+
       1           2    | backend  |
     <--->  CLI  <--->  | daemon   |
       5           4    |          |
                        +----------+
                          3  ^
                             |
                       +-----------+
        Datastore:     |  running  |
                       +-----------+


The first usecase illustrates how a retrieval of a configured value from the system is made.

1. The user makes a `show config` call using the hello world example(see :ref:`clixon_quickstart`). In the following examples uses `text` as modifier, and filters on `hello` top-level symbol:
::

   cli> show configuration text hello 
   hello world;

2. The CLI string `show configuration text hello` is translated to internal NETCONF and sent to the backend:
::

   <rpc username="myuser" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
     <get-config>
       <source><candidate/></source>
       <nc:filter nc:type="xpath" nc:select="hello" xmlns="urn:example:hello"/>
     </get-config>
   </rpc>
   
3. The backend receives the internal Netconf message, reads the `running` datastore and filters the output according to the XPath expression.
   
4. The backend returns the filtered output to the client:
::

   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
     <data>
       <hello xmlns="urn:example:hello">
         <world/>
       </hello>
     </data>
   </rpc-reply>

5. The CLI client translates the netconf to "text" output: `hello world;`
   
The user can also retrieve state data. Instead of reading from the running datastore, the backend reads state data either from a plugin, or from itself (if backend internal).

   
CLI write
---------

::

                        +----------+
     1            2     | backend  |
   <--->  CLI   <--->   | daemon   |
                  4     |          |
                        +----------+
                           3 |
                             v	                
                       +-----------+
        Datastore:     | candidate |
                       +-----------+

The figure illustrates the way messages flow through the system. The
numbers illustrate the enumeration below.

When setting a config value, the candidate datastore is modified and the committed to running which triggers a plugin commit transaction:

1. CLI example command:
::

   cli> set hello world
   cli>

2. Internal netconf containing a "replace" operation:
::

   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" username="clicon">
     <edit-config>
       <target><candidate/></target>
       <default-operation>none</default-operation>
       <config>
         <hello xmlns="urn:example:hello">
           <world nc:operation="replace"/>
         </hello>
       </config>
     </edit-config>
   </rpc>

3. The backend modifies the `candidate` datastore. If there was no previous content it will look like the following after the edit:
::

   <config>
     <hello xmlns="urn:example:hello">
       <world/>
     </hello>
   </config>

4. The backend will reply with an OK:
::

   <rpc-reply>
     <ok/>
   </rpc-reply

Commit
------

::
   
                                       3, 
                        +----------+--------+     4
                   1    | backend  | plugin |   <-->  Underlying
         Frontend <-->  | daemon   |--------+         System
                   6    |          | plugin |   <-->   
                        +----------+--------+
                             ^   2       | 5
                             |	         v
                     +-----------+  +-----------+ 
       Datastores:   | candidate |  |  running  | 
                     +-----------+  +-----------+ 

After one, or several, edits, the user can commit the changes to
running which triggers commit callbacks that will actually change the
underlying system. Often, commits are made at once after every edit
(such as RESTCONF operations). In that case, the edit described in the previous sections and commit are made in series by the client.

1. The client sends the commit message (frontend is not specified in this usecase):
:: 

   <rpc username="olof">
     <commit/>
   </rpc>

2. When the backend receives the commit message, it computes the differences between candidate and running datastores, creates a transaction data structure and initiates a transaction.

3. Each plugin in turn gets callbacks to validate the transaction. The plugins verifies that the proposed changes to the system is sound. If not, the commit fails.

4. Each plugin in turn gets callbacks to commit the transaction to the
   underlying system. In this step, the application-dependent API:s are used to push the changes made.

5. If all validation and callbacks succeed, running is replaced with current

6. An OK is returned to the user.
::

   <rpc-reply>
     <ok/>
   </rpc-reply

RESTCONF RPC
------------
::

                                   4   1
                          +----------+--------+    5
        2             3   | backend  | plugin |   <-->  Underlying
  CURL <--> Restconf <--> | daemon   |--------+         System
        7   frontend  6   |          |
                          +----------+
    
A plugin can register an application-dependent RPC, and a client can then access it.

1. A plugin registers `example-rpc`:
::

   rpc_callback_register(h, example_rpc, NULL, "urn:example:clixon", "example");

2. A user makes an RPC call, in this case RESTCONF:
::

   curl -is -X POST -H "Content-Type: application/yang-data+json" -d '{"clixon-example:input":{"x":0}}' http://localhost/restconf/operations/clixon-example:example

3. The restconf client receives the HTTP POST message (via a reverse proxy such as nginx) and translates the JSON to internal NETCONF:
::

   <rpc username="none">
     <example xmlns="urn:example:clixon">
       <x>0</x>
     </example>
   </rpc>

4. The backend receives the Netconf message and calls the registered callback `example_rpc()` in the plugin.

5. The plugin processes the rpc, for example by accessing state in the underlying system

6. The plugin returns a reply which is returned to the restonf client (for example):
::

   <rpc-reply>
     <x xmlns="urn:example:clixon">0</x>
     <y xmlns="urn:example:clixon">42</y>
   </rpc-reply>

7. The restconf client translates the Netconf message to JSON and returns to the client (via a reverse proxy):
::   

   {
     "clixon-example:output":{
        "x":"0",
	"y":"42"
     }
   }
