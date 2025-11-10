.. _clixon_datastore:
.. sectnum::
   :start: 8
   :depth: 3

*********
Datastore
*********

 .. image:: datastore1.jpg
   :width: 70%

Clixon configuration datastores follow the Netconf model (from `RFC 6241 <http://rfc-editor.org/rfc/rfc6241.txt>`_):

`Candidate`
   A configuration datastore that can be manipulated without impacting the device's current configuration and that can be committed to the running configuration datastore.
`Running`
   A configuration datastore holding the complete configuration currently active on the device.
`Startup`
   The configuration datastore holding the configuration loaded by the device when it boots. Only present on devices that separate the startup configuration datastore from the running configuration datastore.

There are also other datastores, Clixon is not limited to the three datastores above. For example:

`Tmp`
   The tmp datastore appears in several cases as an intermediate datastore.
`Rollback`
   If the confirmed-commit feature is enabled, the rollback datastore holds the running datastore as it existed before the confirm commit. If a cancel or timeout occurs, the rollback datastore is used to revert to.

Datastore files
===============

Clixon datastores are stored in regular files, by default one file per datastore.

Options
-------
Datastore files are controlled by the following options:

CLICON_XMLDB_DIR
   Mandatory directory where datastores are placed.

CLICON_XMLDB_MULTI
   Split configure datastore into multiple sub files. Default is single file. Only applicable for YANG schema mounts

Datastores files are only accessible by the user that starts the
backend. Typically this is `root`, but if the backend is started as a
non-privileged user, or if privileges are dropped (see :ref:`Backend
section<clixon_backend>`) this may be another user, such as in the
following example where `clicon` is used: ::

   sh> ls -l /usr/local/var/example
   -rwx------ 1 clicon clicon   0 sep 15 17:02 candidate_db
   -rwx------ 1 clicon clicon   0 sep 15 17:02 running_db
   -rwx------ 1 clicon clicon   0 sep 14 18:12 startup_db

Split datastores
----------------
By default, a datastore is a single file. However, it is possible to
split the store into multiple files in a `db.d/`  directory. For example::

  > ls -1 /usr/local/var/multi/running.d/
  0.xml
  35819a66.xml
  762894da.xml
  ...

where:

* `0.xml` is a top-level datastore
* `35819a66.xml` is a sub-store linked from `0.xml`

There are some limitations to split datastores:

* XML is the only format supported by split datastores.
* One-level of splits are allowed, that is, a sub-store may not link ot another sub-store

Backward-compatibility is ensured by reading ``<db>_db`` if ``<db>.d/0.xml`` is not present at startup.

The sub filenames are SHA1 digests of the XPath to the XML node where the split is made.

A typical example of splitting a datastore is in combination with RFC 8528 YANG Schema mounts.

Mechanism
^^^^^^^^^
The YANG `cl:xmldb-split` extension and the `cl:link` attribute is used to define how datastores are split as the following example illustrates.

Assume a YANG specification with nodes `a` and `b`::

    container a {
       cl:xmldb-split {
          description "Multi-XMLDB: split datastore here";
       }
       container b {
           description "Will be in separate file";
       }
    }

If the YANG above is instantiated to, for example, ``<a><b>foo</b></a>``, this will result in the following two datastores::

   0.xml:
      <config>
         <a xmlns:cl="http://clicon.org/lib" cl:link="35819a66.xml"/>
      </config>

   35819a66.xml:
      <config>
         <b>foo</b>
      </config>

File formats
============
By default, the datastore files use pretty-printed XML, with the top-symbol `config`. The following is an example of a valid datastore::

   <config>
     <hello xmlns="urn:example:hello">
       <world/>
     </hello>
   </config>

The format of the datastores can be changed using the following options:

`CLICON_XMLDB_FORMAT`
   Datastore format. `xml` is the primary alternative. `json` is also available
