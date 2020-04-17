.. _clixon_xml:

==========
 XML tree
==========

Clixon represents its internal data in an in "in-memory" tree
representation. In the C API, this data structure is called ``cxobj`` and
is used to represent config and state data. Typically, a cxobj is
parsed from or printed to XML or JSON, but is really a generic
representation of a tree.

Such a tree goes thorough three steps:

1. Syntactic creation. This is done either via parsing or via manual API calls.
2. Bind to YANG. Assure that the XML tree complies to a YANG specification.
3. Semantic validation. Ensuring that the XML tree complies to the backend validation rules.

Steps 2 and 3 are optional.
  
Creating XML
============

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
- ``YB_NONE``    Dont bind

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
- ``0``  Partial or no yang assigment made (at least one failed) and xerr set
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

Merging two trees
-----------------
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

Inserting sub-tree
------------------

Inserting a subtree can be made in several ways. The most straightforward is using parsing and the ``YB_PARENT`` YANG binding::

       cxobj *xy;
       xy = xpath_first(xt, NULL, "%s", "y");
       if ((ret = clixon_xml_parse_string("<x><k1>z</k2></x>",
                          YB_PARENT, yspec, &xy, NULL)) < 0)
       if (ret == 0)
          err; /* yang error */

with the same result as in tree merge.

Note that ``xy`` in this example points at the ``y`` node and is where the new tree is pasted. Neither tree need to be a root tree.

Another way to insert a subtree is to use ``xml_insert``::

       if (xml_insert(xy, xi, INS_LAST, NULL, NULL) < 0)
          err;

where both ``xy`` and ``xi`` are YANG bound trees. It is possible to
specify where the new child is insrteed (last in the example), but
this only applies if ``ordered-by user`` is specified in
YANG. Otherwise, the system will order the insertion of the subtree automatically.
       
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
      

Direct children
---------------

The basic C API for searching direct children of a cxobj is the ``clixon_xml_find_index()`` API.

An example call is as follows:
::
   
    clixon_xvec *xv = NULL;
    cvec    *cvk = NULL;
    /* Populate cvk with key/values eg k1=a k2:b */
    if (clixon_xml_find_index(xp, yp, namespace, name, cvk, &xv) < 0)
       err;
    /* Loop over found children*/
    for (i = 0; i < clixon_xvec_len; i++) {
	x = clixon_xpath_i(xvec, i);
        ...
    }

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
     
Paths
-----

If deeper searches are needed, i.e., not just to direct children,
Clixon :ref:`clixon_paths` can be used to make a search request. There
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
  


