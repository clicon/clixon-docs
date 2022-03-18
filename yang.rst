.. _clixon_yang:
.. sectnum::
   :start: 12
   :depth: 3

****
YANG
****

This chapter describes some aspects of he YANG implementation in Clixon. Regarding standard compliance, see :ref:`Standards<clixon_standards>`.


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

CLICON_MODULE_LIBRARY_RFC7895
  Enable RFC 7895 YANG Module library support as state data, instead of RFC8525. Default: ``false``

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
     yang_stmt *y = xml_spec(x);

     if (yang_extension_value(y, "mymode", "urn:example:lib", &value) < 0)
        err;
     if (value != NULL){
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

The RFC says::
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
