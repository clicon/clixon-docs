.. _clixon_xml:
.. sectnum::
   :start: 16
   :depth: 3

*************
XML and paths
*************

This section describes XML trees and how to navigate in trees using paths.

Clixon represents its internal data in an in "in-memory" tree
representation. In the C API, this data structure is called ``cxobj``
(Clixon XML object) and is used to represent config and state
data. Typically, a cxobj is parsed from or printed to XML or JSON, but
is really a generic representation of a tree.

Paths
=====
Clixon uses paths to navigate in XML trees.  Clixon uses the following three methods:

* *XML Path Language* defined in `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_ as part of XML and used in  `NETCONF <http://www.rfc-editor.org/rfc/rfc6241.txt>`_.
* *Instance-identifier*  defined in `RFC 7950: The YANG 1.1 Data Modeling Language <https://www.rfc-editor.org/rfc/rfc7950.txt>`_, a subset of XPath and used in `NACM <https://www.rfc-editor.org/rfc/rfc8341.txt>`_,
* *Api-path* defined and used in `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_

XPath
-----
Example of XPath in a NETCONF `get-config` RPC using the XPath capability:
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

   /x/ex:y/ex:z[ex:i='w']`,

then symbol `x` belongs to "urn:example:default" and symbols `y`, `z` and `i` belong to "urn:example:example".

Instance-identifier
-------------------
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
--------
RESTCONF defines api-paths as a YANG-based path language. Keys are implicit which make path expressions more concise, but they are also less powerful

Example of Api-path in a restconf GET request:
::

   curl -s -X GET http://localhost/restconf/data/ietf-interfaces:interfaces/interface=eth0/description

Clixon uses Api-paths internally in some cases when accessing xml
keys, but more commonly translates Api-paths to XPaths.

Api-path is in comparison to XPaths limited to pure path expressions such as, for example:
::
   
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


