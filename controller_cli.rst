.. _clixon_controller_cli:
.. sectnum::
   :start: 9
   :depth: 3

***
Controller CLI
***

Overview
========

This section desribes the CLI commands which should be implemed for the Clixon Controller.

Edit
----

Configuration is typically changed by adding or removing services.

  ::

    # set services test foo
    # set services test device_a bar
    # set services test device_b baz

    # set services moo foo
    # set services moo device_a bar
    # set services moo device_b baz


Commit
------

Once the configuration is modified the Clixon controller CLI must
support a number of different scenarios:

* Local commit. Only run service-scripts for the changed services.

  ::

    # show compare
    # commit
    # show compare devices

* Commit witout running service scripts. Just add the service
  configuration, don't touch device configurations which is added by the
  service scripts. This is the equivalent to no-networking.

  ::

    # commit no-scripts


* Commit a single service. Only touch the configuration for a single
  service.

  ::

    # commit services test foo


Re-apply
--------

Add the configuration to the local configuration tree again. This will
run any service-scripts again and generate the necessary configuration.

  ::

     # re-apply

It should also be possible re-apply the configuration for a single service:

  ::

     # re-apply service test


Rollback
--------

If one for some reason want to abandon recently made changes the
rollback command can be used. Rollback will discard any recently made
changes and make sure the local configuration is identical to the last
syncronised configuration.

  ::

     # rollback


Sync push
---------

* Normal sync push. Simply pushes the configuration to the device(s).

  ::

     > sync push devices foo*
     Error: Sync to device foo1 failed: Bla bla bla
     Error: Sync to device foo9 failed: Bla bla bla

     > sync push devicegroup foo
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

* Sync pull will fetch the configuration from the remove device(s). A
  subset of devices can be selected with the device argunent.

::

   > sync pull
   > sync pull devices foo*

* Sync pull compare. It is also possible to compare the local
  configuration with the configuration on the devices without
  replacing the local configuration:

::

   > sync pull devices foo* compare
   foo1:
   -      description "Clixon example container";
   +      description "Foo bar baz";
   foo2:
   -      description "Clixon example container";
   +      description "Baz bar foo";

   > sync pull compare
   test1:
   -      description "Clixon example container";
   +      description "Test test test";
   foo2:
   -      description "Clixon example container";
   +      description "Baz bar foo";


* Sync pull merge. This command will fetch the remove device
  configuration and replace the local Clixon configuration. This can
  be used in an even where there are differences between the two
  configurations and one want to replace the local configuration.

  This can done either for a subset of devices or for all devices.

::

   > sync pull merge
   > sync pull devices foo* merge
