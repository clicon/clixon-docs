.. _clixon_xml:

==========
 XML tree
==========

Clixon represents its internal data in an in "in-memory" tree
representation. In the C API, this data structure is called `cxobj` and
is used to represent config and state data. Typically, a cxobj is
parsed from or printed to XML or JSON, but is really a generic
representation of a tree.

Creating XML
============

Creating XML from a string
--------------------------

A simple way to create an cxobj is to parse it from a string:
::

     cxobj *xt = NULL;
     if (xml_parse_va(&xt, yt, "<x xmlns='urn:example:a'><k1>a</k2></x>") < 0)
        err;

where

* `xt` is a top-level cxobj containing the XML tree. 
* `yt` is the yang spec corrresponding to `xt` (or NULL)
* The third parameter is a "printf" like format string followed by a variable argument list

If printed with for example: `xml_print(stdout,xt)` the tree looks as follows:
::
   
   <top>
      <x xmlns="urn:example:a">
         <k1>a</k2>
      </x>
   </top>

Note that a top-level node (`top`) is always created to encapsulate
all trees parsed and that the default namespace in this example
is "urn:example:a".


XML and YANG
------------

If a yang specification (`yt`) is supplied to the parse function, the
XML is bound to YANG. YANG binding is necessary for many Clixon
functions including validation, optimized search, etc. But it is not
mandatory.

For the XML in the example above, the YANG module could look something like:
::

  module mod_a{
    prefix a;
    namespace "urn:example:a";
    list x{
      leaf k1{
        type string;
      }
    }
  }

Creating JSON from a string
----------------------------
You can create an XML tree from JSON as well (note quotes are not escaped for clarity):
::

     cxobj *xt = NUL;L
     cxobj *xerr = NULL;

     if ((ret = json_parse_str(&xt, yt, "{"mod_a:x":{"k1":"a"}}", &xerr)) < 0)
        err;

yielding the same xt tree as in `Creating XML from a string`_.

There are some differences in comparison with the XML parse function:

* There is no variable argument format string
* Namespace prefixes in JSON use YANG module names, making the JSON format dependent on a correct YANG specification. This is similar to prefix handling in Api-path described in :ref:`clixon_paths`.
* Error return is `-1` and OK is `1`
* If the return value of the function is `0`, the JSON is not validated correctly witrh respect to YANG, and an error message is returned in `xerr`.

  
Creating XML programmatically
-----------------------------

You may also manually create a tree by `xml_new()`, `xml_addsub()` and
other functions, this is more efficient than parsing but more work to program.

.. note::
   expand on this.


Config data
-----------

To create a copy of configuration data, a user retrieve a copy from the datastore to get a cxobj handle.
Read-only operations may then be done on the in-memory tree.

The following example code gets a copy of the whole `running` datastore to cxobj `xt`:
::

     cxobj *xt = NULL;
     if (xmldb_get(h, "running", NULL, NULL, &xt) < 0)
        err;

.. note::
        In the case of config data, in-memory trees are read-only *caches* of
        the datastore and can normally not be written back to the datastore.
        Changes to the config datastire should be made via the backend netconf API, eg using
        `edit-config`.


Searching in XML
=================

Clixon search indexes are either *implicitly* created from the YANG
specification, or *explicitly* created using the API.

From YANG it is only `list` and `leaf-list` that are candidates for
optimized lookup, direct `leaf` and `container` lookup is fast either way.

*Binary* search is used by search indexes and works by ordering list
items alphabetically (or numerically), and then dividing the search interval in
two equal parts depending on if the requested item is larger than, or
less than, the middle of the interval.

Binary search complexity is *O(log N)*, whereas linear search is is *O(n)*. 
For example, a search in a vector of one million children will take up to
`20` lookups, whereas linear search takes up to `1.000.000` lookups.

Therefore, if you have a large number of children and you need to make
searches, it is important that you use indexes, either implicit, or explicit.

Implicit indexes
----------------

Implicit YANG-based search indexes are based on `list` and `leaf-lists`. For
any list with keys `k1,...kn`, a set of indexes are created and an optimized search
can be made using the keys in the order they are defined. 

