.. _clixon_path:
.. sectnum::
   :start: 18
   :depth: 3

*****
Paths
*****

Clixon implementes the following path techniques to navigate in XML trees:

* *XML Path Language* defined in `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_ as part of XML and used in  `NETCONF <http://www.rfc-editor.org/rfc/rfc6241.txt>`_.
* *Instance-identifier*  defined in `RFC 7950: The YANG 1.1 Data Modeling Language <https://www.rfc-editor.org/rfc/rfc7950.txt>`_, a subset of XPath and used in `NACM <https://www.rfc-editor.org/rfc/rfc8341.txt>`_,
* *Api-path* defined and used in `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_

XPaths
======
Example of XPath in a NETCONF ``get-config`` RPC using the XPath capability:
::

   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <get-config>
         <source>
	    <candidate/>
	 </source>
	 <filter type="xpath" select="/interfaces/interface[name='eth0']/description" />
      </get-config>
   </rpc>

XPath is a powerful language for addressing parts of an XML document, including types and expressions. The following is a valid but complex XPath:
::

   /assembly[name="robot_4"]//shape/name[contains(text(),'bolt')]/surface/roughness

Clixon uses XPaths extensively due to their expressive power.  However, it is recommended to use instance-identifiers instead if you want optimized access.

Namespaces in XPaths
--------------------
XPath uses `XML names <https://www.w3.org/TR/REC-xml-names/>`_, requiring an *XML namespace context* using the `xmlns` attribute to bind namespaces and prefixes.  An XML namespace
context can specify both:

  * A default namespace for unprefixed names (`/x/`), defined by for example: `xmlns="urn:example:default"`.
  * An explicit namespace for prefixed names prefix (`/ex:x/`), defined by for example: `xmlns:ex="urn:example:example"`.

Further, XML prefixes are *not inherited*, each symbol must be prefixed with a prefix or default. That is, `/ex:x/y` is not the same as `/ex:x/ex:y`, unless `ex` is also default.

Example: Assume an XML namespace context:
::

   <a xmlns="urn:example:default" xmlns:ex="urn:example:example">

with an associated XPath:
::

   /x/ex:y/ex:z[ex:i='w']

then symbol `x` belongs to "urn:example:default" and symbols `y`, `z` and `i` belong to "urn:example:example".

XPath API
=========
This section gives a brief overview of the XPath C-API.

Datatypes
---------
The datatypes of the XPath api are:

* ``xpath_tree``: XPath parse-tree which follows the XPath 1.0 standard. Such a tree is created via parsing from an XPath in string representation and used internally.
* ``xp_ctx``: A context based on the XPath standard, it is used to keep track of context nodes, and also to return results for the general case.

Results frm an XPath lookup is either nodes, booleans, numbers or strings. The API is primarily focussed on returning nodes.

Note that the XPath API is independent of YANG.

Parse and print
---------------
You can parse an XPath string into an `xpath_tree` and then print the result as follows. Note that the parse-tree is quite large, even for small examples::

   xpath_tree *xpt = NULL;
   if (xpath_parse("/x/ex:y/ex:z[ex:i='w']", &xpt) < 0)
      err;
   xpath_tree_print(stderr, xpt);

Functions
---------

Single
^^^^^^
The most basic function is ``xpath_first`` which is used for the most common case of returning a single XML node, given an XPath.

Assume an XML tree ``x`` as follows::

      <y xmlns="urn:example:a">
        <x>
          <k1>a</k1>
        </x>
        <x>
          <k1>b</k1>
        </x>
      </y>

Then, get the first XPath result as follows in C-code::

   cvec *nsc = NULL;
   if (xpath_first(x, nsc, "/y/x") < 0)
      err;

The call returns the node representing::

   <x>
      <k1>a</k1>
   </x>

Multiple
^^^^^^^^
If you would rather get all results, and not just the first, use: ``xpath_vec`` instead as follows::

   *nsc = NULL;
   cxobj **vec = NULL;
   size_t  veclen;
   if (xpath_vec(x, nsc, "/y/x", &vec, &veclen) < 0)
      err;

which returns a vector of the two elements in ``vec``::

   <x>
      <k1>a</k1>
   </x>
   <x>
      <k1>b</k1>
   </x>

Generic
^^^^^^^
An XPath can return a more generic result rather than a vector of XML nodes. The other types are booleans, numbers and string. By using ``xpath_vec_ctx``, you get an ``xpath_ctx`` in return, where the result can be any type.

The ``xpath_vec_ctx`` is the most generic API function call and is used by the others internally.

