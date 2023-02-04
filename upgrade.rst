.. _clixon_upgrade:
.. sectnum::
   :start: 18
   :depth: 3

*******
Upgrade
*******

When Clixon starts, the backend reads a startup configuration (see :ref:`startup modes <clixon_backend>`)
parses it, checks for upgrades, and validates the configuration. It can also add extra XML, outside of the normal startup. On errors, it can enter a failsafe mode.

Clixon has three upgrade methods:

* General-purpose datastore upgrade.
* Module-specific manual upgrade
* Automatic upgrade (experimental)

General-purpose
===============
A plugin registers a `ca_datastore_upgrade` function which gets called
once on startup. This upgrade should be seen as a generic wrapper
function to basic repair or upgrade of existing datastores. The
module-specific upgrade callbacks are more fine-grained.

The general-purpose upgrade callback is usable if module-state is not
available, or actions such as the following need to be done in the whole datastore:

 * Remove or rename nodes
 * Rename namespaces
 * Add nodes

A recommended method, as shown in the example, is to make a pattern
matching using XPath and then perform actions on the resulting nodes.

Example::

  static clixon_plugin_api api = {
     ...
     .ca_datastore_upgrade=example_upgrade
  };
  
  /*! General-purpose datastore upgrade callback called once on startup
   * @param[in] h    Clicon handle
   * @param[in] db   Name of datastore, eg "running", "startup" or "tmp"
   * @param[in] xt   XML tree. Upgrade this "in place"
   * @param[in] msd  Info on datastore module-state, if any
   */
  int
  example_upgrade(clicon_handle h,
                  char         *db,
		  cxobj        *xt,
		  modstate_diff_t *msd)
  {
      cxobj   **xvec = NULL;   /* vector of result nodes */
      size_t    xlen; 
      cvec     *nsc = NULL;    /* Canonical namespace */
      int       i;
      
      /* Skip other than startup datastore */
      if (strcmp(db, "startup") != 0) 
         return 0;
      /* Skip if there is proper module-state in datastore */
      if (msd->md_status) 
         return 0;
      /* Get canonical namespaces for using "normalized" prefixes */      
      if (xml_nsctx_yangspec(yspec, &nsc) < 0)
         err;
      /* Get all xml nodes matching path */
      if (xpath_vec(xt, nsc, "/a:x/a:y", &xvec, &xlen) < 0) 
         err;
      /* Iterate through nodes and remove them */
      for (i=0; i<xlen; i++){
         if (xml_purge(xvec[i]) < 0)
	    err;
      }
  }

The example above first checks whether it is the `startup` datastore
and that it does not contain module-state. It then matches all nodes
matching the XPath `/a:x/a:y` using canonical prefixes, which are then
deleted.
  
Module-specific upgrade
=======================
Module-specific upgrade is only available if module-state is enabled, see :ref:`Module library support <clixon_datastore>`.

If the module-state of the startup configuration does not match the
module-state of the backend daemon, a set of module-specific upgrade callbacks are
made. This allows a user to program upgrade funtions in the backend
plugins to automatically upgrade the XML to the current version.

A user registers upgrade callbacks per YANG module. A user can
register many callbacks, or choose wildcards.  When an upgrade occurs,
the callbacks will be called if they match the module and the modules
have changed.

A module has changed if one of the following is true:
- A module present in the startup is no longer present in the system (DEL)
- A module in the system is not present in the startup (ADD)
- A module present in both the startup and the system has a different revision date (CHANGE)

Registering a callback
----------------------
A user registers upgrade callbacks in the backend `clixon_plugin_init()` function. The signature of upgrade callback is as follows:
::
   
  upgrade_callback_register(h, cb, ns, arg);

where:

* `h` is the Clicon handle,
* `cb` is the name of the callback function,
* `ns` defines the namespace of a Yang module. NULL denotes all modules.
* `arg` is a user defined argument which can be passed to the callback.

One example of registering an upgrade of an interface module: 
::

   upgrade_callback_register(h, upgrade_interfaces, "urn:example:interfaces", NULL);

Upgrade callback
----------------
When Clixon loads a startup datastore with outdated modules, the matching
upgrade callbacks will be called.

