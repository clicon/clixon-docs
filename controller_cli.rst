.. _clixon_controller_cli:
.. sectnum::
   :start: 9
   :depth: 3

***
Controller CLI
***

Commit
========

The Clixon controller CLI must support a number of different
scenarios:

* Normal commit. Plain and simple commit::

    # commit
    
* Commit witout running service scripts. Just add the service
  configuration, don't touch device configurations which is added by the
  service scripts::

    # set services test foo
    # set services test device_a bar
    # set services test device_b baz
    # commit no-scripts
    
  
* Commit a single service. Only touch the configuration for a single
  service::

    # set services test foo
    # set services test device_a bar
    # set services test device_b baz
    # commit services test foo

