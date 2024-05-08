.. _clixon_datastore:
.. sectnum::
   :start: 8
   :depth: 3

*********
Datastore
*********


 .. image:: datastore1.jpg
   :width: 70%

Clixon configuration datastores follow the Netconf model (from `RFC 6241: NETCONF Configuration Protocol <http://rfc-editor.org/rfc/rfc6241.txt>`_):

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
   Split configure datastore into multiple sub files. Default is single file

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
By default, the datastore files use pretty-printed XML, with the top-symbol `config`. The following is an example of a valid datastore:
::

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

Module library support
======================
Clixon can store Yang module-state information according to `RFC 8525: YANG library <http://www.rfc-editor.org/rfc/rfc8525.txt>`_ in the
datastores. With module state, you know which Yang version the XML belongs to, which is useful when upgrading, see :ref:`upgrade <clixon_upgrade>`.


To enable yang module-state in the datastores add the following entry in the Clixon configuration:
::

   <CLICON_YANG_LIBRARY>true</CLICON_YANG_LIBRARY> # (default true)
   <CLICON_XMLDB_MODSTATE>true</CLICON_XMLDB_MODSTATE>

If the datastore does not contain module-state, general-purpose upgrade is the only upgrade mechanism available.

A backend with `CLICON_XMLDB_MODSTATE` disabled will silently ignore module state.

Example of a (simplified) datastore with Yang module-state:
::
   
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

C API
=====
Some C functions to modify the datastore are:

* `xmldb_put` for modifying data
* `xmldb_get0` for a copy of the cache
* `xmldb_cache_get` for a copy of the cache
* `xmldb_copy` for copying datastores: both file and cache are copied
* `xmldb_volatile_set` to disable cache write-through temporarily