XML trees
=========
XML objects are typed as follows `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_:

* Element: non-terminal node with child nodes
* Attribute: name value pair
* Body: text between tags

Elements and attributes have names. An element has a set of children
while body and attribute have values.

For example, the following XML tree::

   <y xmlns="urn:example:a">
      <x>
         <k>a</k>
      </x>
   </y>

can have an internal tree representation as follows (``e`` represents XML element, ``b`` body and ``a`` attribute::

   y:e ---------> xmlns:a (value:"urn:example:a")
        \
          +-----> x:e ---------> k:e ---------> :b (value:"a") 

Yang binding
------------
Typically, XML elements are associated with a YANG data node
specification(``yang_stmt``). A YANG data node is either a
container, leaf, leaf-list, list, or anydata.

A YANG bound XML node have some constraints with respect to children as follows:

* All elements may have attributes
* A leaf or leaf-list has at most one body child.
* A leaf or leaf-list have no elements children
* A container or list have no body children
* A container or list may have one or many element children
* An anydata node may have both elements and one body child(?>

The XML example given earlier could have the following YANG specification::

  module mod_a{
    prefix a;
    namespace "urn:example:a";
    container y {
      list x{
        key k;
        leaf k{
          type string;
        }
      }
    }
  }

Annotating the tree representation with YANG specification, could yield the following YANG bound tree::

   container y    
   y:e ---------> xmlns:a (value:"urn:example:a")
      \
       \          list x         leaf k
         +------> x:e ---------> k:e ---------> :b (value:"a")


Sorted tree
-----------
Once an XML tree is bound to YANG, it can be sorted. 

The tree is sorted using a "multi-layered" approach:

1. XML type: attributes come before elements and bodies.
2. Yang nodename: XML nodes with same nodename are adjacent and follow the order they are given in the Yang specification. If XML nodes belong to different modules, they follow the order they were loaded into the system.
3. Sorting lists and leaf-lists. There are two variants:

  a) Ordered-by-system: This is the default. Elements are sorted according to key value. Key value comparison is typed: if the key type is string, strcmp is used, if the key value is an integer, integer `<>=` is used, etc.
  b) Ordered-by-user: the list items follow the ordered they were entered, regardless of list keys. Ordered-by-user is not recommended in clixon since the optimized searching algorithms uses sorted lists. 

Extending the example above slightly with a new list ``x2`` as follows::

      list x2{
        key k2;
        leaf k2{
          type int32;
        }
      }

could give the following sorted XML tree::

   <y xmlns="urn:example:a">
      <x>
         <k>a</k>
      </x>
       <x>
         <k>b</k>
      </x>
      <x2>
         <k2>9</k2>
      </x2>
      <x2>
         <k2>100</k2>
      </x2>
   </y>
  
Note that among ``y``:s children, the attribute is the first (layer
1), then follows the group of ``x`` elements and the group of ``x2``
elements as they are given in the YANG specification (layer
2). Finally, the lists are internally sorted according to key values.

.. note::
        Sorting is necessary to achieve fast searching as described in Section `Searching in XML`_.

Creating XML
============
The creation of and XML tree goes thorough three steps:

1. Syntactic creation. This is done either via parsing or via manual API calls.
2. Bind to YANG. Assure that the XML tree complies to a YANG specification.
3. Semantic validation. Ensuring that the XML tree complies to the backend validation rules.

Steps 2 and 3 are optional.

Creating XML from a string
--------------------------
A simple way to create an cxobj is to parse it from a string:
::

     cxobj *xt = NULL;
     if ((ret = clixon_xml_parse_string("<y xmlns='urn:example:a'><x><k1>a</k2></x></y>",
                          YB_MODULE, yspec, &xt, NULL)) < 0)
        err;
     if (ret == 0)
        err; /* yang error */

where

* ``YB_MODULE`` is the default Yang binding mode, see `Binding YANG to XML`_.
* ``xt`` is a top-level cxobj containing the XML tree. 
* ``yspec`` is the top-level yang spec obtained by e.g., ``clicon_dbspec_yang(h)``

If printed with for example: ``xml_print(stdout, xt)`` the tree looks as follows::
   
   <top>
      <y xmlns="urn:example:a">
        <x>
          <k1>a</k1>
        </x>
      </y>
   </top>

Note that a top-level node (``top``) is always created to encapsulate
all trees parsed and that the default namespace in this example
is "urn:example:a".

The XML parse API has several other functions, including:

- ``clixon_xml_parse_file()``  Parse a file containing XML
- ``clixon_xml_parse_va()``    Parse a string using variable argument strings

Creating JSON from a string
----------------------------
You can create an XML tree from JSON as well::

     cxobj *xt = NUL;L
     cxobj *xerr = NULL;

     if ((ret = clixon_json_parse_string("{\"mod_a:y\":{\"x\":{\"k1\":\"a\"}}}",
                     YB_MODULE, yspec, &xt, NULL)) < 0)
        err;

yielding the same xt tree as in `Creating XML from a string`_.

In JSON, namespace prefixes use YANG module names, making the JSON
format dependent on a correct YANG binding. 

The JSON parse API also includes:

- ``clixon_json_parse_file()``  Parse a file containing JSON

  
Creating XML programmatically
-----------------------------
You may also manually create a tree by ``xml_new()``, ``xml_new_body()``,
``xml_addsub()``, ``xml_merge()`` and other functions. Instead of parsing a string, a
tree is built manually. This may be more efficient but more work to
program.

The following example creates the same XML tree as in the above examples using API calls::

   cxobj *xt, *xy, *x, *xa;
   if ((xt = xml_new("top", NULL, CX_ELMNT)) == NULL)
      goto done;
   if ((xy = xml_new("y", xt, CX_ELMNT)) == NULL)
      goto done;
   if ((xa = xml_new("xmlns", y, CX_ATTR)) == NULL)
      goto done;
   if (xml_value_set(xa, "urn:example:a") < 0)
      goto done;
   if ((x = xml_new("xy", xt, CX_ELMNT)) == NULL)
      goto done;
   if (xml_new_body("k1", x, "a") == NULL)
      goto done;

.. note::
        If you create the XML tree manually, you may have to explicitly call a yang bind function.

Binding YANG to XML
-------------------
A further step is to ensure that the XML tree complies to a YANG
specification. This is an optional step since you can handle XML
without YANG, but often necessary in Clixon, since some functions
require YANG bindings to be performed correctly. This includes sort,
validate, merge and insert functions, for example.

Yang binding may be done already in the XML parsing phase, and is
mandatory for JSON parsing. If XML is manually created, you need to
explicitly call the Yang binding functions.

For the XML in the example above, the YANG module could look something like:
::

  module mod_a{
    prefix a;
    namespace "urn:example:a";
    container y {
      list x{
        key k1;
        leaf k1{
          type string;
        }
      }
    }
  }
  
Binding is made with the ``xml_bind_yang()`` API. The bind API can be done in some different ways as follows:

- ``YB_MODULE``  Search for matching yang binding among top-level symbols of Yang modules. This is default.
- ``YB_PARENT``  Assume yang binding of existing parent and match its children by name
- ``YB_NONE``    Do not bind

In the example above, the binding is ``YB_MODULE`` since the top-level symbol
``x`` is a top-level symbol of a module.

But assume instead that the string ``<k1 xmlns="urn:example:a">a</k1>``
is parsed or created manually. You can determine which module it belongs to from the
namespace, but there may be many ``k1`` symbols in that module, you do
not know if the "leaf" one in the Yang spec above is the correct one.

The following is an example of how to bind yang to an XML tree ``xt``:
::

   cxobj *xt;
   cxobj *xerr = NULL;
   /* create xt as example above */
   if ((ret = xml_bind_yang(xt, YB_MODULE, yspec, NULL)) < 0)
      goto done;   /* fatal error */
   if (ret == 0)
      goto noyang; /* yang binding error */
     
The return values from the bind API are same as parsing, as follows:

- ``1``  OK yang assignment made
- ``0``  Partial or no yang assignment made (at least one failed) and xerr set
- ``-1``  Error

As an example of `YB_PARENT` Yang binding, the ``k1`` subtree is inserted under an existing XML tree which has already been bound to YANG. Such as an XML tree with the ``x`` symbol.

   
Config data
-----------
To create a copy of configuration data, a user retrieve a copy from the datastore to get a cxobj handle. This tree is fully bound, sorted and defaults set.
Read-only operations may then be done on the in-memory tree.

The following example code gets a copy of the whole `running` datastore to cxobj ``xt``:
::

     cxobj *xt = NULL;
     if (xmldb_get(h, "running", NULL, NULL, &xt) < 0)
        err;

.. note::
        In the case of config data, in-memory trees are read-only *caches* of
        the datastore and can normally not be written back to the datastore.
        Changes to the config datastore should be made via the backend netconf API, eg using
        ``edit-config``.


Modifying XML
=============
Once an XML tree has been created and bound to YANG, it can be modified in several ways.

Merging
-------
If you have two trees, you can merge them with ``xml_merge`` as follows::

	if ((ret = xml_merge(xt, x2, yspec, &reason)) < 0) 
	  err;
	if (ret == 0)
	  err; /* yang failure */

where both ``xt`` and ``x2`` are root XML trees (directly under a module) and fully YANG bound. For example, if ``x2`` is::

   <top>
      <y xmlns="urn:example:a">
        <x>
          <k1>z</k1>
        </x>
      </y>
   </top>

the result tree ``xt`` after merge is::

   <top>
      <y xmlns="urn:example:a">
        <x>
          <k1>a</k1>
        </x>
        <x>
          <k1>z</k1>
        </x>
      </y>
   </top>

Note that the result tree is sorted and YANG bound as well.
   
Inserting
---------
Inserting a subtree can be made in several ways. The most straightforward is using parsing and the ``YB_PARENT`` YANG binding::

       cxobj *xy;
       xy = xpath_first(xt, NULL, "%s", "y");
       if ((ret = clixon_xml_parse_string("<x><k1>z</k2></x>", YB_PARENT, yspec, &xy, NULL)) < 0)
       if (ret == 0)
          err; /* yang error */

with the same result as in tree merge.

Note that ``xy`` in this example points at the ``y`` node and is where the new tree is pasted. Neither tree need to be a root tree.

Another way to insert a subtree is to use ``xml_insert``::

       if (xml_insert(xy, xi, INS_LAST, NULL, NULL) < 0)
          err;

where both ``xy`` and ``xi`` are YANG bound trees. It is possible to
specify where the new child is inserted (last in the example), but
this only applies if ``ordered-by user`` is specified in
YANG. Otherwise, the system will order the insertion of the subtree automatically.
       
Removing
--------
A subtree can be permanently removed, or just pruned in order to insert it somewhere else.
and graft subtrees.

Permanently deleting a (sub)tree ``x`` and remove or from its parent is done as follows::

  xml_purge(x);

Removing a subtree ``x`` from its parent is done as follows::

  xml_rm(x);

or alternatively remove child number ``i`` from parent ``xp``::

    xml_child_rm(xp, i);

In both these cases, the child ``x`` can be used as a stand-alone
tree, or being inserted under another parent. 

Copying
-------
An XML tree ``x0`` can be copied as follows::

   cxobj *x1;
   x1 = xml_new("new", NULL, xml_type(x0));
   if (xml_copy(x0, x1) < 0)
      err;

Alternatively, a tree can be duplicated as follows::

   x1 = xml_dup(x0);

In these cases, the new object ``x1`` can be use as a separate tree for insertion, for example.
  
Searching in XML
=================
Clixon search indexes are either *implicitly* created from the YANG
specification, or *explicitly* created using the API.

From YANG it is only ``list`` and ``leaf-list`` that are candidates for
optimized lookup, direct ``leaf`` and ``container`` lookup is fast either way.

*Binary* search is used by search indexes and works by ordering list
items alphabetically (or numerically), and then dividing the search interval in
two equal parts depending on if the requested item is larger than, or
less than, the middle of the interval.

Binary search complexity is *O(log N)*, whereas linear search is is *O(n)*. 
For example, a search in a vector of one million children will take up to
`20` lookups, whereas linear search takes up to `1.000.000` lookups.

Therefore, if you have a large number of children and you need to make
searches, it is important that you use indexes, either implicit, or explicit.

Auto-generated indexes
----------------------
Auto-generated (or implicit) YANG-based search indexes are based on ``list`` and ``leaf-lists``. For
any list with keys ``k1,...kn``, a set of indexes are created and an optimized search
can be made using the keys in the order they are defined. 

For example, assume the following YANG (this YANG is reused in later examples):
::

  module mod_a{
    prefix a;
    namespace "urn:example:a";
    import clixon-config {
      prefix "cc";
    }
    list x{
      key "k1 k2";
      leaf k1{
        type string;
      }
      leaf k2{
        type string;
      }
      leaf-list y{
        type string;
      }
      leaf z{
        type string;
      }
      leaf w{
        type string;
	cc:search_index;
      }
      ...

Assume also an example XML tree as follows:
::

   <top xmlns="urn:example:a">
     <x>
       <k1>a</k1>
       <k2>a</k2>
       <y>cc</y>
       <y>dd</y>
       <z>ee</z>
       <w>ee</w>
     </x>
     <x>
       <k1>a</k1>
       <k2>b</k2>
       <y>cc</y>
       <y>dd</y>
       <z>ff</z>
       <w>ff</w>
     </x>
     <x>
       <k1>b</k1>
       ...
   </top>
      
Then there will be two implicit search indexes created for all XML nodes ``x`` so that
they can be accessed with *O(log N)*  with e.g.:

* XPath or Instance-id: ``x[k1="a"][k2="b"]``.
* Api-path: ``x=a,b``.

If other search variables are used, such as: ``x[z="ff"]`` the time complexity will be *O(n)* since there is no explicit index for ``z``.  The same applies to using key variables in another order than they appear in the YANG specification, eg: ``x[k2="b"][k1="a"]``.

A search index is also generated for leaf-lists, using ``x`` as the base node, the following searches are optimized:

* XPath or Instance-id: ``y[.="bb"]``.
* Api-path: ``y=bb``.
  
In the following cases, implicit indexes are *not* created:

* No YANG definition of the XML children exists. There are several use-cases. For example that YANG is not used or the tree is part of YANG `ANYXML`. 
* The list represents `state` data
* The list is `ordered-by user` instead of the default YANG `ordered-by system`.

Explicit indexes
----------------
In those cases where implicit YANG indexes cannot be used, indexes can
be explicitly declared for fast access. Clixon uses a YANG extension to declare such indexes: `search_index` as shown in the example above for leaf ``w``::

      leaf w{
        type string;
	cc:search_index;
      }

In this example, ``w`` can be used as a search index with *O(log N)* in the search API.

The corresponding direct API call is: ``yang_list_index_add()``

Direct children
---------------
The basic C API for searching direct children of a cxobj is the ``clixon_xml_find_index()`` API.

An example call is as follows:
::
   
    clixon_xvec *xv = NULL;
    cvec    *cvk = NULL;

    if ((xv = clixon_xvec_new()) == NULL)
       goto done;
    /* Populate cvk with key/values eg k1=a k2:b */
    if (clixon_xml_find_index(xp, yp, namespace, name, cvk, xv) < 0)
       err;
    /* Loop over found children*/
    for (i = 0; i < clixon_xvec_len(xv); i++) {
	x = clixon_xpath_i(xvec, i);
        ...
    }
    if (xv)
       clixon_xvec_free(xv);

where

+----------+-------------------------------------------+
| ``xp``   | is an XML parent                          |
+----------+-------------------------------------------+
| ``yp``   | is the YANG specification of xp           |
+----------+-------------------------------------------+
| ``name`` | is the name of the wanted children        |
+----------+-------------------------------------------+
| ``cvk``  | is a vector of index name and value pairs |
+----------+-------------------------------------------+
| ``xvec`` | is a result vector of XML nodes.          |
+----------+-------------------------------------------+

For example, using the previous XML tree and if ``name=x`` and  ``cvk``
contains the single pair: ``k1=a``, then ``xvec`` will contain both ``x``
entries after calling the function:
::

     0: <x><k1>a</k1><k2>a</k2><y>cc</y><y>dd</y><z>foo</a></x>
     1: <x><k1>a</k1><k2>b</k2><y>cc</y><y>dd</y><z>bar</a></x>

and the search was done using *O(logN)*.
     
Using paths in XML
------------------
If deeper searches are needed, i.e., not just to direct children,
Clixon `paths`_ can be used to make a search request. There
are three path variants, each with its own pros and cons:

* XPath is most expressive, but only supports *O(logN)* search for
  YANG `list` entries (not leaf-lists), and adds overhead in terms of
  memory and cycles.
* Api-path is least expressive since it can only express YANG `list`
  and `leaf-list` key search.
* Instance-identifier can express all optimized searches as well as
  non-key searches. This is the recommended option.

Assume the same YANG as in the previous example, a path to find ``y`` entries with a specific value could be:

* XPath or instance-id: ``/a:x[a:k1="a"][a:k2="b"]/a:y[.="bb"]`` 
* Api-path: ``/mod_a:x=a,b/y=bb``

which results in the following result:
::

     0: <y>bb</y>
  
An example call using instance-id:s is as follows:
::

   cxobj **vec = NULL;
   size_t  len = 0;
   if (clixon_xml_find_instance_id(xt, yt, &vec, &len,
          "/a:x[a:k1=\"a\"][k2=\"b\"]/a:y[.=\"bb\"") < 0) 
      goto err;
   for (i=0; i<len; i++){
      x = vec[i];
         ...
   }

The example shows the usage of auto-generated key indexes which makes this
work in *O(logN)*, with the same exception rules as for direct children state in `Auto-generated indexes`_.

An example call using api-path:s instead is as follows:
::

   cxobj **vec = NULL;
   size_t  len = 0;
   if (clixon_xml_find_api_path(xt, yt, &vec, &len,
          "/mod_a:x=a,b/y=bb") < 0) 
      goto err;
   for (i=0; i<len; i++){
      x = vec[i];
         ...
   }

The corresponding API for XPath is ``xpath_vec()``.

Multiple keys
-------------
Optimized *O(logN)* lookup works with multiple key YANG `lists` but not
for explicit indexes. Further, less significant keys can be omitted
which may result multiple result nodes.

For example, the following lookups can be made using *O(logN)* using implicit indexes:
::

   x[k1="a"][k2="b"]/y[.="cc"]
   x[k1="a"]/y[.="cc"]
   x[k1="a"][k2="b"]

The following lookups are made with *O(N)*:
::

   x[k2="b"][k1="a"]
   x[k1="a"][z="foo"]


Internal representation
=======================
A cxobj has several components, which are all accessible via the API. For example:

+------------+-----------------------------------------------------------+
| name       | Name of node                                              |
+------------+-----------------------------------------------------------+
| *prefix*   | Optional prefix denoting a localname according to XML     |
|            | namespaces                                                |
+------------+-----------------------------------------------------------+
| *type*     |  A node is either an element, attribute or body text      |
+------------+-----------------------------------------------------------+
| *value*    | Attributes and bodies may have values.                    |
+------------+-----------------------------------------------------------+
| *children* | Elements may have a set of XML children                   |
+------------+-----------------------------------------------------------+
| *spec*     | A pointer to a YANG specification of this XML node        |
+------------+-----------------------------------------------------------+

The most basic way to traverse an cxobj tree is to linearly iterate
over all children from a parent element node.
::

   cxobj *x = NULL;
   while ((x = xml_child_each(xt, x, CX_ELMNT)) != NULL) {
     ...
   }

where ``CX_ELMNT`` selects element children (no attributes or body text).

However, it is recommended to use the `Searching in XML`_ for more efficient
searching.
		   
Character encoding
==================
Clixon implements encoding of character data as defined in `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_, Section 2.4.