Matching XPaths
---------------
The API uses a namespace context, ``nsc`` which associates prefixes with namespaces.  This follows the standard `XML namespaces <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_

XML defines a `qualified` name as::

  Prefix ':' LocalPart

``Prefix`` is bound to a namespace using the ``xmlns`` attribute, while in XPaths, ``Prefix`` is bound to a namespace using the ``nsc`` list.

In general, a call to an XPath API function is on the form::

  xpath_vec_ctx(XML, NSC, XPATH, localonly)

The semantics of the XPath match follow three different variants:

1. `local` . ``localonly`` is set: Matching is made by name (LocalPart) only. All prefixes are ignored
2. `prefix` : ``NSC=NULL``. Prefix match is made lexically, ignoring namespace binding.
3. `namespace`: prefix to namespace is looked up and the resulting namespace must match.

In other words, if you choose  ``localonly`` in the API, XPath matching will compare names only, and if you omit the XPath namespace binding in ``nsc``, the XPath matching will only match prefixes.

Examples
^^^^^^^^
The three different variants are illustrated by examples.

Assume an XML as follows::

      <y xmlns="urn:example:m" xmlns:n="urn:example:n">
        <x>
          <n:k1>a</n:k1>
        </x>
        <x>
          <n:k1>b</n:k1>
        </x>
      </y>

Local match
^^^^^^^^^^^
If ``localonly`` is set, prefixes are ignored (NSC is ignored) and all following XPaths match::

  /y/x/k1
  /a:y/b:x/c:k1

Only names need to match.

Prefix match
^^^^^^^^^^^^
If ``NSC=NULL``, prefixes must match lexically. The following XPaths match::

  /y/x/k1
  /y/x/n:k1

The following does not::

  /m:y/m:x/k1

The prefixes in the XPath and XML must be string-equal, regardless of which namespace they are associated with.

Namespace match
^^^^^^^^^^^^^^^
If ``NSC`` contains a namespace binding, prefixes are looked up and the namespaces must match.

Assume a namespace binding ``NSC`` as follows::

  NULL : urn:example:m
  n : urn:example:n

Then the following XPath matches::

  /y/x/n:k1

Likewise, if ``NSC`` is::

  x : urn:example:m
  y : urn:example:n

then the following XPath matches:

  /x:y/x:x/y:k1

The prefixes in the XPath are evaluated to namespaces that in turn must match.

XML and XPath mapping
---------------------
You can map between XPath:s and XML via the two functions ``xml2xpath`` and ``xpath2xml``.

For example, consider the XML of::

      <y xmlns="urn:example:a">
        <x>
          <k1>a</k1>
        </x>
      </y>

The corresponding XPath is ``y/x[k1='a']``.

The two functions map between the two representations.

Instance-identifier
===================
Instance-id:s are defined in YANG for some minor usage but appears in
for example NACM and provides a useful subset of XPath. The subset is as follows (see Section 9.13 in `YANG 1.1 <https://www.rfc-editor.org/rfc/rfc7950.txt>`_):

* Child paths using slashes: ``/ex:system/ex:services``
* List entries for one or several keys: ``/ex:system[ex:ip='192.0.2.1'][ex:port='80']``
* Leaf-list entries for one key: ``/ex:system/ex:cipher[.='blowfish-cbc']``
* Position in lists: ``/ex:stats/ex:port[3]``

Example of instance-id in NACM:
::

     <path xmlns:acme="http://example.com/ns/itf">
           /acme:interfaces/acme:interface[acme:name='dummy']
     </path>

Namespaces in instance-identifiers are the same as in XPaths.

Api-path
========
RESTCONF defines api-paths as a YANG-based path language. Keys are implicit which make path expressions more concise, but they are also less powerful

Example of Api-path in a restconf GET request:
::

   curl -s -X GET http://localhost/restconf/data/ietf-interfaces:interfaces/interface=eth0/description

Clixon uses Api-paths internally in some cases when accessing xml
keys, but more commonly translates Api-paths to XPaths.

Api-path is in comparison to XPaths limited to pure path expressions such as, for example::

   a/b=3,4/c

which corresponds to the XPath: `a[i=3][j=4]/c`. Note that you cannot express any other index variables than the YANG list keys.

Namespaces in Api-paths
-----------------------
In contrast to XPath, Api-path namespaces are defined implicitly by a
YANG context using *module-names* as prefixes.  The namespace is
defined in the Yang module by the `namespace` keyword. Api-paths must
have a Yang definition whereas XPaths can be completely defined
in XML.

A prefix/module-name is *inherited*, such that a child inherits the prefix
of a parent, and there are no defaults. For example, `/moda:x/y` is the same as `/moda:a/moda:y`.

Further, an Api-path uses a shorthand for defining list indexes. For
example, `/modx:z=w` denotes the element in a list of `z`:s whose key
is the value `w`. This assumes that `z` is a Yang list (or leaf-list)
and the index value is known.

Example: Assume two YANG modules `moda` and `modx` with namespaces "urn:example:default" and "urn:example:example" respectively, with the following Api-path (equivalent to the XPath example above):
::

   /moda:x/modx:y/z=w

where, as above, `x` belongs to "urn:example:default" and `y`, and `z` belong to "urn:example:example".
