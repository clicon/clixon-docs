.. _clixon_cli:

CLI
===

The CLI using `CLIgen <https://github.com/olofhagsand/cligen/blob/master/cligen_tutorial.pdf>`_ is a central part of Clixon.

Using the CLI
-------------

Once the backend is started, the easiest way to use Clixon is via the CLI. 

The following example shows how to add an interface in candidate, validate and commit it to running, then look at it (as xml) and finally delete it:
::
   
   clixon_cli -f /usr/local/etc/example.xml 
   cli> set interfaces interface eth9 ?
     description               enabled                   ipv4                     
     ipv6                      link-up-down-trap-enable  type                     
   cli> set interfaces interface eth9 type ex:eth
   cli> validate 
   cli> commit 
   cli> show configuration xml 
   <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
     <interface>
       <name>eth9</name>
       <type>ex:eth</type>
       <enabled>true</enabled>
     </interface>
   </interfaces>
 cli> delete interfaces interface eth9