`CLICON_XMLDB_PRETTY`
   XMLDB datastore pretty print. The default value is `true`, which inserts spaces and line-feeds making the XML/JSON human readable. If false, the XML/JSON is more compact.

Limitations:
* Format settings apply to all datastores
* `xml` and `json` are allowed for single datastores
* `xml` is allowed for split datstores

Caching
=======

Clixon stores datastore content in an in-memory write-through cache
managed by the Clixon backend.
As soon as data is modified in-mem, a write is made to file.
Reads from file is made only on startup, or more precisley, if the cache is empty.
Modifications by an external part of the file is only read by the backend on startup.

Candidate
=========
The candidate datastore can be manipulated without impacting the current, running
configuration of a device, and can be committed to the running datastore.

Options
-------
CLICON_AUTOLOCK
   Get exclusive lock automatically by edit-config and copy-config and released by discard and commit.

Shared candidate
----------------
By default and as defined in `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_, the candidate datastore
is `shared`, meaning that other clients can modify the candidate
configuration simultaneously.

Exclusive access
----------------
An exclusive lock on the shared candidate prohibits other clients to make changes while the client is editing.

There are two ways to obtain an exclusive lock, either explicitly or implicitly:

* An `explicit` lock is made the NETCONF ``lock`` operation or by corresponding CLI operation.
* `Implicit` locks are enabled by setting `<CLICON_AUTOLOCK>true</CLICON_AUTOLOCK>` in the configuration file.

An implicit lock is obtained by edit-config and copy-config and released by discard and commit.

Locking is not available in RESTCONF.

Private candidate
-----------------
Clixon implements private candidate as defined in `NETCONF and RESTCONF Private Candidate Datastores <https://datatracker.ietf.org/doc/html/draft-ietf-netconf-privcand-08>`_.

To enable private candidate the CLICON_XMLDB_PRIVATE_CANDIDATE option must be set to true and the feature  ``ietf-netconf-private-candidate:private-candidate`` must be added to the Clixon configuration.::

      <CLICON_FEATURE>ietf-netconf-private-candidate:private-candidate</CLICON_FEATURE>
      <CLICON_XMLDB_PRIVATE_CANDIDATE>true</CLICON_XMLDB_PRIVATE_CANDIDATE>

All NETCONF clients must send the capability: ``urn:ietf:params:netconf:capability:private-candidate:1.0`` when connecting, otherwise the session will be terminated by Clixon.

When a conflict is detected by the ``<update>`` operation or implicitly during commit, the error message contains the type of conflict and the xpath to the object of the first conflict encountered. ::

    <rpc-error>
        <error-type>application</error-type>
        <error-tag>operation-failed</error-tag>
        <error-severity>error</error-severity>
        <error-message>Conflict occured: Cannot change node value, node is removed: xpath0: /interfaces/interface[name="intf_one"]</error-message>
    </rpc-error>

A conflict can be handled by discarding all changes made to the private candidate using the ``<discard-changes>`` command and then retrieving the latest configuration from the running configuration to the private candidate using ``<update>``. ::

    <discard-changes/>
    <update xmlns="urn:ietf:params:xml:ns:netconf:private-candidate:1.0"/>

On a successful ``<commit>``, the running configuration is updated with the changes from the private candidate, after which the private candidate is deleted. A new private candidate is created from the running configuration on the first call of any operation involving the candidate, e.g. ``<edit-config>``.

**CLI example**

The following commands may be specified to manage private candidates using the cli ::

    show("Show configuration") {
        candidate("Show private candidate configuration"), cli_show_auto_mode("candidate", "text", true, false);
        running("Show running configuration"), cli_show_auto_mode("running", "text", true, false);
     }
    compare("Compare private candidate and running databases"), compare_dbs("running", "candidate", "text");
    set("Edit private candidate") @datamodel, cli_auto_set();
    discard("Discard changes in private candidate"), discard_changes();
    update("Update private candidate from running if no conflicts"), cli_update();
    commit("Commit the changes if no conflicts"), cli_commit();

