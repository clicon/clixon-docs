.. _clixon_yang:
.. sectnum::
   :start: 13
   :depth: 3

****
YANG
****

This chapter describes some aspects of the YANG implementation in Clixon. Regarding standard compliance, see :ref:`Standards<clixon_standards>`.

Leafrefs
========
Some notes on `YANG leafref <https://www.rfc-editor.org/rfc/rfc7950.html#section-9.9.3>`_ implementation in Clixon, especially as used in `openconfig modules <https://datatracker.ietf.org/doc/html/draft-openconfig-netmod-opstate-01>`_,  which rely heavily on leafrefs.

Typically, a YANG leafref declaration looks something like this::

  container c {  
    leaf x {
      type leafref {
        path "../config/name";   /* "deferring node" */
        require-instance <bool>; /* determines existing deferred node */
      }
    }
    container config {
      leaf name {
        type unit32;             /* "deferred node" */
      }
    }
  }

This YANG example is typical of Openconfig lists defined in the `openconfig modeling <https://datatracker.ietf.org/doc/html/draft-openconfig-netmod-opstate-01#section-8.1.2>`_, where a key leaf references a "config" node further down in the tree.

Other typical uses is where the path is an absolute path, such as eg ``path "/network-instances/network-instance/config/name";``
  
Types
-----
Consider the YANG example above, the type of ``x`` is the deferred node:s, in this example ``uint32``.
The validation/commit process, as well as the autocli type system and completion handles accordingly.

For example, if the deferred node is a more complex type such as identityref with options "a, b", the completion of "x" will show the options "a,b".

Require-instance
----------------
Assume the yang above have the following two XML trees::

  <c>
    <x>foo</x>
  </c>

and::

  <c>
    <x>foo</x>
    <config>
      <name>foo</name>
    </config>
  </c>
  
The validity of the trees is controlled by the `require-instance property <https://www.rfc-editor.org/rfc/rfc7950.html#section-9.9.3>`_ . According to this semantics:

 - If require-instance is false, both trees above are valid,
 - If require-instance is true(or not present), the upper tree is invalid and the lower is valid

In most models defined by openconfig and ietf, require-instance is typically false.

YANG Library
============

Clixon partially supports `YANG library RFC 8525 <http://www.rfc-editor.org/rfc/rfc8525.txt>`_ that provides information about YANG modules and datastores, 

The following configure options are associated to the YANG library

CLICON_YANG_LIBRARY
  Enable YANG library support as state data according to RFC8525. Default: ``true``

CLICON_MODULE_SET_ID
  Contains a server-specific identifier representing the current set of modules and submodules.

CLICON_XMLDB_MODSTATE
  Tag datastores with RFC 8525 YANG Module Library info. See :ref:`datastore <clixon_datastore>` for details on how to tag datastores with Module-set info.

The module-set of RFC8525 can be retrieved using NETCONF get or RESTCONF GET as operational data. The fields that are supported are the following:

 - Content-id of the whole module-set
 - Name of each module
 - Namespace
 - Revision
 - Feature
 - Submodules 

The following fields are not supported   

 - import-only-module
 - deviation
 - schema
 - datastore

Example
-------

An example of a NETCONF ``get`` reply with module-state data of the main example is the following::

  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="42">
    <data>
      <yang-library xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
        <module-set>
          <name>default</name>
  	  <module>
            <name>clixon-autocli</name>
  	    <revision>2022-02-11</revision>
  	    <namespace>http://clicon.org/autocli</namespace>
  	  </module>
  	  <module>
  	    <name>clixon-example</name>
  	    <revision>2020-12-01</revision>
  	    <namespace>urn:example:clixon</namespace>
  	  </module>
          ...
        </module-set>
      </yang-library>
    </data>
  </rpc-reply>
  
Extensions
==========
Clixon implements YANG extensions.  There are several uses, but one is
to "annotate" a YANG specification with application-specific data that can be used
in plugin code for some reason.