For example, assume the following YANG (this YANG is reused in later examples):
::

  module mod_a{
    prefix a;
    namespace "urn:example:a";
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
      ...

Assume also an example XML tree as follows:
::

   <top xmlns="urn:example:a">
     <x>
       <k1>a</k1>
       <k2>a</k2>
       <y>cc</y>
       <y>dd</y>
       <z>foo</a>
     </x>
     <x>
       <k1>a</k1>
       <k2>b</k2>
       <y>cc</y>
       <y>dd</y>
       <z>bar</a>
     </x>
     <x>
       <k1>b</k1>
       ...
   </top>
      
Then there will be two implicit search indexes created for all XML nodes `x` so that
they can be accessed with *O(log N)*  with e.g.:

* XPath or Instance-id: `x[k1="a"][k2="b"]`.
* Api-path: `x=a,b`.

If other search variables are used, such as: `x[z="foo"]` the time complexity will be `O(n)` since there is no explicit index for `z`.  The same applies to using key variables in another order than they appear in the YANG specification, eg: `x[k2="b"][k1="a"]`.

A search index is also generated for leaf-lists, using `x` as the base node, the following searches are optimized:

* XPath or Instance-id: `y[.="bb"]`.
* Api-path: `y=bb`.
  
In the following cases, implicit indexes are *not* created:

* No YANG definition of the XML children exists. There are several use-cases. For example that YANG is not used or the tree is part of YANG `ANYXML`. 
* The list represents `state` data
* The list is `ordered-by user` instead of the default YANG `ordered-by system`.

In those cases where implicit YANG indexes cannot be used, explicit indexes can be created for fast access.

Explicit indexes [#f1]_
------------------------

You can register explicit indexes using the function `clixon_index_register()`.

.. note:: *This section is not completed* 


Direct children
---------------

The basic C API for searching direct children of a cxobj is the `xml_find_index()` API.

An example call is as follows:
::
   
    cxobj  **xvec = NULL;
    size_t   xlen = 0;
    cvec    *cvk = NULL; vector of index keys 
    ... Populate cvk with key/values eg k1=a k2:b
    if (clixon_find_index(xp, yp, name, cvk, &xvec, &xlen) < 0)
       err;
    /* Loop over found children*/
    for (i = 0; i < xlen; i++) {
	x = xvec[i];
        ...
    }

where

+--------+-------------------------------------------+
| `xp`   | is an XML parent                          |
+--------+-------------------------------------------+
| `yp`   | is the YANG specification of xp           |
+--------+-------------------------------------------+
| `name` | is the name of the wanted children        |
+--------+-------------------------------------------+
| `cvk`  | is a vector of index name and value pairs |
+--------+-------------------------------------------+
| `xvec` | is a result vector of XML nodes.          |
+--------+-------------------------------------------+

For example, using the previous XML tree and if `name=x` and  `cvk`
contains the single pair: `k1=a`, then `xvec` will contain both `x`
entries after calling the function:
::

     0: <x><k1>a</k1><k2>a</k2><y>cc</y><y>dd</y><z>foo</a></x>
     1: <x><k1>a</k1><k2>b</k2><y>cc</y><y>dd</y><z>bar</a></x>

and the search was done using `O(logN)`.
     
Paths
-----

If deeper searches are needed, i.e., not just to direct children,
Clixon :ref:`clixon_paths` can be used to make a search request. There
are three path variants, each with its own pros and cons:

* XPath is most expressive, but only supports `O(logN)` search for
  YANG `list` entries (not leaf-lists), and adds overhead in terms of
  memory and cycles.
* Api-path is least expressive since it can only express YANG `list`
  and `leaf-list` key search.
* Instance-identifier can express all optimized searches as well as
  non-key searches. This is the recommended option.

Assume the same YANG as in the previous example, a path to find `y` entries with a specific value could be:

* XPath or instance-id: `/a:x[a:k1="a"][a:k2="b"]/a:y[.="bb"]` 
* Api-path: `/mod_a:x=a,b/y=bb`

which results in the following result:
::

     0: <y>bb</y>
  
An example call using instance-id:s is as follows:
::

   cxobj **xvec = NULL;
   size_t  xlen;
   if (clixon_find_instance_id(xt, yt, &xvec, &xlen,
          "/a:x[a:k1=\"a\"][k2=\"b\"]/a:y[.=\"bb\"") < 0) 
      goto err;
   for (i=0; i<xlen; i++){
      x = xvec[i];
         ...
   }

The example shows the usage of implicit key indexes which makes this
work in *O(logN)*, with the same exception rules as for direct children state in `Implicit indexes`_.

An example call using api-path:s instead is as follows:
::

   cxobj **xvec = NULL;
   size_t  xlen;
   if (clixon_find_api_path(xt, yt, &xvec, &xlen, "/mod_a:x=a,b/y=bb") < 0) 
      goto err;
   for (i=0; i<xlen; i++){
      x = xvec[i];
         ...
   }

The corresponding API for XPath is `xpath_vec()`.


Multiple keys
-------------

Optimized `O(logN)` lookup works with multiple key YANG `lists` but not
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

where `CX_ELMNT` selects element children (no attributes or body text).

However, it is recommended to use the `Searching in XML`_ for more efficient
searching.
  

Footnotes
---------
.. [#f1] Is planned for Clixon 4.4