Scenario using these cli commands to handle a conflict ::

    > set interfaces interface intf_one description "Link to Gothenburg"
    > compare
        interface intf_one {
    -        description "Link to London";
    +        description "Link to Gothenburg";
    }

 some other session updates the same interface

    > commit
    Conflict occured: Cannot change node value, it is already changed: xpath0: /interfaces/interface[name="intf_one"]/description value0: Link to London value1: Link to Gothenburg
    > discard
    > update
    > set interfaces interface intf_one description "Link to Gothenburg"
    > compare
        interface intf_one {
    -        description "Link to Stockholm";
    +        description "Link to Gothenburg";
    }
    > commit
    > show running
    ietf-interfaces:interfaces {
        interface intf_one {
            description "Link to Gothenburg";
        }
    }


Module library support
======================

Clixon can store Yang module-state information according to `RFC 8525: YANG library <http://www.rfc-editor.org/rfc/rfc8525.txt>`_ in the
datastores. With module state, you know which Yang version the XML belongs to, which is useful when upgrading, see :ref:`upgrade <clixon_upgrade>`.

Options
-------

CLICON_XMLDB_MODSTATE
   Tag datastores with RFC 8525 YANG Module Library info

Operation
---------

To enable yang module-state in the datastores add the following entry in the Clixon configuration::

   <CLICON_YANG_LIBRARY>true</CLICON_YANG_LIBRARY> # (default true)
   <CLICON_XMLDB_MODSTATE>true</CLICON_XMLDB_MODSTATE>

If the datastore does not contain module-state, general-purpose upgrade is the only upgrade mechanism available.

A backend with ``CLICON_XMLDB_MODSTATE`` disabled will silently ignore module state.

Example of a (simplified) datastore with Yang module-state::

   <config>
     <a1 xmlns="urn:example:a">some text</a1>
     <!-- Here goes regular config -->
     <yang-library xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
       <content-id>42</content-id>
       <module-set>
         <name>default</name>
         <module>
           <name>A</name>
           <revision>2019-01-01</revision>
           <namespace>urn:example:a</namespace>
         </module>
       </module-set>
     </yang-library>
   </config>

Note that the module-state is not available to the user, the backend
datastore handler strips the module-state info. It is only shown in
the datastore itself.

System-only config
==================

`System-only` config is a mechanism to disable storing of sensitive
configuration data in the datastore.  Instead, a user implements
application callbacks to store the data in a `system state`, which
could be something like a system configuration file (e.g., a password
file), a system call, device config, etc.

System-only config is never stored in datastore files and is stored in memory only temporary while a candidate commit is taking place.

.. note::
    System-only config is never stored in datastore files.

This guide follows the test and main example in the clixon repository. The code in this description is somewhat simplified, see the following files for full details:  ``test/test_datastore_system_only.sh`` and the clixon main example ``example/main/example_backend.c``.

Options
-------
CLICON_XMLDB_SYSTEM_ONLY_CONFIG
   Enable system-only-config, set to true

Restrictions
------------
The following functionality is restricted with system-only config:

* Rollbacks
* Commit confirm

The reason saved configurations as rollbacks do not support system-only config is simply that system-only config is not stored in configuration files.  Rollback of system-only needs to be solved by other means.



Source-of-truth
---------------
System-only config acts as source of truth in the sense that a user can modify the system directly, outside clixon. For example, modifying a password via the OS, or calling a system-call directly.

This is different from regular configured data, where NETCONF clients
are the only configuration source.  Any direct modification of
configured data is ignored, and is overwritten at next commit.

Running
^^^^^^^
Example: Assume a system-only data ``A`` initially is set to: ``A=X``. A ``get-config running`` retrieves ``X``.

Then, set ``A=Y`` directly by the system (not via clixon), then ``get-config running`` retrieves ``Y``.

Likewise, if ``A=Z`` via NETCONF and committed, then ``get-config running`` retrieves ``Z``.