The signature of an upgrade callback is as follows::

  int upgrade_interfaces(h, xt, ns, op, from, to, arg, cbret)

where:

* `xt` is the XML tree to be upgraded
* `ns` is the namespace of the YANG module.
* `op` is a flag indicating upgrading operation, one of: ``XML_FLAG_ADD``, ``XML_FLAG_DEL``, ``XML_FLAG_CHANGE``. Note that this applies to per-module: whether a `module` has been added, deleted or changed.
* `from` is the revision date in the startup file of the module. It is zero if the operation is ``ADD``
* `to` is the revision date of the YANG module in the system. It is zero if the operation is ``DEL``
  
If no action is made by the upgrade callback, and thus the XML is not upgraded, the next step is XML/Yang validation.

An out-dated XML may still pass validation and the system will go up in normal state.

However, if the validation fails, the backend will try to enter the
failsafe mode so that the user may perform manual upgrading of the
configuration.

Example upgrade
---------------
The `Clixon main example <https://github.com/clicon/clixon/blob/master/example/main/example_backend.c>`_ shows code for upgrading of an interface module. The example is inspired by the ietf-interfaces module that made a subset of the upgrades shown in the examples.

The code is split in two steps.
The `upgrade_2014_to_2016` callback does the following transforms:

  * Move ``/if:interfaces-state/if:interface/if:admin-status`` to ``/if:interfaces/if:interface/``
  * Move ``/if:interfaces-state/if:interface/if:statistics`` to ``if:interfaces/if:interface/``
  * Rename ``/interfaces/interface/description`` to ``/interfaces/interface/descr``

The `upgrade_2016_to_2018` callback does the following transforms:
  * Delete ``/if:interfaces-state``
  * Wrap ``/interfaces/interface/descr`` to ``/interfaces/interface/docs/descr``
  * Change type ``/interfaces/interface/statistics/in-octets`` to ``decimal64`` and divide all values with 1000

Extra XML
=========
If the Yang validation succeeds and the startup configuration has been committed to the running database, a user may add "extra" XML.

There are two ways to add extra XML to running database after
start. Note that this XML is "merged" into running, not "committed".

The extra-xml feature is not available if startup mode is `none`. It
will also not occur in failsafe mode.


Via file
--------
The first way is via a file. Assume you want to add this xml:
::

  <config>
    <x xmlns="urn:example:clixon">extra</x>
  </config>

You add this via the -c option:
::
   
   clixon_backend ... -c extra.xml

Reset callback
--------------
The second way is by programming the plugin_reset() in the backend
plugin. The following code illustrates how to do this (see also example_reset() in example_backend.c)::

   int
   example_reset(clicon_handle h,
                 const char   *db)
   {
      cxobj *xt = NULL;
      yang_stmt *yspec;

      yspec = clicon_dbspec_yang(h);      
      /* Parse extra XML */
      if (clixon_xml_parse_string("<x xmlns=\"urn:example:clixon\">extra</x>"
                                YB_MODULE, yspec, &xt, NULL) < 0)
         err;
      xml_name_set(xt, "config");
      /* Write to db */
      if (xmldb_put(h, (char*)db, OP_MERGE, xt, NULL, NULL) < 0)
         err;
   }
   static clixon_plugin_api api = {
       ...
       .ca_reset=example_reset,
       ...
   }       

The ``example_reset`` function is registered in the plugin init code and is then called with an empty temp database (db). The code writes the extra XML into db (``xmldb_put``).

After exit of the callback, the system merges the temporary db into
the running datastore in the same way as via file, ie not via a
commit.

Failsafe mode
=============
If the startup fails, the backend looks for a `failsafe` configuration
in ``<CLICON_XMLDB_DIR>/failsafe_db``. If such a config is not found, the
backend terminates. In this mode, running and startup mode are
unchanged.

If the failsafe is found, the running-db is copied to tmp-db and the failsafe config is loaded and
committed into the running db.

If the startup mode was `startup`, the `startup` database will
contain syntax errors or invalidated XML.

If the startup mode was `running`, the the `tmp` database will contain
syntax errors or invalidated XML.

