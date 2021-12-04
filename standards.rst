.. _clixon_standards:

Standards
=========

YANG
----

YANG and XML are central to Clixon.  Yang modules are used as a
specification for encoding XML or JSON configuration and state
data. The YANG spec is also used to generate an interactive CLI,
NETCONF and RESTCONF clients, as well as the format of the XML
datastore.

The YANG standards that Clixon follows include:

* `YANG 1.0 RFC 6020 <https://www.rfc-editor.org/rfc/rfc6020.txt>`_
* `YANG 1.1 RFC 7950 <https://www.rfc-editor.org/rfc/rfc7950.txt>`_
* `YANG module library RFC 7895 <http://www.rfc-editor.org/rfc/rfc7895.txt>`_

Clixon deviates from the YANG standard as follows (reference to RFC7950 sections in parenthesis):

Not implemented:

* action (7.15)
* augment in a uses sub-clause (7.17) (module-level augment is implemented)
* instance-identifier type (9.13)
* status (7.21.2)
* YIN (13)
* Default values on leaf-lists (7.7.2)
* error-message is not implemented as sub-statement of "range", "length" and "pattern"

Further:

Clixon supports the following extended XPath functions (10):
  
   - current()
   - deref()
   - derived-from(),
   - derived-from-or-self()
   - bit-is-set()
  
* The following extended XPath functions are *not* supported (10):
  
   - re-match()
   - enum-value()

See also support of standard XPath functions `XML and XPath`_
     
* if-feature-expr() is restricted to single layer expressions with and/or (7.20.2):
   - ``x and y`` and ``x or y`` *is* supported
   - ``x or (not y and z)`` is *not* supported 

Regular expressions
^^^^^^^^^^^^^^^^^^^
Clixon supports two regular expression engines:

`Posix`
   The default method, The regexps:s are translated to posix before matching with the standard Linux regex engine. This translation is not complete but can be considered "good-enough" for most yang use-cases. For reference, all standard `Yang models <https://github.com/YangModels/yang>`_ have been tested.
`Libxml2`
   Libxml2  uses the XSD regex engine. This is a complete XSD engine but you need to compile and link with libxml2 which may add overhead.

To use libxml2 in clixon you need enable libxml2 in both cligen and clixon:
::
   
  ./configure --with-libxml2 # both cligen and clixon

You then need to set the following configure option:
::

  <CLICON_YANG_REGEXP>libxml2</CLICON_YANG_REGEXP>


NETCONF
-------
Clixon implements the following NETCONF RFC:s:

* `RFC 5277: NETCONF Event Notifications <http://www.rfc-editor.org/rfc/rfc5277.txt>`_
* `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_
* `RFC 6242: Using the NETCONF Configuration Protocol over Secure Shell (SSH) <http://www.rfc-editor.org/rfc/rfc6242.txt>`_
* `RFC 8071: NETCONF Call Home and RESTCONF Call Home <http://www.rfc-editor.org/rfc/rfc8071.txt>`_. RESTCONF call home not implemented
* `RFC 8341: Network Configuration Access Control Model <http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM). Notification not implemented.

The following RFC6241 capabilities/features are hardcoded in Clixon:

* :candidate (RFC6241 8.3)
* :validate (RFC6241 8.6)
* :xpath (RFC6241 8.9)
* :notification (RFC5277)

The following features are optional and can be enabled by setting CLICON_FEATURE:

* :startup (RFC6241 8.7)
* :writable-running (RFC6241 8.2) - just write to running, no commit semantics

Clixon does *not* support the following NETCONF features:

* :url capability
* copy-config source config
* edit-config testopts 
* edit-config erropts
* edit-config config-text
* edit-config operation

Further, in `get-config` filter expressions, the RFC6241 XPath
Capability is preferred over default subtrees. This has two reasons:

1. XPath has better performance since the underlying system uses xpath, and subtree filtering is done after the complete tree is retrieved.
2. Subtree filtering does not support namespaces yet.

Default values
^^^^^^^^^^^^^^

Clixon only stores explicit set default values in datastores, while unset values are populated in memory on retrieval. This means that get-config will report all default values, not only those explicitly set. 

`RFC 6243: With-defaults Capability for NETCONF <http://www.rfc-editor.org/rfc/rfc6243.txt>`_ is not implemented. Among the modes described in the RFC, Clixon implements "report-all" with-respect to ``get`` and ``get-config`` operations, but "explicit" with respect to how configurations are saved in datastores.

RESTCONF
--------

Clixon supports the two RESTCONF compile-time variants: *FCGI* and *Native*. Both implements `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_.

The following features of RFC8040 are supported:

* OPTIONS, HEAD, GET, POST, PUT, DELETE, PATCH
* Stream notifications (Sec 6)
* Query parameters: `insert`, `point`, `content`, `depth`, `start-time` and `stop-time`.
* Monitoring (Sec 9)

The following features are *not* implemented:

* ETag/Last-Modified
* Query parameters: `fields`, `filter`, `with-defaults`

RESTCONF event notification as described in RFC7950 section 6 is supported as follows:

* is *not* supported by *native* 
* is supported by *FCGI* 

`NMDA` is partly supported according to `RFC 8324 <https://tools.ietf.org/html/rfc8342>`_ and `RFC 8527 <https://tools.ietf.org/html/rfc8527>`_. With-defaults and with-origin are not implemented.

`RFC 8072: YANG Patch Media Type <https://www.rfc-editor.org/rfc/rfc8072.txt>`_ is not implemented.

In the native mode, clixon also supports:

* HTTP/1 as a transport implemented by libevhtp
* HTTP/2 (RFC7540) as a transport implemented by libnghttp2.
* Transport Layer Security (TLS) implemented by libopenssl,
* ALPN as defined in RFC 7301 for http/1, http/2 protocol selection

XML and XPath
-------------
Clixon has its own implementation of XML and XPath. See more in the detailed API reference.

The XML-related standards include:

* `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_. (DOCTYPE/ DTD not supported)
* `Namespaces in XML 1.0 <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_
* `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_
       
Clixon XML supports version and UTF-8 only.

The following XPath axes are supported:

* child,
* descendant,
* descendant-or-self,
* self
* parent

The following xpath axes are *not* supported: preceding, preceding_sibling, namespace, following_sibling, following, ancestor,ancestor_or_self, and attribute

The following XPath functions as defined in Section 2.3 / 4 of the XPath 1.0 standard are supported:

* contains()
* count()
* false()
* name()
* node()
* not()
* position()
* text()
* true()

The following standard XPath functions are *not* supported:

* boolean
* ceiling
* comment
* concat
* floor
* id
* lang
* last
* local-name
* namespace-uri
* normalize-space
* number
* processing-instructions
* round
* starts-with
* string
* substring
* substring-after
* substring-before
* sum
* translate

Pagination
----------

The pagination solution is based on the following drafts:

- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-00>`_
- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-nc-02>`_
- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-rc-02>`_

See :ref:`clixon_pagination` for more info.

  
Unicode
-------
Unicode is not supported in YANG and XML.

JSON
----

Clixon implements JSON according to  `ECMA JSON Data Interchange Syntax <http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf>`_ and  `RFC 7951 JSON Encoding of Data Modeled with YANG <https://www.rfc-editor.org/rfc/rfc8040.txt>`_.
