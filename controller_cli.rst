.. _clixon_controller_cli:
.. sectnum::
   :start: 9
   :depth: 3

***
Controller CLI
***

Commit
------

The Clixon controller CLI must support a number of different
scenarios:

* Normal commit. Plain and simple commit.

  ::
     
    # commit
    
* Commit witout running service scripts. Just add the service
  configuration, don't touch device configurations which is added by the
  service scripts. This is the equivalent to no-networking.

  ::
     
    # set services test foo
    # set services test device_a bar
    # set services test device_b baz
    # commit no-scripts
    
  
* Commit a single service. Only touch the configuration for a single
  service.

  ::
     
    # set services test foo
    # set services test device_a bar
    # set services test device_b baz
    # commit services test foo

Sync push
---------

* Normal sync push. Simply pushes the configuration to the device(s).

  ::

     > sync push devices foo*
     Error: Sync to device foo1 failed: Bla bla bla
     Error: Sync to device foo9 failed: Bla bla bla

* Dry-run. Push the configuration to the devices and have it
  validated. In the scenario no configuration should be applied, just
  validated. A diff should be presented at the end.

  ::

     > sync push dry-run devices foo*
     Configuration was validated successsfully
     Diff:
     -      description "Clixon example container";
     +      description "Foo bar baz";
     
Sync pull
---------



::
   
   > sync pull devices foo* compare