Repair
======
If the system is in failsafe mode (or fails to start), a user can
repair a broken configuration and then restart the backend. This can
be done out-of-band by editing the startup db and then restarting
clixon.

In some circumstances, it is also possible to repair the startup
configuration on-line without restarting the backend. This section
shows how to repair a startup datastore on-line.

However, on-line repair *cannot* be made in the following circumstances:

* The broken configuration contains syntactic errors - the system cannot parse the XML.
* The startup mode is `running`. In this case, the broken config is in the `tmp` datastore that is not a recognized Netconf datastore, and has to be accessed out-of-band.
* Netconf must be used. Restconf cannot separately access the different datastores.

First, copy the (broken) startup config to candidate. This is necessary since you cannot make `edit-config` calls to the startup db:
::
   
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <copy-config>
      <source><startup/></source>
      <target><candidate/></target>
    </copy-config>
  </rpc>

You can now edit the XML in candidate. However, there are some restrictions on the edit commands. For example, you cannot access invalid XML (eg that does not have a corresponding module) via the edit-config operation.
For example, assume `x` is obsolete syntax, then this is *not* accepted:
::
   
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <edit-config>
      <target><candidate/></target>
      <config>
        <x xmlns="example" operation='delete'/>
      </config>
    </edit-config>
  </rpc>

Instead, assuming `y` is a valid syntax, the following operation is allowed since `x` is not explicitly accessed:
::
   
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <edit-config>
      <target><candidate/></target>
      <config operation='replace'>
        <y xmlns="example"/>
      </config>
    </edit-config>
  </rpc>

Finally, the candidate is validate and committed:
::
   
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <commit/>
  </rpc>

The example shown in this Section is also available as a regression `repair test script <https://github.com/clicon/clixon/tree/master/test/test_upgrade_repair.sh>`_.

Automatic upgrades
==================
There is an EXPERIMENTAL xml changelog feature based on
"draft-wang-netmod-module-revision-management-01" (Zitao Wang et al)
where changes to the Yang model are documented and loaded into
Clixon. The implementation is not complete.

When upgrading, the system parses the changelog and tries to upgrade
the datastore automatically. This feature is experimental and has
several limitations.

You enable the automatic upgrading by registering the changelog upgrade method in ``clixon_plugin_init()`` using wildcards::

   upgrade_callback_register(h, xml_changelog_upgrade, NULL, 0, 0, NULL);

The transformation is defined by a list of changelogs. Each changelog defined how a module (defined by a namespace) is transformed from an old revision to a new. Example from `auto upgrade test script <https://github.com/clicon/clixon/tree/master/test/test_upgrade_auto.sh>`_::  

  <changelogs xmlns="http://clicon.org/xml-changelog">
    <changelog>
      <namespace>urn:example:b</namespace>
      <revfrom>2017-12-01</revfrom>
      <revision>2017-12-20</revision>
      ...
    <changelog>
  </changelogs>

Each changelog consists of set of (ordered) steps::

    <step>
      <name>1</name>
      <op>insert</op>
      <where>/a:system</where>
      <new><y>created</y></new>
    </step>
    <step>
      <name>2</name>
      <op>delete</op>
      <where>/a:system/a:x</where>
    </step>

Each step has an (atomic) operation:

* rename - Rename an XML tag
* replace - Replace the content of an XML node
* insert - Insert a new XML node
* delete - Delete and existing node
* move - Move a node to a new place

A *step* has the following arguments:

* where - An XPath node-vector pointing at a set of target nodes. In most operations, the vector denotes the target node themselves, but for some operations (such as insert) the vector points to parent nodes.
* when - A boolean XPath determining if the step should be evaluated for that (target) node.

Extended arguments:

* tag - XPath string argument (rename)
* new - XML expression for a new or transformed node (replace, insert)
* dst - XPath node expression (move)

Step summary:

* rename(where:targets, when:bool, tag:string)
* replace(where:targets, when:bool, new:xml)
* insert(where:parents, when:bool, new:xml)
* delete(where:parents, when:bool)
* move(where:parents, when:bool, dst:node)