It can be illustrated by some examples. Assume a data-field "value" of
type "string" including some special characters (wrt XML):
"<description/>". This string can be input using NETCONF or RESTCONF using XML and JSON as follows:

1. Restconf POST using JSON, eg: ``{"value": "<description/>"}``
2. Restconf POST using XML regular x3 encoding, eg: ``<value>&lt;description/&gt;</value>``
3. Restconf POST using XML and CDATA: ``<value><![CDATA[<description/>]]></value>``
4. Netconf edit-config using XML regular encoding: ``<value>&lt;description/&gt;</value>``
5. Netconf edit-config using XML CDATA: ``<value><![CDATA[<description/>]]></value>``

The input is received by the backend where the value is stored in the backend as follows:

1. ``<value>&lt;description/&gt;</value>``
2. ``<value>&lt;description/&gt;</value>``
3. ``<value><![CDATA[<description/>]]></value>``
4. ``<value>&lt;description/&gt;</value>``
5. ``<value><![CDATA[<description/>]]></value>``

Note that in most cases, the data is just propagated from input to datastore, except in the JSON to XML translation(case 1).

For output assuming the value above, there are two data formats to consider in the datastore above: 1) with regular x3 encoding and 2) using CDATA.

There are the following cases:

1. Restconf GET datastore entry 1 using JSON: ``"{"value": "<description/>"}``
2. Restconf GET datastore entry 2 using JSON: ``"{"value": "<![CDATA[<description/>]]>"}`` (recently changed)
3. Restconf GET datastore entry 1 using XML: ``<value>&lt;description/&gt;</value>``
4. Restconf GET datastore entry 2 using XML: ``<value><![CDATA[<description/>]]></value>``
5. Netconf get-config datastore entry 1: ``<value>&lt;description/&gt;</value>``
6. Netconf get-config datastore entry 2: ``<value><![CDATA[<description/>]]></value>``

Internally, data is saved in cleartext which is encoded when
translated to XML.  CDATA encoding is an exception where it is stored internally as well.

