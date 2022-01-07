.. _clixon_yang:
.. sectnum::
   :start: 12
   :depth: 3

****
YANG
****

This chapter describes aspects of he YANG implementation in Clixon. Regarding standard compliance, see :ref:`Standards<clixon_standards>`.

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

 - if require-instance is false (or not present) both trees above are valid,
 - if require-instance is true, theupper tree is invalid and the lower is valid

In most models defined by openconfig and ietf, require-instance is typically false.

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