Candidate
^^^^^^^^^
A candidate which is not locked or modified behaves as running. That is, changed system-only config also appears in candidate.

However, as soon as candidate is modified or locked, the system-only config is not changed.

For example, assume a system-only config is initially set to ``A=X``.

Assume then a NETCONF client performs ``lock candidate`` (or ``edit-config``). Then, ``get-config candidate`` still retrieves ``X`` and there is no difference between running and candidate.

Then, ``A=Y`` is set directly by the system (not via clixon), then ``get-config candidate`` retrieves ``X`` but ``get-config running`` retrieves ``Y``.

A ``show compare`` of candidate and running shows::

  - A Y
  + A X

If now a NETCONF client performs ``edit-config A=Z``. Then, ``get-config candidate`` retrieves ``Z`` and ``get-config running`` retrieves ``Y``.

A ``show compare`` of candidate and running shows::

  - A Y
  + A Z

And a ``commit`` sets running: ``A=Z``.

Note that for RESTCONF or short edit/commit cycles, this is usually inconsequential.

Extensions
----------
You identify and mark state-only YANG elements with the ``system-only-config`` extension. This is done either by:

1. Directly marking the element
2. Using augment to mark a standard YANG from a local YANG

The logic of the second approach is a standard YANG not under your control.

The example uses the second approach with a `clixon-standard` YANG as follows::

   module clixon-standard{
      yang-version 1.1;
      namespace "urn:example:std";
      prefix std;
      grouping system-only-group {
         leaf system-only-data {   // <----
            type string;
         }
      }
      grouping store-grouping {
         container keys {
            list key {
               key "name";
               leaf name {
                  type string;
               }
               uses system-only-group;
            }
         }
      }
      container store {
         uses store-grouping;
      }
   }

The second local module augments the standard YANG by marking the ``system-only-data`` with the ``system-only-config`` extension::

   module clixon-local{
      yang-version 1.1;
      namespace "urn:example:local";
      prefix local;
      import clixon-lib {
         prefix cl;
      }
      import clixon-standard {
         prefix std;
      }
      augment "/std:store/std:keys/std:key/std:system-only-data" {
         cl:system-only-config;  // <----
      }
   }

Commit callback
---------------
The next step is to commit the system-only-config data following the regular commit callback mechanism described in the :ref:`backend section<clixon_backend>`.

The only difference is that the system-only config data is then deleted and not stored in running-db.

System-only callback
--------------------
The last step is to write a system-only callback to recreate system-only data on read. This is necessary in a NETCONF get-config request, for example.

The purpose is to merge the system-only data into the cached XML tree
in-memory after reading the datastore.

The way to add system-only config data is similar to how to add state data with the ``ca_statedata`` callback, as desctibed in the :ref:`backend section<clixon_backend>`.

In this example, the system-only config data is hardcoded, whereas a real example would extract this information from the system.

Note that the contructed XML must be a valid with respect to YANG from the top-level all the way down to the system-only data, which includes having valid list keys::

   int
   main_system_only_callback(clixon_handle h,
                             cvec         *nsc,
                             char         *xpath,
                             cxobj        *xconfig)
   {
      if (clixon_xml_parse_string("<store xmlns=\"urn:example:std\">
                                      <keys>
                                         <key>
                                            <name>a</name>
                                            <system-only-data>mydata</system-only-data>
                                         </key>
                                      </keys>
                                   </store>",
                                  YB_NONE, 0, &xconfig, 0) < 0)
         err;
      ...
   }
   static clixon_plugin_api api = {
      ...
      .ca_system_only=main_system_only_callback,

C API
=====
Some C functions to modify the datastore are:

* `xmldb_put` for modifying data
* `xmldb_get0` for a copy of the cache
* `xmldb_cache_get` for a copy of the cache
* `xmldb_copy` for copying datastores: both file and cache are copied
* `xmldb_volatile_set` to disable cache write-through temporarily