An extension with an argument is introduced in YANG as follows::

   module example-lib {
      namespace "urn:example:lib";
      extension mymode {
         argument annotation;
      }

Such an extension can then be used in YANG declarations in two ways, either
*inline* or *augmented*.

An inlined extension is useful in a YANG module that the designer has
control over and can add extension reference directly in the YANG
specification.

Assume for example that an interface declaration is extended with the extension declared above, as follows::

   module my-interface {
     import example-lib{
       prefix exl;
     }
     container "interfaces" {
       list "interface" {
         exl:mymode "my-interface";
         ...

If you instead use an external YANG, where you cannot edit the YANG
itself, you can use augmentation instead, as follows::

  module my-augments {
   import example-lib{
      prefix exl;
   }
   import ietf-interfaces{
      prefix if;
   }
   augment "/if:interfaces/if:interface"{
      exl:mymode "my-interface";
   }
   ...

When this is done, it is possible to access the extension value in
plugin code and use that value to perform application-specific
actions. For example, assume an XML interface object ``x`` retrieve
the annotation argument::

     char      *value = NULL;
     int        exist = 0;
     yang_stmt *y = xml_spec(x);

     if (yang_extension_value(y, "mymode", "urn:example:lib", &exist, &value) < 0)
        err;
     if (exist){
        // use extension value
        if (strcmp(value, "my-interface") == 0)
	   ...
	 
A more advanced usage is possible via an extension callback
(``ca_callback``) which is defined for backend, cli, netconf and
restconf plugins. This allows for advanced YANG transformations. Please
consult the main example to see how this could be done.

Unique
======

The YANG unique statement is described in Section 7.8.3 of `RFC 7950 <https://www.rfc-editor.org/rfc/rfc7950.html/>`_. However, the RFC is somewhat vague in the descriptions of its arguments.

Clixon therefore supports two simultaneous distinct cases: multiple direct children and single descendants

Multiple direct children
------------------------
This is examplified in the RFC, such as::

     list server {
       key "name";
       unique "ip port";
       leaf ip...
       leaf port...

where ``ip`` and ``port`` are direct children of ``server`` and the uniquess applies to their combination in all list instances.

Single descendants
------------------

The RFC says:
  schema node identifiers, which MUST be given in the descendant form

This does not exclude more elaborate schema nodes than direct children
but are not explicitly allowed.

Therefore, Clixon also supports a single advanced schema node id. Such a schema node id
may define a set of leafs. The uniqueness is then validated
against all instances, such as for example::

     list server {
       key "name";
       unique c/inner/value;
       container c {
          list inner {
	     leaf value...

However, only a *single* such argument is allowed. The reason is that
such a schema node may potentially refer to a set of instances (not
just one) and the semantics of a combination of multiple such ids is unclear.

If-feature and anydata
======================

The YANG if-feature statement is described in Section 7.20.2 of `RFC 7950 <https://www.rfc-editor.org/rfc/rfc7950.html/>`_.  The RFC states that:

   Definitions tagged with "if-feature" are ignored when the server does not support that feature.

This is implemented by doing the following to disabled YANG nodes:

(1) Configuration data nodes are replaced locally to a single ANYDATA data. This means that XML derived from disabled features are accepted but no validation is possible.
(2) Other YANG nodes, such as RPCs or state data are removed.

Example, assume the following YANG::

  container c{
     if-feature A;
     leaf b {
        type string;
     }
  }
  rpc r {
     	input {
	    leaf x {
	        if-feature A;
		type string;
	    }
	}
  }

If feature ``A`` is NOT enabled, the YANG is transformed to::

  anydata c{
  }
  rpc r {
     	input {
	}
  }
  
The following config option is related:

CLICON_YANG_UNKNOWN_ANYDATA
   Treat unknown XML/JSON nodes as anydata when loading from startup db.
  
Schema mounts
=============

Clixon implements Yang schema mounts as defined in: `RFC 8528: YANG Schema Mount <http://www.rfc-editor.org/rfc/rfc8528.txt>`_  with the following restrictions:

1. A YANG mount-point can only be defined in a `presence container`.
2. Only `inline` mount-points are supported
3. `config false` mount-points are not supported

Configuration
-------------
The following configure options are associated to mount-points:

CLICON_YANG_SCHEMA_MOUNT
  Enable YANG library support as state data according to RFC8525. Should be set to: ``true``

YANG
----
Mount-points are enabled by importing `ietf-yang-schema-mount` and then apply the `mount-point` extension at a presence container. Example::

   module mymod {
      namespace "urn:example:my";
      ...
      import ietf-yang-schema-mount {
         prefix yangmnt;
      }
      list mylist {
         key name;
         leaf name{
            type string;
         }
         container myroot{
            presence "Otherwise root is not visible";
            yangmnt:mount-point "mylabel"{
               description "Root for other yang models";
            }
         }
      }
   }

Once declared in the YANG schema, moint-points will appear dynamically
in the data as they are added. For example, if a NETCONF `<edit-config>`
adds the `myroot` container above, it will be recognized as a
mount-point and populated with another set of YANG modules than the
top-level.

Populating a moint-point with YANG schemas is made by an application-dependent callback as described in Section `Mount callback`_.

State
-----
The moint-points appear in the state-data and can be retrieved using NETCONF get. The data appears in two places. 

First, on the top-level `schema-mounts`::

   <schema-mounts xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-schema-mount">
      <mount-point>
         <module>mymod</module>
         <label>mylabel</label>
         <config>true</config>
         <inline/>
      </mount-point>
   </schema-mounts>
      
Second, at the mount-point level for all dynamically added moint-points::

   <mylist xmlns="urn:example:my">
      <name>x</name>
      <myroot>
         <yang-library xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
            <module-set>
               <name>mylabel</name>
               <module>
                  <name>clixon-mount1</name>
                  <namespace>urn:example:mount1</namespace>
                  <revision>2023-05-01</revision>
               </module>
            </module-set>
         </yang-library>
      </myroot>
   </mylist>

In this example, there is one dynamically created moint-point in the
list `x` where the single YANG module `clixon-mount1` is mounted.
   
Mount callback
--------------
Mount-points need to be populated with YANG schemas. This is done by defining the `ca_yang_mount` callback. The following example illustrates how this is done in a C plugin::

   static clixon_plugin_api api = {
       ...
       .ca_yang_mount=example_mount,

As input the callback takes the XML mount-point, and as output a yang-lib module-set tree. It also provides how to validate the YANG schemas and whether it is read-only or read-write::

   int
   main_yang_mount(clixon_handle   h,
                   cxobj          *xt,
                   int            *config,
                   validate_level *vl,
                   cxobj         **yanglib)

For example, the callback could return something like::

   <yang-library xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
      <module-set>
         <name>mount</name>
         <module>
            <name>clixon-mount1</name>
            <namespace>urn:example:mount1</namespace>
            <revision>2023-05-01</revision>
         </module>
      </module-set>
   </yang-library>

Clixon calls this callback when needed, such as when a new mount-point is created.

CLI
---
It is possible to extend the Autocli with mount-points. However, it is application-dependent. For the interested user, the `Clixon controller <https://clixon-controller-docs.readthedocs.io>`_ has an adapted autocli for mount-points.
